# Components: itops-console

**Stage**: INCEPTION - Application Design
**Created**: 2026-11-24
**Plan**: `aidlc-docs/inception/plans/application-design-plan.md` (Q1 to Q9 = A)
**Inputs**: `requirements.md`, `stories.md`, `docs/tech-env.md`

One async Python process holds every component (`tech-env.md`). The package
keeps the `tech-env.md` project structure with no renaming (Q1 = A); modules
marked **new** are added by this design. This document names each component,
its purpose, its responsibilities and its interface. Business rules (exact
thresholds, state transitions, message wording) are decided in Functional
Design.

---

## Package map

| ID | Component | Module | Group | New |
|---|---|---|---|---|
| C1 | Core types | `baseline/models.py` | Baseline core | |
| C2 | Policy parser | `baseline/policy.py` | Baseline core | |
| C3 | Check rules | `baseline/rules.py` | Baseline core | |
| C4 | Device scoring and status | `baseline/certification.py` | Baseline core | new |
| C5 | Fleet computation | `baseline/fleet.py` | Baseline core | |
| C6 | Read client | `mcp/clients.py` | Engine integration | |
| C7 | Act client | `mcp/clients.py` | Engine integration | |
| C8 | Evidence adapter | `mcp/evidence.py` | Engine integration | new |
| C9 | Approval store (harness layer) | `mcp/approvals.py` | Engine integration | |
| C10 | Act guard (server-layer cross-check) | `mcp/act_guard.py` | Engine integration | new |
| C11 | Scoring service | `scoring.py` | Engine integration and scoring | new |
| C12 | LLM client | `llm.py` | Agent | |
| C13 | Tool registry | `agent/tools.py` | Agent | new |
| C14 | Agent loop | `agent/loop.py` | Agent | |
| C15 | Prompts | `agent/prompts.py` | Agent | |
| C16 | Fleet narrative writer | `agent/narrative.py` | Agent | new |
| C17 | Chat session service | `chat/session.py` | Terminal chat | new |
| C18 | Device identity resolver | `chat/identity.py` | Terminal chat | new |
| C19 | Terminal command loop | `chat/terminal.py` | Terminal chat | new |
| C20 | Terminal printer | `chat/printer.py` | Terminal chat | new |
| C21 | Web app factory and routes | `app.py` | Web | |
| C22 | Request security | `security.py` | Web | new |
| C23 | Approval page | `templates/approval.html` and routes in `app.py` | Web | |
| C24 | Fleet page | `templates/fleet.html` and routes in `app.py` | Web | |
| C25 | Transcript viewer | `templates/transcripts.html`, `templates/transcript.html` and routes in `app.py` | Web | |
| C26 | Settings | `settings.py` | Cross-cutting | |
| C27 | Session events and dispatcher | `events.py` | Cross-cutting | new |
| C28 | Transcript store | `transcript.py` | Cross-cutting | |
| C29 | Structured logging and security alerts | `logs.py` | Cross-cutting | new |
| C30 | Composition root and entry point | `main.py` | Cross-cutting | new |
| C31 | Scenario suite runner | `evals/run.py` | Outside the package | |

`security.py` is imported by the `tech-env.md` endpoint example and named in
SEC-REQ-14, but it is not in the `tech-env.md` structure list, so it is marked
new. `static/htmx.min.js` (vendored, hash recorded) belongs to the web group.

---

## Baseline core (`baseline/`)

**Rule for the whole group**: imports only the standard library, `pydantic`,
`yaml` and other `baseline` modules. It never opens a file, reads the clock,
calls the network or calls a model (FR-RULE-0a). The current time is always a
parameter. Types inside the group are frozen dataclasses; pydantic appears
only in the policy parser's boundary model (Q3 = A).

### C1 Core types (`baseline/models.py`)
- **Purpose**: the shared vocabulary of the rule engine, certification and
  fleet computation.
