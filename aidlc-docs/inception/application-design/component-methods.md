# Component Methods: itops-console

**Stage**: INCEPTION - Application Design
**Created**: 2026-11-24
**Companion to**: `components.md` (component IDs C1 to C31)

Signatures are Python 3.12 with full type hints, as the code will be checked
with mypy (strict on `baseline/`). Each method has a one-line purpose. Field
lists are indicative; exact fields, validation limits, thresholds and
messages are set in Functional Design. `async` marks methods that await I/O.

Shared aliases used below:

```python
type JsonValue = str | int | float | bool | None | list[JsonValue] | dict[str, JsonValue]
type ChatMessage = dict[str, JsonValue]      # one OpenAI-format chat message
type ToolSchema = dict[str, JsonValue]       # one OpenAI-format tool definition
type Clock = Callable[[], datetime]          # returns timezone-aware UTC now
```

---

## C1 Core types (`baseline/models.py`)

```python
class Outcome(StrEnum):        # PASS, FAIL, NO_DATA, STALE, NOT_APPLICABLE, UNVERIFIED
class CheckName(StrEnum):      # ENCRYPTION, SCREEN_LOCK, PATCH_ENROLLMENT, OS_PATCHING,
                               # THIRD_PARTY_PATCHING, ENDPOINT_PROTECTION, BACKUP
class DeviceClass(StrEnum):    # PERSONAL_PRODUCTIVITY, SHARED, SPECIAL_PURPOSE
class OsFamily(StrEnum):       # WINDOWS, MACOS
class TdxStatus(StrEnum):      # ACTIVE, INACTIVE, ABSENT
class DeviceStatus(StrEnum):   # CERTIFIED, CERTIFIED_PENDING_VERIFICATION, NOT_CERTIFIED,
                               # HYGIENE_MET, HYGIENE_NOT_MET,
                               # CANNOT_SCORE_CLASS_UNKNOWN, CLEAN_UP

@dataclass(frozen=True)
class DeviceContext:           # device_id, os, os_version, device_class
@dataclass(frozen=True)
class Advisory:                # code, message
@dataclass(frozen=True)
class Detail:                  # label, value (always-shown facts, e.g. last backup per destination)
@dataclass(frozen=True)
class CheckResult:             # check, outcome, reason, source_reliable, advisories, details

# Evidence, one type per check; None in DeviceEvidence means "no record"
@dataclass(frozen=True)
class EncryptionEvidence: ...          # volumes with type, encrypted, protection on; reported_at
@dataclass(frozen=True)
class ScreenLockEvidence: ...          # enabled, timeout_secs, reported_at
@dataclass(frozen=True)
class PatchEnrollmentEvidence: ...     # enrolled, last_checkin
@dataclass(frozen=True)
class OsPatchEvidence: ...             # os_version, vendor_supported, missing patches with publish
                                       # dates, unpatched vulnerabilities (KEV flag, CVSS), reported_at
@dataclass(frozen=True)
class ThirdPartyPatchEvidence: ...     # missing patches with severity, publish date, KEV, CVSS; inspected_at
@dataclass(frozen=True)
class EndpointProtectionEvidence: ...  # installed, last_checkin
@dataclass(frozen=True)
class BackupEvidence: ...              # last_connected, last_completed per destination
@dataclass(frozen=True)
class DeviceEvidence:                  # device_id, hostname, os, os_version, k2_class, tdx_class,
                                       # tdx_status, one optional evidence field per check

@dataclass(frozen=True)
class ClassResolution:         # device_class (or None), source: "override" | "k2" | "tdx" | "none"
@dataclass(frozen=True)
class DeviceReport:            # device_id, hostname, device_class, class_source, status,
                               # stopped_reporting, results, scored_at

# Rule values, one type per check, plus the aggregate
@dataclass(frozen=True)
class EncryptionRule: ...
@dataclass(frozen=True)
class ScreenLockRule: ...
@dataclass(frozen=True)
class PatchEnrollmentRule: ...
@dataclass(frozen=True)
class OsPatchingRule: ...
@dataclass(frozen=True)
class ThirdPartyPatchingRule: ...
@dataclass(frozen=True)
class EndpointProtectionRule: ...
@dataclass(frozen=True)
class BackupRule: ...
@dataclass(frozen=True)
class UnreliableSource:        # check, os, min_release (optional), note
@dataclass(frozen=True)
class Policy:                  # version, certified classes, per-check rules and applicability,
                               # unreliable sources, per-device class overrides
```

