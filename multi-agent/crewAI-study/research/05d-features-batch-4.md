# crewAI Feature Deep Dive: Batch 4

**Features Covered:** Telemetry & Observability, Human-in-the-Loop & Feedback, Security & Guardrails
**Repository:** `/Users/sheldon/Documents/claw/reference/crewAI`
**crewAI Version:** `1.13.0rc1`
**Research Date:** 2026-03-27

---

## Table of Contents

1. [Telemetry & Observability](#1-telemetry--observability)
2. [Human-in-the-Loop & Feedback](#2-human-in-the-loop--feedback)
3. [Security & Guardrails](#3-security--guardrails)

---

## 1. Telemetry & Observability

### Overview

crewAI implements a dual-layer observability system:

1. **OpenTelemetry-based tracing** (`telemetry/`) -- spans exported to `telemetry.crewai.com:4319`
2. **Event-driven trace collection** (`events/listeners/tracing/`) -- batched events sent to the crewAI Plus backend API

Both systems are opt-in via environment variables or explicit configuration. Sensitive data is explicitly excluded from telemetry.

### 1.1 OpenTelemetry Layer

**Files:**
- `lib/crewai/src/crewai/telemetry/telemetry.py` (1036 lines)
- `lib/crewai/src/crewai/telemetry/constants.py`
- `lib/crewai/src/crewai/telemetry/utils.py`

#### Architecture

The `Telemetry` class is a **singleton** that wraps OpenTelemetry's `TracerProvider` with a `BatchSpanProcessor` and a custom `SafeOTLPSpanExporter` that never propagates export failures to the application.

```python
# telemetry/telemetry.py -- SafeOTLPSpanExporter (lines 65-85)
class SafeOTLPSpanExporter(OTLPSpanExporter):
    def export(self, spans: Any) -> SpanExportResult:
        try:
            return super().export(spans)
        except Exception as e:
            logger.error(e)
            return SpanExportResult.FAILURE  # NEVER raises
```

**Key design pattern:** Every telemetry operation is wrapped in `_safe_telemetry_operation()`, which silently swallows exceptions and returns `None`:

```python
# telemetry/telemetry.py -- lines 244-261
def _safe_telemetry_operation(
    self, operation: Callable[[], Span | None]
) -> Span | None:
    if not self._should_execute_telemetry():
        return None
    try:
        return operation()
    except Exception as e:
        logger.debug(f"Telemetry operation failed: {e}")
        return None
```

#### Disable Telemetry

Three independent env vars, any one disables telemetry:

```python
# telemetry/telemetry.py -- lines 146-152
@classmethod
def _is_telemetry_disabled(cls) -> bool:
    return (
        os.getenv("OTEL_SDK_DISABLED", "false").lower() == "true"
        or os.getenv("CREWAI_DISABLE_TELEMETRY", "false").lower() == "true"
        or os.getenv("CREWAI_DISABLE_TRACKING", "false").lower() == "true"
    )
```

#### Signal Handling

The `Telemetry` class registers `atexit` handlers and signal handlers (SIGTERM, SIGINT, SIGHUP, SIGTSTP, SIGCONT) to ensure spans are **force-flushed** before process exit:

```python
# telemetry/telemetry.py -- lines 170-227
def _register_shutdown_handlers(self) -> None:
    atexit.register(self._shutdown)
    # Registers signal handlers that emit events to crewai_event_bus
```

#### Telemetry Events Tracked

The `Telemetry` class records spans for:

| Span Name | Tracked Data |
|-----------|-------------|
| `Crew Created` | Version, Python version, crew config, process, memory settings, agent/task fingerprints |
| `Task Execution` | Task attributes, agent role, fingerprints |
| `Tool Usage` | Tool name, attempts, LLM model |
| `Tool Repeated Usage` | Same as above + attempt count |
| `Tool Usage Error` | Same + error context |
| `Crew Execution` | Full crew details (only when `share_crew=True`) |
| `Crew Individual Test Result` | Quality score, exec time, model name |
| `Flow Creation` | Flow name |
| `Flow Plotting` | Flow name, node names |
| `Flow Execution` | Flow name, node names |
| `Human Feedback` | Event type, routing info, outcome |

**Important:** `share_crew=True` on a Crew enables sending full agent goals/backstories and task descriptions to the telemetry backend. This is opt-in at the Crew level.

```python
# telemetry/telemetry.py -- lines 301-408
if crew.share_crew:
    self._add_attribute(span, "crew_agents", json.dumps([{
        "goal": agent.goal,
        "backstory": agent.backstory,
        # ... full details
    }]))
```

### 1.2 Event-Driven Trace Collection

**Files:**
- `lib/crewai/src/crewai/events/listeners/tracing/trace_listener.py` (907 lines)
- `lib/crewai/src/crewai/events/listeners/tracing/trace_batch_manager.py` (490 lines)
- `lib/crewai/src/crewai/events/listeners/tracing/first_time_trace_handler.py` (276 lines)
- `lib/crewai/src/crewai/events/listeners/tracing/types.py`
- `lib/crewai/src/crewai/events/listeners/tracing/utils.py` (558 lines)

#### Architecture

`TraceCollectionListener` is a **singleton** `BaseEventListener` that subscribes to the `crewai_event_bus` and batches events into a `TraceBatch`. It is separate from the OpenTelemetry layer -- it sends events to the **crewAI Plus backend API** (not OTLP).

**Batch lifecycle:**

1. Batch initialized on first event (`CrewKickoffStartedEvent`, `FlowStartedEvent`, or any action event)
2. Events buffered via `add_event()` with thread-safe counting
3. On `CrewKickoffCompletedEvent` / `CrewKickoffFailedEvent` / `FlowFinishedEvent`:
   - Wait for all in-flight event handlers to complete (2s timeout)
   - Sort events by `emission_sequence`
   - Send to backend via `PlusAPI`
   - Finalize batch

**Key detail -- race condition handling:** The batch can be initialized from multiple event sources concurrently (`FlowStartedEvent` vs action events). The `_initialize_flow_batch` and `_initialize_crew_batch` methods use an `is_batch_initialized()` check to prevent duplicate initialization, and correctly set `batch_owner_type` to distinguish flows from crews:

```python
# trace_listener.py -- lines 271-279
@event_bus.on(CrewKickoffStartedEvent)
def on_crew_started(source: Any, event: CrewKickoffStartedEvent) -> None:
    if self.batch_manager.batch_owner_type != "flow":
        # Always call to claim ownership
        self._initialize_crew_batch(source, event)
```

#### Trace Event Types

The `TraceEvent` dataclass captures:

```python
# trace_listener.py/types.py -- TraceEvent
@dataclass
class TraceEvent:
    event_id: str           # UUID, auto-generated
    timestamp: str           # ISO format UTC
    type: str               # e.g. "task_started", "llm_call_completed"
    event_data: dict        # Typed per event type
    emission_sequence: int | None
    parent_event_id: str | None
    previous_event_id: str | None
    triggered_by_event_id: str | None
```

`TraceCollectionListener` handles these event categories:
- **Context events:** Crew/Flow start/complete/fail, Task start/complete/fail, Agent start/complete/error
- **Action events:** LLM calls, tool usage, memory operations, reasoning, knowledge retrieval
- **A2A events:** Full Agent-to-Agent protocol (delegation, conversation, push notifications, streaming)
- **Signal events:** SIGTERM/SIGINT flush to prevent data loss

#### First-Time User Flow

`FirstTimeTraceHandler` manages the first-run tracing experience:

1. Detects first execution via `is_first_execution()` (checks `~/.crewai_user.json`)
2. Auto-collects traces in ephemeral mode
3. Prompts user with a **20-second timeout** `click.confirm()` dialog after first execution
4. If user consents: initializes backend batch, sends events, displays URL (auto-opens browser)
5. If user declines: saves `trace_consent: False` to user data
6. All future runs respect the saved preference

```python
# trace_listener.py -- utils.py, lines 461-481
def should_auto_collect_first_time_traces() -> bool:
    if _is_test_environment(): return False
    if has_user_declined_tracing(): return False
    if is_tracing_enabled_in_context(): return False
    return is_first_execution()
```

#### Privacy-Preserving Machine ID

The user ID for trace batches is a **SHA-256 hash** of `[username]|[machine_id]`, never the raw username or machine ID:

```python
# trace_listener.py -- utils.py, lines 375-383
def _generate_user_id() -> str:
    seed = f"{getpass.getuser()}|{_get_machine_id()}"
    return hashlib.sha256(seed.encode()).hexdigest()
```

The machine ID itself is assembled from platform-specific sources (MAC address on Darwin, DMI IDs on Linux, WMIC on Windows) and hashed.

### 1.3 Console Formatter

`lib/crewai/src/crewai/events/utils/console_formatter.py` provides rich console output during execution, including:
- Live updating of in-progress operations
- Pausing during human input
- Memory save/retrieve notifications
- Memory query failure formatting

This is integrated via `event_listener.formatter` (a `ConsoleFormatter` instance) throughout the tracing and flow systems.

---

## 2. Human-in-the-Loop & Feedback

### Overview

crewAI's HITL system is built primarily for **Flows** (not Crews), providing a `@human_feedback` decorator that pauses execution, collects human review of method output, and optionally uses an LLM to collapse free-form feedback into routing outcomes.

### 2.1 Core Decorator: `@human_feedback`

**File:** `lib/crewai/src/crewai/flow/human_feedback.py` (675 lines)

#### Usage Patterns

**Basic (no routing):**
```python
@start()
@human_feedback(message="Review this content:")
def generate_content(self):
    return {"title": "Article", "body": "Content..."}
```

**With routing:**
```python
@start()
@human_feedback(
    message="Review and approve or reject:",
    emit=["approved", "rejected", "needs_revision"],
    llm="gpt-4o-mini",
    default_outcome="needs_revision",
)
def review_document(self):
    return document_content

@listen("approved")
def publish(self):
    print(f"Publishing: {self.last_human_feedback.output}")
```

**Async with custom provider:**
```python
@human_feedback(
    message="Review this:",
    emit=["approved", "rejected"],
    llm="gpt-4o-mini",
    provider=SlackProvider(channel="#reviews"),
)
```

#### Decorator Implementation

The `human_feedback()` decorator returns a wrapper that:

1. Calls the original method to get `method_output`
2. Optionally runs `_pre_review_with_lessons()` if `learn=True` -- applies past feedback lessons via memory recall
3. Requests feedback via provider (sync or async)
4. Runs `_process_feedback()` to determine the `outcome`
5. If `learn=True`, calls `_distill_and_store_lessons()` to extract reusable rules from the feedback and store in memory
6. Returns the `outcome` string (for routing) or `HumanFeedbackResult`

**Clever detail -- preserving method output for routing:** When `emit` is set, the decorator stashes the actual method output in `self._human_feedback_method_outputs[func.__name__]` so the flow's final result contains the method output, not the collapsed outcome string.

#### Feedback Outcome Collapsing

When `emit` is specified, feedback is collapsed to an outcome via `_collapse_to_outcome()`:

```python
# flow/flow.py -- lines 3051-3096
class FeedbackOutcome(BaseModel):
    outcome: Literal[outcomes_tuple] = Field(
        description=f"Must be one of: {', '.join(outcomes)}"
    )
```

Uses **dynamic Pydantic model generation** with `Literal` types to constrain LLM output to valid outcomes. Falls back to string comparison if structured output fails.

#### HITL Learning System

The `learn=True` option enables a feedback loop:

1. **`_pre_review_with_lessons()`** -- Before showing output to human, queries memory for past HITL lessons using `mem.recall(query, source="hitl")`, then uses an LLM to incorporate those lessons into the output
2. **`_distill_and_store_lessons()`** -- After feedback, uses an LLM to extract **generalizable rules** from the feedback and stores them in memory for future use

```python
# human_feedback.py -- lines 373-407
def _pre_review_with_lessons(flow_instance, method_output):
    matches = mem.recall(query, source=learn_source)
    # LLM call to apply lessons
```

### 2.2 Async Feedback System

**Files:**
- `lib/crewai/src/crewai/flow/async_feedback/types.py` (291 lines)
- `lib/crewai/src/crewai/flow/async_feedback/providers.py` (196 lines)
- `lib/crewai/src/crewai/flow/flow_config.py` (73 lines)

#### PendingFeedbackContext

When a flow pauses for async feedback, `PendingFeedbackContext` captures everything needed to resume:

```python
# async_feedback/types.py -- lines 18-64
@dataclass
class PendingFeedbackContext:
    flow_id: str
    flow_class: str         # Fully qualified class name
    method_name: str
    method_output: Any
    message: str
    emit: list[str] | None
    default_outcome: str | None
    metadata: dict[str, Any]
    llm: dict[str, Any] | str | None  # Serialized LLM config
    requested_at: datetime
```

Includes `_make_json_safe()` for serializing arbitrary objects (Pydantic models, dataclasses) to JSON.

#### HumanFeedbackPending Exception

Raised by async providers to signal the flow should **pause without propagating the exception to the caller**:

```python
# async_feedback/types.py -- lines 141-211
class HumanFeedbackPending(Exception):
    def __init__(self, context, callback_info=None, message=None):
        self.context = context
        self.callback_info = callback_info or {}
        # Default message: f"Human feedback pending for flow '{context.flow_id}'..."
```

**Key design:** When raised inside the decorator, the exception is **caught by the framework**, state is automatically persisted, and the `HumanFeedbackPending` object is returned as the kickoff result -- NOT re-raised. The caller checks `isinstance(result, HumanFeedbackPending)`.

```python
# Flow.kickoff() returns HumanFeedbackPending as a VALUE, not raising it
result = flow.kickoff()
if isinstance(result, HumanFeedbackPending):
    print(f"Waiting for feedback: {result.context.flow_id}")
```

#### Provider Protocol

```python
# async_feedback/types.py -- lines 214-291
@runtime_checkable
class HumanFeedbackProvider(Protocol):
    def request_feedback(
        self, context: PendingFeedbackContext, flow: Flow[Any]
    ) -> str:
        # Sync: return feedback string
        # Async: raise HumanFeedbackPending(context, callback_info)
        ...
```

#### Default ConsoleProvider

The `ConsoleProvider` is the default synchronous provider (lines 19-195 in providers.py):

- Uses `input()` for blocking console input
- Displays the method output with a formatted panel
- Emits `HumanFeedbackRequestedEvent` and `HumanFeedbackReceivedEvent` to the event bus
- Pauses live console updates during input (`formatter.pause_live_updates()`)

#### Global HITL Provider Config

`flow_config` singleton allows setting a global provider at deployment startup:

```python
# flow_config.py -- lines 17-72
flow_config.hitl_provider = SlackProvider(channel="#reviews")
```

The decorator also accepts a per-decorator `provider` parameter, which takes precedence.

### 2.3 Human Input in Tasks

**File:** `lib/crewai/src/crewai/core/providers/human_input.py`

For Crews (not Flows), there is a separate `human_input` mechanism on `Task`:

```python
# From task.py grep
"human_input?": task.human_input,  # Telemetry tracking
```

The actual blocking human input for Crew tasks is handled through the agent execution loop via `Agent` prompting (not a formal decorator).

---

## 3. Security & Guardrails

### Overview

Security in crewAI has three distinct layers:
1. **Guardrails** -- LLM-based output validation on Tasks
2. **Fingerprinting** -- Unique identifiers for agents/crews for auditing
3. **SecurityConfig** -- Container for security-related component metadata

### 3.1 LLM Guardrails

**Files:**
- `lib/crewai/src/crewai/tasks/llm_guardrail.py` (103 lines)
- `lib/crewai/src/crewai/utilities/guardrail.py` (147 lines)
- `lib/crewai/src/crewai/utilities/guardrail_types.py` (19 lines)
- `lib/crewai/src/crewai/tasks/hallucination_guardrail.py` (104 lines)
- `lib/crewai/src/crewai/events/types/llm_guardrail_events.py` (74 lines)

#### LLMGuardrail -- Task Output Validation

`LLMGuardrail` is an LLM-based validator that creates a **Guardrail Agent** to check if a task output complies with a natural language description:

```python
# tasks/llm_guardrail.py -- lines 32-102
class LLMGuardrail:
    def __init__(self, description: str, llm: BaseLLM):
        self.description = description
        self.llm = llm

    def _validate_output(self, task_output: TaskOutput) -> LiteAgentOutput:
        agent = Agent(
            role="Guardrail Agent",
            goal="Validate the output of the task",
            backstory="You are a expert at validating the output of a task...",
            llm=self.llm,
        )
        query = f"""
        Ensure the following task result complies with the given guardrail.
        Task result: {task_output.raw}
        Guardrail: {self.description}
        Your task: Confirm if the Task result complies, provide feedback if not.
        """
        return agent.kickoff(query, response_format=LLMGuardrailResult)

    def __call__(self, task_output: TaskOutput) -> tuple[bool, Any]:
        result = self._validate_output(task_output)
        if result.pydantic.valid:
            return True, task_output.raw
        return False, result.pydantic.feedback
```

Returns `(success: bool, data: Any)` where data is the raw output on success or feedback string on failure.

#### Guardrail Callable Interface

```python
# utilities/guardrail_types.py -- lines 12-14
GuardrailCallable: TypeAlias = Callable[
    [TaskOutput | LiteAgentOutput], tuple[bool, Any]
]
```

#### process_guardrail -- Event Emission

```python
# utilities/guardrail.py -- lines 82-146
def process_guardrail(output, guardrail, retry_count, event_source=None, from_agent=None, from_task=None) -> GuardrailResult:
    crewai_event_bus.emit(event_source, LLMGuardrailStartedEvent(...))
    result = guardrail(output)
    guardrail_result = GuardrailResult.from_tuple(result)
    crewai_event_bus.emit(event_source, LLMGuardrailCompletedEvent(...))
    return guardrail_result
```

Emits `LLMGuardrailStartedEvent` and `LLMGuardrailCompletedEvent` for observability. Uses `GuardrailResult` Pydantic model with a `field_validator` that enforces mutual exclusivity of `result` and `error` based on `success`.

#### GuardrailResult Model

```python
# utilities/guardrail.py -- lines 19-79
class GuardrailResult(BaseModel):
    success: bool
    result: Any | None
    error: str | None

    @field_validator("result", "error")
    @classmethod
    def validate_result_error_exclusivity(cls, v, info):
        # Raises ValueError if both result and error are set when success=True
        # or both set when success=False
        ...
```

#### HallucinationGuardrail -- No-Op Stub

```python
# tasks/hallucination_guardrail.py -- lines 20-103
class HallucinationGuardrail:
    def __init__(self, llm, context=None, threshold=None, tool_response=""):
        self._logger.log("warning",
            "Hallucination detection is a no-op in open source, "
            "use it for free at https://app.crewai.com\n"
        )

    def __call__(self, task_output: TaskOutput) -> tuple[bool, Any]:
        if callable(_validate_output_hook):
            return _validate_output_hook(self, task_output)
        self._logger.log("warning",
            "Premium hallucination detection skipped\n"
        )
        return True, task_output.raw
```

**Critical finding:** `HallucinationGuardrail` is a **no-op in open source**. It always returns `True` and logs a warning directing users to the crewAI cloud. This is a premium feature that requires the managed cloud product.

### 3.2 Fingerprinting

**Files:**
- `lib/crewai/src/crewai/security/fingerprint.py` (164 lines)
- `lib/crewai/src/crewai/security/security_config.py` (88 lines)
- `lib/crewai/src/crewai/security/constants.py`

#### Fingerprint Class

Provides dual identifiers for components:

```python
# security/fingerprint.py -- lines 45-77
class Fingerprint(BaseModel):
    _uuid_str: str = PrivateAttr(default_factory=lambda: str(uuid4()))
    _created_at: datetime = PrivateAttr(default_factory=datetime.now)
    metadata: Annotated[dict[str, Any], BeforeValidator(_validate_metadata)]

    @property
    def uuid_str(self) -> str: return self._uuid_str
    @property
    def created_at(self) -> datetime: return self._created_at
    @property
    def uuid(self) -> UUID: return UUID(self.uuid_str)
```

**Two UUID generation modes:**

1. **Random UUID** (default): `uuid4()` for unique runtime tracking
2. **Deterministic UUID** (seed-based): `uuid5(NAMESPACE, seed)` for reproducible identifiers

```python
# security/fingerprint.py -- lines 79-112
@classmethod
def _generate_uuid(cls, seed: str) -> str:
    if not seed.strip():
        raise ValueError("Seed cannot be empty or whitespace")
    return str(uuid5(CREW_AI_NAMESPACE, seed))

@classmethod
def generate(cls, seed=None, metadata=None):
    fingerprint = cls(metadata=metadata or {})
    if seed:
        fingerprint.__dict__["_uuid_str"] = cls._generate_uuid(seed)
    return fingerprint
```

#### Metadata Validation

`_validate_metadata()` (lines 17-42) enforces:
- Must be a dict with string keys
- Max **one level** of nesting (no deeply nested dicts)
- Max **10KB** total size (prevents DoS)

```python
# security/fingerprint.py -- lines 38-40
if len(str(v)) > 10_000:
    raise ValueError("Metadata size exceeds maximum allowed (10KB)")
```

#### SecurityConfig

```python
# security/security_config.py -- lines 20-87
class SecurityConfig(BaseModel):
    model_config = ConfigDict(arbitrary_types_allowed=True)
    fingerprint: Fingerprint = Field(default_factory=Fingerprint)

    @field_validator("fingerprint", mode="before")
    @classmethod
    def validate_fingerprint(cls, v):
        if v is None: return Fingerprint()
        if isinstance(v, str): return Fingerprint.generate(seed=v)
        if isinstance(v, dict): return Fingerprint.from_dict(v)
        if isinstance(v, Fingerprint): return v
        raise ValueError(f"Invalid fingerprint type: {type(v)}")
```

Accepts fingerprint as `None`, `str` (seed), `dict`, or `Fingerprint` instance.

#### Namespace for Deterministic UUIDs

```python
# security/constants.py -- lines 13-16
CREW_AI_NAMESPACE: Annotated[
    UUID,
    "Create a deterministic UUID using v5 (SHA-1). Custom namespace for CrewAI to enhance security.",
] = UUID("f47ac10b-58cc-4372-a567-0e02b2c3d479")
```

This is a hardcoded UUID namespace (similar to DNS namespace for domain names) that ensures deterministic fingerprints generated by different installations are still globally unique.

### 3.3 Guardrail Events

```python
# events/types/llm_guardrail_events.py -- lines 8-73
class LLMGuardrailStartedEvent(LLMGuardrailBaseEvent):
    type: str = "llm_guardrail_started"
    guardrail: str | Callable[..., Any]
    retry_count: int

class LLMGuardrailCompletedEvent(LLMGuardrailBaseEvent):
    type: str = "llm_guardrail_completed"
    success: bool
    result: Any
    error: str | None
    retry_count: int

class LLMGuardrailFailedEvent(LLMGuardrailBaseEvent):
    type: str = "llm_guardrail_failed"
    error: str
    retry_count: int
```

Note: `LLMGuardrailFailedEvent` is defined but not currently emitted by `process_guardrail()` -- only `Started` and `Completed` are emitted.

---

## Key Findings Summary

### Telemetry & Observability

| Aspect | Finding |
|--------|---------|
| **OTEL integration** | Full OpenTelemetry SDK with BatchSpanProcessor, safe exporter that never propagates errors |
| **Disable flags** | Three independent env vars: `OTEL_SDK_DISABLED`, `CREWAI_DISABLE_TELEMETRY`, `CREWAI_DISABLE_TRACKING` |
| **Signal handling** | Registers atexit + 5 signal handlers (SIGTERM/INT/HUP/TSTP/CONT) for graceful flush |
| **Trace collection** | Separate event-driven system sending to crewAI Plus backend API (not OTLP) |
| **First-run UX** | Auto-collects ephemeral traces on first run, prompts user with 20s timeout, auto-opens browser on consent |
| **Privacy** | Machine ID and user ID are SHA-256 hashed before transmission |
| **share_crew** | Only sends full crew details (goals, backstories) when explicitly enabled at Crew level |

### Human-in-the-Loop

| Aspect | Finding |
|--------|---------|
| **Scope** | Primarily for Flows (not Crews); Crews have `human_input` flag but no formal decorator |
| **Blocking model** | Sync via `ConsoleProvider` (default), async via custom `HumanFeedbackProvider` raising `HumanFeedbackPending` |
| **Routing** | Uses dynamic `Literal` Pydantic model + function calling to constrain LLM output to valid outcomes |
| **Persistence** | `PendingFeedbackContext` serializes everything needed to resume; `HumanFeedbackPending` is caught and returned as value |
| **Learning** | `learn=True` enables feedback loop: memory recall for pre-review, LLM distillation for post-feedback storage |
| **Live updates** | Console formatter pauses during human input to prevent interleaved output |

### Security & Guardrails

| Aspect | Finding |
|--------|---------|
| **LLMGuardrail** | Fully functional: spins up a Guardrail Agent to validate task output against natural language criteria |
| **HallucinationGuardrail** | **No-op stub in open source** -- always returns True, logs warning pointing to crewAI cloud |
| **GuardrailCallable** | Simple `Callable[[TaskOutput | LiteAgentOutput], tuple[bool, Any]]` interface |
| **GuardrailResult** | Pydantic model with mutual exclusivity validation between `result` and `error` |
| **Fingerprinting** | Dual UUID modes: random (runtime tracking) and deterministic v5 (seeded, namespace-based) |
| **Metadata limits** | 10KB max, 1 level nesting -- explicit DoS protection |
| **SecurityConfig** | Currently only holds `fingerprint`; TODOs indicate planned auth, scoping, delegation features |

---

## Code Reference Index

| File | Lines | Purpose |
|------|-------|---------|
| `telemetry/telemetry.py` | 1036 | Singleton Telemetry with OTEL + signal handling |
| `telemetry/constants.py` | 11 | OTEL endpoint URL and service name |
| `telemetry/utils.py` | 116 | Span attribute helpers |
| `events/listeners/tracing/trace_listener.py` | 907 | TraceCollectionListener, event subscriptions |
| `events/listeners/tracing/trace_batch_manager.py` | 490 | Batch management, backend API calls |
| `events/listeners/tracing/first_time_trace_handler.py` | 276 | First-run trace UX flow |
| `events/listeners/tracing/utils.py` | 558 | Privacy-preserving IDs, consent management |
| `events/types/llm_guardrail_events.py` | 74 | Guardrail event types |
| `flow/human_feedback.py` | 675 | @human_feedback decorator |
| `flow/async_feedback/types.py` | 291 | PendingFeedbackContext, HumanFeedbackPending |
| `flow/async_feedback/providers.py` | 196 | ConsoleProvider |
| `flow/flow_config.py` | 73 | Global flow config singleton |
| `flow/flow.py` | -- | `_request_human_feedback`, `_collapse_to_outcome` |
| `tasks/llm_guardrail.py` | 103 | LLMGuardrail implementation |
| `utilities/guardrail.py` | 147 | `process_guardrail`, `GuardrailResult` |
| `utilities/guardrail_types.py` | 19 | Type aliases |
| `tasks/hallucination_guardrail.py` | 104 | HallucinationGuardrail (no-op) |
| `security/fingerprint.py` | 164 | Fingerprint class |
| `security/security_config.py` | 88 | SecurityConfig model |
| `security/constants.py` | 17 | CREW_AI_NAMESPACE UUID |