- **Responsibilities**:
  - Enumerations: the six outcomes (`pass`, `fail`, `no_data`, `stale`,
    `not_applicable`, `unverified`), the seven check names, device classes,
    OS families, device statuses.
  - Evidence types, one per check, plus a `DeviceEvidence` aggregate holding
    device identity, OS, class sources (K2, TDX), TDX status and every check's
    evidence.
  - Result types: `CheckResult` (outcome, reason, source reliability,
    advisories, always-shown details), `DeviceReport`.
  - Rule value types, one per check, and the `Policy` aggregate.
  - Fleet-only types are defined in `baseline/fleet.py` (C5), so the fleet
    summary unit can be dropped whole (Units Generation Q1 = A).
- **Interface**: types only.

### C2 Policy parser (`baseline/policy.py`)
- **Purpose**: turn the baseline policy text into validated rule values
  (FR-POL-1 to FR-POL-3).
- **Responsibilities**: `yaml.safe_load` on the text it is given; a pydantic
  boundary model that forbids unknown keys and bounds every value; conversion
  to the core `Policy`; a `PolicyError` that carries the reason; a dump
  function for the round-trip property (TST-PBT-3).
- **Interface**: `parse_policy(text) -> Policy`, `dump_policy(policy) -> str`.
- **Does not**: open the policy file. The composition root (C30) reads the
  file and passes the text, so the core stays free of I/O.

### C3 Check rules (`baseline/rules.py`)
- **Purpose**: one pure scoring function per check (FR-RULE-1 to FR-RULE-8).
- **Responsibilities**: inclusive freshness windows (FR-RULE-0c); per-OS
  logic; applicability by device class (FR-CLS-4); advisories beside a result
  that never change it (FR-RULE-0e); the unreliable-source downgrade from
  `fail` to `unverified` (FR-RULE-0d).
- **Interface**: `evaluate_<check>(evidence, rule, context, now) -> CheckResult`
  for each of the seven checks, `apply_unreliable_source(...)`, `is_fresh(...)`.
- **Quality target**: 100% branch coverage (NFR-Q-1).

### C4 Device scoring and status (`baseline/certification.py`)
- **Purpose**: score a whole device and give it one status.
- **Responsibilities**: class resolution in the order override, K2, TDX
  (FR-CLS-1, FR-CLS-2); "cannot score: device class unknown" (FR-CLS-3);
  "clean up: remove from tools" for devices not active in TDX (FR-SCOPE-2);
  calling every applicable rule; device status (certified, certified pending
  verification, not certified, baseline hygiene met or not met) and the
  "stopped reporting" flag (FR-CERT-1 to FR-CERT-4).
- **Interface**: `resolve_device_class(...)`, `device_status(...)`,
  `is_stopped_reporting(...)`, `score_device(evidence, policy, now) -> DeviceReport`.

### C5 Fleet computation (`baseline/fleet.py`)
- **Purpose**: the fleet rankings (FR-FLT-2 to FR-FLT-4).
- **Responsibilities**: per check, the personal productivity devices for which
  it is the only blocking result, and its total non-passing results; per
  catalog remediation, the devices it would certify and improve; checks with no
  remote fix; lists of stopped-reporting, class-unknown and clean-up devices.
- **Interface**: `summarize_fleet(reports, catalog, clean_up) -> FleetSummary`.
- **Types**: `FleetSummary`, `CheckGap`, `RemediationImpact`,
  `RemediationCatalogEntry`, `DeviceRef` (frozen dataclasses).
- **Droppable** (FR-FLT-6): only the scoring service's fleet method (C11) and
  the fleet page (C24) use it.

---

## Engine integration and scoring (`mcp/`, `scoring.py`)

### C6 Read client (`mcp/clients.py`)
- **Purpose**: the only connection to the itops-mcp read server.
- **Responsibilities**: connect with the read token only; typed methods for the
  read side of the engine contract (requirements section 6): list in-scope
  devices, list devices not active in TDX, resolve free text to candidates,
  evidence for one device and for the fleet; a generic call for allowlisted
  investigation tools; pydantic validation of every response; error mapping
  to `EngineUnavailable`, `EngineAuthError` naming the read server
  (FR-TOK-1) and `EngineRefusal` carrying the server's text verbatim.
- **Does not**: hold the act token or reach the act server.