Fleet-only types are defined with C5 in `baseline/fleet.py` (Units Generation
Q1 = A: the fleet summary unit can be dropped whole).

## C2 Policy parser (`baseline/policy.py`)

```python
class PolicyError(ValueError):
    reason: str                                    # shown when startup stops

class PolicyFile(BaseModel):                       # boundary model, extra="forbid", bounded values
    ...

def parse_policy(text: str) -> Policy:
    """Parse policy YAML text with yaml.safe_load and validate it; raise PolicyError with the reason."""

def dump_policy(policy: Policy) -> str:
    """Serialize a Policy back to YAML text (round-trip property, TST-PBT-3)."""
```

## C3 Check rules (`baseline/rules.py`)

```python
def is_fresh(reported_at: datetime, window: timedelta, now: datetime) -> bool:
    """True when the report is no older than the window (inclusive, FR-RULE-0c)."""

def evaluate_encryption(ev: EncryptionEvidence | None, rule: EncryptionRule,
                        ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score encryption (FR-RULE-1)."""

def evaluate_screen_lock(ev: ScreenLockEvidence | None, rule: ScreenLockRule,
                         ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score screen lock with per-OS logic and advisories (FR-RULE-2, FR-RULE-3)."""

def evaluate_patch_enrollment(ev: PatchEnrollmentEvidence | None, rule: PatchEnrollmentRule,
                              ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score patch enrollment; no record is a failure (FR-RULE-4)."""

def evaluate_os_patching(ev: OsPatchEvidence | None, rule: OsPatchingRule,
                         ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score OS patching, support status and zero days (FR-RULE-5)."""

def evaluate_third_party_patching(ev: ThirdPartyPatchEvidence | None, rule: ThirdPartyPatchingRule,
                                  ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score third-party patching by severity and age (FR-RULE-6)."""

def evaluate_endpoint_protection(ev: EndpointProtectionEvidence | None, rule: EndpointProtectionRule,
                                 ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score the endpoint protection agent's presence and check-in (FR-RULE-7)."""

def evaluate_backup(ev: BackupEvidence | None, rule: BackupRule,
                    ctx: DeviceContext, now: datetime) -> CheckResult:
    """Score backup connection, with the last-completed advisory and details (FR-RULE-8)."""

def apply_unreliable_source(result: CheckResult, ctx: DeviceContext,
                            sources: Sequence[UnreliableSource]) -> CheckResult:
    """Turn a fail from a listed unreliable source into unverified, with the label (FR-RULE-0d)."""
```

## C4 Device scoring and status (`baseline/certification.py`)

```python
def resolve_device_class(device_id: str, k2_class: DeviceClass | None,
                         tdx_class: DeviceClass | None,
                         overrides: Mapping[str, DeviceClass]) -> ClassResolution:
    """Resolve class in the order override, K2, TDX (FR-CLS-1, FR-CLS-2)."""

def is_stopped_reporting(results: Sequence[CheckResult]) -> bool:
    """True when every applicable result is stale or no_data (FR-CERT-4)."""

def device_status(device_class: DeviceClass, results: Sequence[CheckResult],
                  policy: Policy) -> DeviceStatus:
    """One status from the class and the results (FR-CERT-1 to FR-CERT-3)."""

def score_device(evidence: DeviceEvidence, policy: Policy, now: datetime) -> DeviceReport:
    """Scope, class, every applicable rule, status: the whole device in one pure call."""
```

## C5 Fleet computation (`baseline/fleet.py`)