### C7 Act client (`mcp/clients.py`)
- **Purpose**: the only connection to the itops-mcp act server.
- **Responsibilities**: connect with the act token only and register the
  approval store's elicitation handler (C9); `plan_remediation` (changes
  nothing, FR-REM-2) and `execute_remediation`; the same error mapping as C6.
- **Used by**: the act guard (C10) only.

### C8 Evidence adapter (`mcp/evidence.py`)
- **Purpose**: Q2 = A. The only place where engine evidence shapes meet core
  types, so a change from the forthcoming agent-core MCP template touches only
  this module.
- **Responsibilities**: pydantic models of the engine's normalized evidence;
  conversion to `DeviceEvidence`; an `EvidenceShapeError` when a response
  cannot be converted. A conversion failure means "not scored, with the
  reason", never a passing result (fail closed, SEC-REQ-12).
- **Interface**: `to_device_evidence(engine_evidence) -> DeviceEvidence`.

### C9 Approval store, the harness layer of FR-APR-8 (`mcp/approvals.py`)
- **Purpose**: hold each approval request from the act server as an open
  approval until a person answers it ("approval is a pause, not a value",
  `tech-env.md`). Security-critical module (SEC-REQ-14).
- **Responsibilities**:
  - **Elicitation handler** (registered with C7): validate the request against
    the itops-mcp approval schema; decline automatically on a mismatch, with
    the reason (FR-APR-2); decline automatically an elicitation for a plan no
    session is executing; otherwise open a pending approval (an
    `asyncio.Future` under a random id), emit "approval opened" with the page
    link (FR-APR-6), and wait.
  - **Answer**, called only by the approval route: accept with validated form
    content, decline or cancel. Each approval is answered at most once; an
    answer for an approval that is not open is refused as a security event
    (SEC-REQ-4).
  - **Timeout and shutdown**: an approval not answered within
    `APPROVAL_TIMEOUT_SECS`, or open when the console stops, closes as
    cancelled (FR-APR-5).
  - **Ledger**: a read-only record of every approval and its outcome, for the
    act guard.
- **Interface**: `expect(...)`, `handle_elicitation(...)`, `get_open(...)`,
  `list_open()`, `answer(...)`, `cancel_all(...)`, `ledger`.
- **Does not**: contain any path that accepts an approval other than
  `answer` called from the approval route. Every automatic answer is a
  decline.

### C10 Act guard, the console-side check of the server layer (`mcp/act_guard.py`)
- **Purpose**: Q6 = A. Separate from the approval store, it checks at run time
  that the act server never executes without an accepted answer.
  Security-critical module.
- **Responsibilities**: pass planning through to C7; before
  `execute_remediation`, register the expected plan with the approval store;
  after the result, look the plan up in the approval ledger. If the server
  reports an execution with no accepted answer for that plan, the outcome is
  "violation": nothing is reported as successful, and a security event is
  logged and alerted. Server refusals are returned verbatim (FR-EXE-1). The
  guard also checks that the result names the engine's no-op demo runner
  (FR-REM-3).
- **Interface**: `plan(...)`, `execute(...) -> ExecutionOutcome`.
- **Receives**: the approval store through a narrow interface (`expect` and
  `ledger`) with no answering method.

### C11 Scoring service (`scoring.py`)
- **Purpose**: the I/O around the pure core: fetch, adapt, score, emit.
- **Responsibilities**: score one device for a chat session as soon as it is
  identified (FR-DEV-3), with the real current time from an injected clock
  (FR-DEMO-2); emit the console-initiated scoring step and its results;
  compute the fleet summary for the fleet page (FR-FLT-1).
- **Interface**: `score_device(device_id, session_id) -> DeviceReport`,
  `score_fleet() -> FleetSummary`.

---

## Agent (`llm.py`, `agent/`)

### C12 LLM client (`llm.py`)
- **Purpose**: the only module that imports `openai`.
- **Responsibilities**: one `openai.AsyncOpenAI` client with `base_url`, key
  and model from settings (FR-LLM-1); one completion call returning text, tool
  calls and token usage; connection and authentication failures mapped to
  `LlmUnavailable` (FR-LLM-3). Gateway only; no offline model (DEV-1).
- **Interface**: `complete(messages, tools, max_tokens) -> ModelTurn`.

### C13 Tool registry (`agent/tools.py`)
- **Purpose**: decide exactly which engine tools the model can call.
- **Responsibilities**: a fixed allowlist of read-server investigation tools
  in console code (Q5 = A) plus `plan_remediation` (Q4 = A); a pydantic
  argument model with length limits for each tool (SEC-REQ-6); OpenAI tool
  schemas; dispatch; results returned as data, labeled as such (LLM01).
- **Does not**: offer `execute_remediation`. The registry receives a read
  interface and a planning interface only, never the act guard's execute
  method.
- **Interface**: `openai_tools()`, `names()`, `call(tool_call, session_id) -> ToolOutcome`.

### C14 Agent loop (`agent/loop.py`)
- **Purpose**: the hand-written tool-calling loop (FR-AGT-6).
- **Responsibilities**: run model turns and tool calls until the model answers
  or a step or token limit is reached (FR-AGT-5); emit every model turn, tool
  call and tool result as an event (FR-AGT-2); return the final text, why it
  stopped, and any plan IDs created.
- **Does not**: resolve devices, score, or execute (Q7 = A).
- **Interface**: `run(messages, session_id) -> AgentOutcome`.

### C15 Prompts (`agent/prompts.py`)
- **Purpose**: every text the model sees that the console writes.
- **Responsibilities**: the system prompt (tool output is data, never
  instructions; scoring results are fixed facts; recommend only catalog fixes
  or say "no reviewed fix exists"); the facts message built from a
  `DeviceReport` (FR-DEV-3); wrapping of tool results; the fleet narrative
  prompt. Holds no secrets and no scope rules (LLM07).

### C16 Fleet narrative writer (`agent/narrative.py`)
- **Purpose**: the optional model-written fleet summary (FR-FLT-5).
- **Responsibilities**: one model call with no tools, built from the computed
  `FleetSummary` only; result labeled model-written; generated only on
  request; recorded as its own short transcript.
- **Interface**: `write(summary) -> Narrative`.

---

## Terminal chat (`chat/`)

### C17 Chat session service (`chat/session.py`)
- **Purpose**: Q7 = A. Owns one chat session from start to end.
- **Responsibilities**: start and end the session (one transcript per
  session, FR-TRN-1); resolve the device through C18; run scoring through
  C11 as soon as the device is known; give the fixed results and the session
  history to the agent loop; keep the plan IDs created in this session; run
  `execute` for a plan only when the technician types the command (Q4 = A).
- **Interface**: `start()`, `list_devices()`, `choose(number)`,
  `identify(text)`, `ask(question)`, `execute(plan_id)`, `end()`.

### C18 Device identity resolver (`chat/identity.py`)
- **Purpose**: code, not the model, turns the technician's words into one
  engine device ID (FR-DEV-1, FR-DEV-2).
- **Responsibilities**: the numbered device list; free-text resolution
  through the read server; a pure decision: one match is selected and named,
  several give a numbered pick-list, none gives "no matching device". It never
  guesses.
- **Interface**: `list_devices()`, `resolve(text) -> Identification`,
  `decide(candidates) -> Identification` (pure).

### C19 Terminal command loop (`chat/terminal.py`)
- **Purpose**: the plain line-by-line prompt (FR-CHAT-1).
- **Responsibilities**: read lines without blocking the event loop
  (NFR-P-1); parse each line into a command (device list, pick by number,
  name a device, execute a plan, help, quit) or a question; call the session
  service. Command syntax is set in Functional Design.
- **Does not**: print. All output goes through events (Q9 = A).
- **Interface**: `run()`, `handle(command)`, `parse_command(line)` (pure).

### C20 Terminal printer (`chat/printer.py`)
- **Purpose**: the only writer to the terminal after startup.
- **Responsibilities**: an event sink that formats each event as text;
  security events print as alert lines (SEC-REQ-11); terminal control
  characters and escape sequences are stripped from tool output and model text
  before printing, so device data cannot rewrite the screen (MS-8).
- **Interface**: `handle(event)`, `format_event(event)` (pure), `sanitize(text)` (pure).

---

## Web (`app.py`, `security.py`, `templates/`, `static/`)