```python
@dataclass(frozen=True)
class DeviceRef:               # device_id, hostname
@dataclass(frozen=True)
class RemediationCatalogEntry: # remediation_id, name, fixes (checks), remote
@dataclass(frozen=True)
class CheckGap:                # check, only_blocker_count, non_passing_count
@dataclass(frozen=True)
class RemediationImpact:       # remediation_id, name, would_certify, would_improve
@dataclass(frozen=True)
class FleetSummary:            # in_scope_count, check_gaps, remediation_impacts,
                               # no_remote_fix_checks, stopped_reporting, class_unknown, clean_up

def check_gaps(reports: Sequence[DeviceReport]) -> tuple[CheckGap, ...]:
    """Per check: devices one fix from certified, and total non-passing results (FR-FLT-2)."""

def remediation_impacts(reports: Sequence[DeviceReport],
                        catalog: Sequence[RemediationCatalogEntry]) -> tuple[RemediationImpact, ...]:
    """Per remediation: devices it would certify and improve (FR-FLT-3)."""

def summarize_fleet(reports: Sequence[DeviceReport],
                    catalog: Sequence[RemediationCatalogEntry],
                    clean_up: Sequence[DeviceRef]) -> FleetSummary:
    """The whole fleet summary, including the three separate lists (FR-FLT-4)."""
```

---

## C6 and C7 Read client and act client (`mcp/clients.py`)

```python
class EngineError(Exception): ...
class EngineUnavailable(EngineError):
    server: Literal["read", "act"]
class EngineAuthError(EngineError):                # expired or rejected token (FR-TOK-1)
    server: Literal["read", "act"]
class EngineRefusal(EngineError):
    text: str                                      # the server's words, verbatim

class DeviceSummary(BaseModel):    # device_id, hostname, serial, assigned_person, os, tdx_status
class DeviceCandidate(BaseModel):  # device_id, hostname, serial, assigned_person, matched_on
class ToolResult(BaseModel):       # tool, content, audit_ref
class RemediationPlan(BaseModel):  # plan_id, remediation_id, remediation_name, targets,
                                   # script identity, runner
class ExecutionResult(BaseModel):  # plan_id, status, audit_ref, runner, message

class ReadClient:
    def __init__(self, url: str, token: SecretStr, timeout_secs: float) -> None: ...
    async def connect(self) -> None:
        """Open the MCP session with the read token; raise EngineAuthError or EngineUnavailable."""
    async def close(self) -> None:
        """Close the session; safe to call twice."""
    async def list_devices(self) -> list[DeviceSummary]:
        """In-scope devices (TDX active) for the numbered list (FR-DEV-1)."""
    async def list_inactive_devices(self) -> list[DeviceSummary]:
        """Devices in tools but not active in TDX (FR-SCOPE-2)."""
    async def resolve_device(self, text: str) -> list[DeviceCandidate]:
        """Candidate devices for free text (FR-DEV-2)."""
    async def get_evidence(self, device_id: str) -> EngineEvidence:
        """Normalized evidence for one device (FR-DEV-3)."""
    async def get_fleet_evidence(self) -> list[EngineEvidence]:
        """Normalized evidence for every in-scope device (FR-FLT-1)."""
    async def call_tool(self, name: str, arguments: Mapping[str, JsonValue]) -> ToolResult:
        """Call one investigation tool; the caller has already checked the allowlist."""

type ElicitationHandler = Callable[[str, Mapping[str, JsonValue]], Awaitable[ElicitAnswer]]
# ElicitAnswer is the accept, decline or cancel shape the fastmcp 4 client expects;
# its exact type is confirmed early in Construction (requirements risk).

class ActClient:
    def __init__(self, url: str, token: SecretStr, timeout_secs: float,
                 elicitation_handler: ElicitationHandler) -> None: ...
    async def connect(self) -> None:
        """Open the MCP session with the act token and register the elicitation handler."""
    async def close(self) -> None: ...
    async def plan_remediation(self, remediation_id: str,
                               device_ids: Sequence[str]) -> RemediationPlan:
        """Ask the act server for a plan; changes nothing (FR-REM-2)."""
    async def execute_remediation(self, plan_id: str) -> ExecutionResult:
        """Ask the act server to execute; the server elicits approval during this call."""
```

## C8 Evidence adapter (`mcp/evidence.py`)

```python
class EngineEvidence(BaseModel):       # mirrors the engine's normalized evidence, extra="forbid"
    ...

class EvidenceShapeError(ValueError): ...

def to_device_evidence(engine: EngineEvidence) -> DeviceEvidence:
    """Convert engine evidence to core evidence; raise EvidenceShapeError if it cannot (Q2 = A)."""
```