### C21 Web app factory and routes (`app.py`)
- **Purpose**: the local web server's pages and form handlers.
- **Responsibilities**: `create_app(deps)` builds the FastAPI app with
  `/docs`, `/redoc` and `/openapi.json` disabled (SEC-REQ-7); installs the
  request security middleware and error handlers from C22; reads its
  components from `app.state`, never from module imports (Q8 = A); validates
  every path, query and form value with pydantic and explicit length limits
  (SEC-REQ-6).
- **Routes**:

| Method | Path | Access | Purpose | Requirement |
|---|---|---|---|---|
| GET | `/health` | public | Liveness only; returns no data | SEC-REQ-2 |
| GET | `/start?token=` | launch token | Exchange the launch token for a session cookie, then redirect to `/` so the token leaves the address bar | SEC-REQ-2 |
| GET | `/` | session | Home: open approvals and links to the other pages | |
| GET | `/approvals/{approval_id}` | session | Approval page | FR-APR-1, FR-APR-3, FR-APR-7 |
| POST | `/approvals/{approval_id}` | session, origin, CSRF | Approve, decline or cancel | FR-APR-4, SEC-REQ-3, SEC-REQ-4 |
| GET | `/fleet` | session | Fleet summary | FR-FLT-1 to FR-FLT-4 |
| POST | `/fleet/narrative` | session, origin, CSRF | Model-written narrative, on request | FR-FLT-5 |
| GET | `/transcripts` | session | List of past sessions | FR-TRN-2 |
| GET | `/transcripts/{session_id}` | session | One transcript, read-only | FR-TRN-2 |
| GET | `/static/htmx.min.js` | session | Vendored HTMX, used by the fleet page only | SEC-REQ-9 |

No route edits or deletes a record. No route sends CORS headers.

### C22 Request security (`security.py`)
- **Purpose**: every control between a browser and a route.
  Security-critical module (SEC-REQ-14).
- **Responsibilities**:
  - **Launch session** (SEC-REQ-2, SECURITY-12): a random launch token per
    start; its link printed once in the terminal; exchange for a session
    cookie (`HttpOnly`, `SameSite=Strict`, `Secure` if the target browsers
    accept it on `127.0.0.1`); a server-side session that ends when the
    console stops; the launch token is invalidated after repeated failed
    exchanges, with an alert (brute-force protection).
  - **Deny by default**: every route except `/health` and `/start` requires
    the session (SECURITY-08).
  - **Origin and CSRF** checks on every POST (SEC-REQ-3).
  - **Security headers** on every HTML response, CSP compatible with HTMX
    without `unsafe-inline` (SEC-REQ-5).
  - **Request body size cap** (SEC-REQ-6).
  - **Generic error handlers**: no stack traces or paths (SEC-REQ-7,
    SEC-REQ-12).
  - **Static integrity**: the vendored HTMX file is checked against its
    recorded sha256 at startup (SEC-REQ-9).
  - A security event for every refusal (SEC-REQ-11).
- **Interface**: `LaunchSession`, `CsrfTokens`, `require_session`,
  `require_same_origin`, `require_csrf`, `SecurityHeadersMiddleware`,
  `BodySizeLimitMiddleware`, `install_error_handlers`,
  `verify_static_integrity`.

### C23 Approval page (`templates/approval.html`, routes in `app.py`)
- **Purpose**: the screen where a person reads and answers an approval.
- **Responsibilities**: every target in full with no truncation, scrolling
  region or collapsing (FR-APR-3); the remediation name, script identity and
  approval code (FR-APR-1); the notices "demo runner: no device is changed"
  (FR-REM-3) and "the console cannot prove a person clicked" (FR-APR-7); an
  approve control with no default, plus decline and cancel (FR-APR-4). A plain
  HTML form that loads no script, operable by keyboard with visible focus,
  semantic markup, outcomes written as words (NFR-A-1 to NFR-A-3).

### C24 Fleet page (`templates/fleet.html`, routes in `app.py`)
- **Purpose**: the endpoint engineer's view (FR-FLT-1 to FR-FLT-5).
- **Responsibilities**: tables for gaps by check and remediations by devices
  certified and improved; the three separate device lists; a "write a
  narrative" button that loads the labeled model text with HTMX. Droppable as
  a unit with C5 and C16 (FR-FLT-6).