`mcp/clients.py` imports `EngineEvidence` from this module; this module
imports nothing from `mcp/clients.py`, so there is no cycle. The scoring
service builds `DeviceRef` values for the clean-up list from `DeviceSummary`.

## C9 Approval store (`mcp/approvals.py`)

```python
class ApprovalAction(StrEnum):     # APPROVE, DECLINE, CANCEL (what a person can send)
class ApprovalOutcome(StrEnum):    # ACCEPTED, DECLINED, CANCELLED, TIMED_OUT, AUTO_DECLINED

class ApprovalRequest(BaseModel):  # the itops-mcp approval schema, extra="forbid": plan_id,
                                   # approval_code, remediation, script identity, targets, runner

@dataclass
class PendingApproval:             # approval_id, request, session_id, opened_at (future is private)

@dataclass(frozen=True)
class ApprovalRecord:              # approval_id, plan_id, session_id, outcome,
                                   # opened_at, closed_at, reason

class ApprovalNotOpen(Exception):  # unknown, already answered or timed out: a replay
    ...

class ApprovalLedger(Protocol):
    def accepted_for(self, plan_id: str) -> ApprovalRecord | None:
        """The accepted record for this plan in this run, if any."""
    def records(self) -> Sequence[ApprovalRecord]:
        """Every approval and outcome in this run, oldest first."""

class ApprovalExpectations(Protocol):          # the narrow view given to the act guard
    def expect(self, plan_id: str, session_id: str) -> AbstractContextManager[None]: ...
    @property
    def ledger(self) -> ApprovalLedger: ...

class ApprovalStore:
    def __init__(self, events: EventBus, timeout: timedelta, clock: Clock,
                 page_url: Callable[[str], str]) -> None: ...
    def expect(self, plan_id: str, session_id: str) -> AbstractContextManager[None]:
        """Mark a plan as being executed by a session for the length of one execute call."""
    async def handle_elicitation(self, message: str,
                                 schema: Mapping[str, JsonValue]) -> ElicitAnswer:
        """The act client's elicitation handler: validate, open, wait for a person or decline."""
    def get_open(self, approval_id: str) -> PendingApproval | None:
        """The open approval with this id, if any (for the approval page)."""
    def list_open(self) -> Sequence[PendingApproval]:
        """Every open approval (for the home page)."""
    def answer(self, approval_id: str, action: ApprovalAction,
               form: Mapping[str, str]) -> ApprovalRecord:
        """Called only by the approval route: resolve once, or raise ApprovalNotOpen."""
    def cancel_all(self, reason: str) -> None:
        """Close every open approval as cancelled (shutdown)."""
    @property
    def ledger(self) -> ApprovalLedger: ...
```

The `tech-env.md` endpoint example calls `pending.validate()` and
`pending.accept()` on the pending approval. In this design those become one
method, `ApprovalStore.answer()`, so there is exactly one answering entry
point to test and to guard with an import-boundary test.

## C10 Act guard (`mcp/act_guard.py`)

```python
class ExecutionStatus(StrEnum):    # EXECUTED, REFUSED, NOT_EXECUTED, VIOLATION

@dataclass(frozen=True)
class ExecutionOutcome:            # plan_id, status, audit_ref, server_text (verbatim),
                                   # demo_runner, approval (ApprovalRecord or None)

class ActGuard:
    def __init__(self, act: ActClient, approvals: ApprovalExpectations,
                 events: EventBus) -> None: ...
    async def plan(self, remediation_id: str, device_ids: Sequence[str],
                   session_id: str) -> RemediationPlan:
        """Plan through the act client and emit the plan event (changes nothing)."""
    async def execute(self, plan_id: str, session_id: str) -> ExecutionOutcome:
        """Expect, execute, then cross-check the result against the ledger (Q6 = A)."""
```

## C11 Scoring service (`scoring.py`)

```python
class CatalogSource(Protocol):
    async def catalog(self) -> Sequence[RemediationCatalogEntry]:
        """Remediation catalog entries and the checks each one fixes (source: open item 2)."""

class ScoringService:
    def __init__(self, read: ReadClient, policy: Policy, catalog: CatalogSource,
                 clock: Clock, events: EventBus) -> None: ...
    async def score_device(self, device_id: str, session_id: str) -> DeviceReport:
        """Fetch, adapt and score one device; emit the console-initiated scoring step."""
    async def score_fleet(self) -> FleetSummary:
        """Fetch, adapt and score every in-scope device, then summarize the fleet."""
```

---

## C12 LLM client (`llm.py`)

```python
class LlmUnavailable(Exception): ...

@dataclass(frozen=True)
class ToolCall:            # call_id, name, arguments_json
@dataclass(frozen=True)
class TokenUsage:          # prompt_tokens, completion_tokens
@dataclass(frozen=True)
class ModelTurn:           # text, tool_calls, usage, finish_reason

class LlmClient:
    def __init__(self, base_url: str, api_key: SecretStr, model: str,
                 timeout_secs: float) -> None: ...
    async def complete(self, messages: Sequence[ChatMessage],
                       tools: Sequence[ToolSchema] = (),
                       max_tokens: int | None = None) -> ModelTurn:
        """One chat completion through the gateway; raise LlmUnavailable on connection or auth failure."""
    async def close(self) -> None: ...
```

## C13 Tool registry (`agent/tools.py`)

```python
READ_TOOL_ALLOWLIST: Final[frozenset[str]]      # names fixed from the recorded engine contract

class ReadTools(Protocol):
    async def call_tool(self, name: str, arguments: Mapping[str, JsonValue]) -> ToolResult: ...

class Planner(Protocol):                         # satisfied by ActGuard.plan; no execute method
    async def plan(self, remediation_id: str, device_ids: Sequence[str],
                   session_id: str) -> RemediationPlan: ...

@dataclass(frozen=True)
class ToolSpec:            # name, description, args_model (pydantic, length limits), kind: "read" | "plan"

@dataclass(frozen=True)
class ToolOutcome:         # call_id, name, ok, content (text for the model), audit_ref, plan_id, error

class ToolRegistry:
    def __init__(self, read: ReadTools, planner: Planner, specs: Sequence[ToolSpec]) -> None:
        """Refuse at construction any spec not in the allowlist or not plan_remediation."""
    def names(self) -> frozenset[str]:
        """The tool names the model may call."""
    def openai_tools(self) -> list[ToolSchema]:
        """Tool definitions in OpenAI format."""
    async def call(self, call: ToolCall, session_id: str) -> ToolOutcome:
        """Validate arguments, dispatch, and return the result as data; unknown tools are refused."""
```

## C14 Agent loop (`agent/loop.py`)

```python
@dataclass(frozen=True)
class AgentLimits:         # max_steps, max_tokens (from settings)

class StopReason(StrEnum): # ANSWERED, STEP_LIMIT, TOKEN_LIMIT, LLM_UNAVAILABLE

@dataclass(frozen=True)
class AgentOutcome:        # text, stop_reason, steps, tokens_used, messages, plan_ids

class AgentLoop:
    def __init__(self, llm: LlmClient, tools: ToolRegistry, events: EventBus,
                 limits: AgentLimits) -> None: ...
    async def run(self, messages: Sequence[ChatMessage], session_id: str) -> AgentOutcome:
        """Run model turns and tool calls until answered or a limit is reached, emitting each step."""
```

## C15 Prompts (`agent/prompts.py`)

```python
def system_prompt() -> str:
    """The fixed system prompt: tool output is data; scores are facts; catalog fixes only."""

def facts_message(report: DeviceReport) -> ChatMessage:
    """The scoring results as fixed facts for the model (FR-DEV-3)."""

def tool_result_message(outcome: ToolOutcome) -> ChatMessage:
    """A tool result wrapped and labeled as data (LLM01)."""

def narrative_messages(summary: FleetSummary) -> list[ChatMessage]:
    """The prompt for the fleet narrative, from computed numbers only (FR-FLT-5)."""
```

## C16 Fleet narrative writer (`agent/narrative.py`)

```python
@dataclass(frozen=True)
class Narrative:           # text, model, generated_at, label ("model-written")

class NarrativeWriter:
    def __init__(self, llm: LlmClient, events: EventBus, clock: Clock) -> None: ...
    async def write(self, summary: FleetSummary) -> Narrative:
        """One tool-free model call; recorded as its own short transcript."""
```