### C25 Transcript viewer (`templates/transcripts.html`, `templates/transcript.html`, routes in `app.py`)
- **Purpose**: the lead's read-only view of past sessions (FR-TRN-2).
- **Responsibilities**: list sessions with start time and device; show one
  transcript with scoring steps, tool calls and results, approvals, answers and
  engine audit references; no edit or delete controls; session IDs checked
  against a fixed format before any file is read.

---

## Cross-cutting

### C26 Settings (`settings.py`)
- **Purpose**: every setting, validated at startup (FR-TOK-1, US-5.6).
- **Responsibilities**: a pydantic model of every variable in `.env.example`;
  secrets as `SecretStr`; the console refuses to start when a value is
  missing or invalid and names it; engine URLs must be loopback and the gateway
  URL must be `https` (SEC-REQ-1, SEC-REQ-13); the web host is fixed at
  `127.0.0.1` and is not a setting.
- **Variables**: the `tech-env.md` table minus the Ollama example (DEV-2),
  plus optional `WEB_PORT` (default 8080), `TRANSCRIPT_DIR` and `LOG_DIR`.
- **Interface**: `load_settings(environ, env_file) -> Settings`.

### C27 Session events and dispatcher (`events.py`)
- **Purpose**: Q9 = A. One event type for everything the console shows or
  records.
- **Responsibilities**: the `SessionEvent` type and the list of event kinds;
  an `EventBus` that redacts known secret values, then hands each event to
  every sink in a fixed order: transcript, log, terminal. Nothing can be
  printed without being recorded, or recorded without being printed.
- **Interface**: `EventBus.emit(...)`, `EventBus.security(...)`, the
  `EventSink` protocol.

### C28 Transcript store (`transcript.py`)
- **Purpose**: one JSON Lines file per session (FR-TRN-1), never rewritten
  (FR-TRN-3).
- **Responsibilities**: a writer sink that opens files only in append mode; a
  reader for the viewer; line encoding and decoding that round-trips
  (TST-PBT-3). No method rewrites or deletes a file.
- **Interface**: `TranscriptWriter.handle(event)`,
  `TranscriptReader.list_sessions()`, `TranscriptReader.read(session_id)`,
  `to_line(record)`, `from_line(line)`.

### C29 Structured logging and security alerts (`logs.py`)
- **Purpose**: SEC-REQ-10 and SEC-REQ-11.
- **Responsibilities**: JSON log lines (timestamp, session or request ID,
  level, message) to one local log directory; a redaction filter for secret
  values; retention of at least 90 days; a log sink for events that writes
  security events at WARNING. The terminal alert comes from the printer (C20)
  receiving the same event.
- **Interface**: `configure_logging(...)`, `JsonFormatter`, `RedactingFilter`,
  `LogSink`.

### C30 Composition root and entry point (`main.py`)
- **Purpose**: Q8 = A. The one place that builds components.
- **Responsibilities**: load settings; configure logging; read the policy
  file and parse it; check the HTMX hash; build every component once and pass
  each one explicitly; connect the MCP clients; print the launch link; run the
  web server and the terminal loop on one event loop (NFR-P-1); on quit or
  signal, cancel open approvals, end the launch session and close clients.
- **Interface**: `build(...) -> Console`, `serve(console)`, `run()` (the
  `itops-console` console script).
- No other module creates a shared component; there are no module-level
  singletons, so tests build their own with fakes.

---

## Outside the package

### C31 Scenario suite runner (`evals/run.py`)
- **Purpose**: measure the agent (FR-EVAL-1, FR-EVAL-2, NFR-Q-3, NFR-Q-4).
- **Responsibilities**: load the ten approved scenario files with
  `yaml.safe_load`; build the console through `main.build` with the eval
  model (FR-LLM-2) and no web server; run each scenario's question through a
  chat session; grade cause and fix; write results per model and date to
  `evals/results/`. Never executes a remediation. Not run in CI.
- **Interface**: `load_scenarios(...)`, `run_scenario(...)`, `grade(...)`,
  `write_results(...)`, `main(argv)`.