---

## C17 Chat session service (`chat/session.py`)

```python
class ChatSession:
    def __init__(self, session_id: str, identity: DeviceIdentityResolver,
                 scoring: ScoringService, agent: AgentLoop, act: ActGuard,
                 events: EventBus) -> None: ...
    async def start(self) -> None:
        """Emit session_started, which opens the session's transcript."""
    async def list_devices(self) -> list[DeviceSummary]:
        """Emit the numbered device list and remember it for choose()."""
    async def choose(self, number: int) -> Identification:
        """Select from the last list or pick-list; score when a device is selected."""
    async def identify(self, text: str) -> Identification:
        """Resolve free text; score when exactly one device is selected."""
    async def ask(self, question: str) -> AgentOutcome:
        """Run the agent with the fixed facts and history; keep any plan IDs created."""
    async def execute(self, plan_id: str) -> ExecutionOutcome:
        """Technician command only: execute a plan created in this session through the act guard."""
    async def end(self) -> None:
        """Emit session_ended and close the transcript."""

class ChatSessionFactory:
    def new(self) -> ChatSession:
        """A session with a new random session ID and the shared components."""
```

## C18 Device identity resolver (`chat/identity.py`)

```python
class IdentificationKind(StrEnum):   # SELECTED, PICK_LIST, NO_MATCH

@dataclass(frozen=True)
class Identification:                # kind, device (when selected), choices (when a pick-list)

def decide(candidates: Sequence[DeviceCandidate]) -> Identification:
    """Pure: one candidate is selected, several give a pick-list, none gives no match."""

class DeviceIdentityResolver:
    def __init__(self, read: ReadClient) -> None: ...
    async def list_devices(self) -> list[DeviceSummary]: ...
    async def resolve(self, text: str) -> Identification: ...
```

## C19 Terminal command loop (`chat/terminal.py`)

```python
class CommandKind(StrEnum):    # DEVICES, PICK, DEVICE, EXECUTE, HELP, QUIT, QUESTION

@dataclass(frozen=True)
class Command:                 # kind, argument

def parse_command(line: str) -> Command:
    """Pure: turn one input line into a command or a question (syntax set in Functional Design)."""

class LineReader(Protocol):
    async def readline(self) -> str | None: ...   # None at end of input

class TerminalChat:
    def __init__(self, sessions: ChatSessionFactory, reader: LineReader,
                 events: EventBus) -> None: ...
    async def run(self) -> None:
        """Read lines until quit or end of input, one session at a time."""
    async def handle(self, command: Command) -> bool:
        """Dispatch one command to the session service; False ends the loop."""
```

## C20 Terminal printer (`chat/printer.py`)

```python
def sanitize(text: str) -> str:
    """Pure: remove terminal control characters and escape sequences (MS-8)."""

def format_event(event: SessionEvent) -> str:
    """Pure: the terminal text for one event; security events as alert lines."""

class TerminalPrinter:                     # an EventSink
    def __init__(self, stream: TextIO) -> None: ...
    def handle(self, event: SessionEvent) -> None: ...
```

---

## C21 Web app factory and routes (`app.py`)

```python
@dataclass(frozen=True)
class WebDeps:             # settings, launch, csrf, approvals, scoring, narrative,
                           # transcripts (reader), events, templates

def create_app(deps: WebDeps) -> FastAPI:
    """Build the app: docs off, security middleware and error handlers on, deps on app.state."""

# Route handlers (path and form values validated with pydantic and explicit limits)
async def health() -> PlainTextResponse: ...
async def start(request: Request, token: str) -> RedirectResponse: ...
async def home(request: Request) -> HTMLResponse: ...
async def approval_page(request: Request, approval_id: str) -> HTMLResponse: ...
async def answer_approval(request: Request, approval_id: str, action: ApprovalAction,
                          csrf_token: str) -> HTMLResponse: ...
async def fleet_page(request: Request) -> HTMLResponse: ...
async def fleet_narrative(request: Request, csrf_token: str) -> HTMLResponse: ...
async def transcripts_page(request: Request) -> HTMLResponse: ...
async def transcript_page(request: Request, session_id: str) -> HTMLResponse: ...
```

## C22 Request security (`security.py`)

```python
class LaunchSession:
    def __init__(self, events: EventBus, max_failed_exchanges: int) -> None:
        """Create a random launch token for this run."""
    def start_path(self) -> str:
        """'/start?token=...', printed once in the terminal and never logged."""
    def exchange(self, token: str) -> str:
        """Constant-time check; return a new session ID, or raise and count the failure."""
    def is_valid(self, session_id: str | None) -> bool: ...
    def end(self) -> None:
        """Invalidate the launch token and every session (console stop)."""

class CsrfTokens:
    def issue(self, session_id: str) -> str: ...
    def verify(self, session_id: str, token: str) -> bool: ...

def require_session(request: Request) -> str:
    """FastAPI dependency: the session ID, or refuse with a security event."""
def require_same_origin(request: Request) -> None:
    """Refuse a POST whose Origin is not this server's, with a security event."""
def require_csrf(request: Request, token: str) -> None:
    """Refuse a POST without the session's CSRF token, with a security event."""

class SecurityHeadersMiddleware: ...       # CSP, nosniff, frame DENY, referrer policy
class BodySizeLimitMiddleware: ...         # cap on request body size

def install_error_handlers(app: FastAPI, events: EventBus) -> None:
    """Generic error pages; log the detail, never show it."""
def verify_static_integrity(path: Path, expected_sha256: str) -> None:
    """Refuse to start if the vendored HTMX file does not match its recorded hash."""
```

## C23 to C25 Pages (`templates/`)

| Template | Context (values passed in) |
|---|---|
| `approval.html` | the `PendingApproval` (every target, remediation, script identity, approval code, runner), CSRF token, demo-runner and human-verification notices |
| `approval_answered.html` | the `ApprovalRecord` |
| `fleet.html` | the `FleetSummary`, CSRF token |
| `fleet_narrative.html` (partial) | the `Narrative` with its model-written label |
| `transcripts.html` | `SessionSummary` list |
| `transcript.html` | one session's `TranscriptRecord` list |
| `home.html` | open approvals, links |
| `error.html` | a generic message only |

---

## C26 Settings (`settings.py`)

```python
class Settings(BaseModel):                 # frozen, extra="forbid"
    llm_base_url: HttpUrl                  # https only
    llm_api_key: SecretStr
    llm_model: str
    llm_eval_model: str
    itops_read_url: AnyHttpUrl             # loopback only
    itops_act_url: AnyHttpUrl              # loopback only
    itops_read_token: SecretStr
    itops_act_token: SecretStr
    baseline_policy_path: Path
    agent_max_steps: int
    agent_max_tokens: int
    approval_timeout_secs: int
    web_port: int = 8080
    transcript_dir: Path
    log_dir: Path

class SettingsError(ValueError):
    problems: list[str]                    # one line per missing or invalid variable, no values

def load_settings(environ: Mapping[str, str], env_file: Path | None) -> Settings:
    """Read the environment and .env; raise SettingsError naming each problem."""

def secret_values(settings: Settings) -> list[str]:
    """Every secret's value, for the redactors only."""
```

## C27 Session events and dispatcher (`events.py`)

```python
class EventKind(StrEnum):
    # session:   SESSION_STARTED, SESSION_ENDED, NOTICE
    # device:    DEVICE_LIST, DEVICE_PICK_LIST, DEVICE_SELECTED, DEVICE_NO_MATCH
    # scoring:   SCORING_STARTED, SCORING_RESULT
    # agent:     MODEL_TURN, TOOL_CALL, TOOL_RESULT, AGENT_STOPPED, LLM_UNAVAILABLE
    # remediate: PLAN_CREATED, APPROVAL_OPENED, APPROVAL_ANSWERED, APPROVAL_AUTO_DECLINED,
    #            APPROVAL_TIMED_OUT, EXECUTION_RESULT, ENGINE_REFUSAL
    # fleet:     NARRATIVE_WRITTEN
    # problems:  ENGINE_ERROR, ERROR, SECURITY_ALERT

class Severity(StrEnum):   # INFO, WARNING, ERROR

@dataclass(frozen=True)
class SessionEvent:        # kind, at, session_id (None for events outside a session),
                           # severity, data (JSON values only)

class EventSink(Protocol):
    def handle(self, event: SessionEvent) -> None: ...

class EventBus:
    def __init__(self, sinks: Sequence[EventSink], clock: Clock,
                 secrets: Sequence[str]) -> None:
        """Sinks in fixed order: transcript, log, terminal."""
    def emit(self, kind: EventKind, session_id: str | None, **data: JsonValue) -> SessionEvent:
        """Redact, build, and hand the event to every sink."""
    def security(self, reason: str, session_id: str | None = None,
                 **data: JsonValue) -> SessionEvent:
        """Emit SECURITY_ALERT at WARNING (SEC-REQ-11)."""
```

Security alert reasons (the `reason` value): `bad_session`,
`launch_token_rejected`, `launch_token_locked`, `origin_refused`,
`csrf_refused`, `approval_not_open`, `schema_mismatch`,
`unexpected_elicitation`, `unapproved_execution`, `non_demo_runner`.

## C28 Transcript store (`transcript.py`)

```python
@dataclass(frozen=True)
class TranscriptRecord:    # seq, at, session_id, kind, severity, data
@dataclass(frozen=True)
class SessionSummary:      # session_id, kind ("chat" | "fleet_narrative"), started_at,
                           # device_id, record_count

def to_line(record: TranscriptRecord) -> str:
    """One JSON line."""
def from_line(line: str) -> TranscriptRecord:
    """Parse one JSON line (round-trip with to_line, TST-PBT-3)."""

class TranscriptWriter:                    # an EventSink
    def __init__(self, directory: Path) -> None: ...
    def handle(self, event: SessionEvent) -> None:
        """Append the event to its session's file (mode "a"); ignore events with no session."""
    def close(self) -> None: ...

class TranscriptReader:
    def __init__(self, directory: Path) -> None: ...
    def list_sessions(self) -> list[SessionSummary]: ...
    def read(self, session_id: str) -> list[TranscriptRecord]:
        """Check the ID's format and that the path stays inside the directory, then read."""
```

## C29 Structured logging and security alerts (`logs.py`)

```python
def configure_logging(log_dir: Path, secrets: Sequence[str],
                      retention_days: int = 90) -> logging.Logger:
    """JSON lines to the local log directory, redacted, kept at least retention_days."""

class JsonFormatter(logging.Formatter): ...    # timestamp, session or request ID, level, message
class RedactingFilter(logging.Filter): ...     # replaces secret values before formatting

class LogSink:                                 # an EventSink
    def __init__(self, logger: logging.Logger) -> None: ...
    def handle(self, event: SessionEvent) -> None:
        """Log every event; security alerts at WARNING."""
```

## C30 Composition root and entry point (`main.py`)

```python
@dataclass
class Console:             # settings, events, approvals, read, act, guard, scoring, llm,
                           # agent, narrative, launch, app, terminal

def build(settings: Settings, policy_text: str, *, clock: Clock,
          stdin: LineReader, stdout: TextIO, catalog: CatalogSource) -> Console:
    """Create every component once and wire them explicitly (Q8 = A); no I/O beyond construction."""

async def serve(console: Console) -> None:
    """Connect clients, print the launch link, run web server and terminal; clean shutdown."""

def run() -> None:
    """Console-script entry: load settings, read the policy file, check HTMX, build, serve."""
```

## C31 Scenario suite runner (`evals/run.py`)

```python
@dataclass(frozen=True)
class Scenario:            # scenario_id, device_id, question, expected_cause,
                           # expected_fix (a remediation ID or "no reviewed fix")
@dataclass(frozen=True)
class ScenarioResult:      # scenario_id, model, transcript session_id, final_text,
                           # cause_correct, fix_correct

def load_scenarios(directory: Path) -> list[Scenario]: ...
async def run_scenario(scenario: Scenario, console: Console) -> ScenarioResult: ...
def grade(result: ScenarioResult, scenario: Scenario) -> ScenarioResult:
    """Fill in cause_correct and fix_correct (method: open item 6)."""
def write_results(results: Sequence[ScenarioResult], model: str, run_date: date,
                  directory: Path) -> Path: ...
def main(argv: Sequence[str]) -> int: ...
```
