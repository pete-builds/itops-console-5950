# Application Design Plan

**Stage**: INCEPTION - Application Design
**Created**: 2026-11-19
**Inputs**: `requirements.md`, `stories.md`, `personas.md`, `execution-plan.md`,
`docs/tech-env.md` (project structure, architecture and patterns)
**Depth**: Standard. Every component is new, but the stack, the process model
and the module list are already set by `tech-env.md`. This stage fixes the
component boundaries, the interfaces between them and the orchestration;
business rules wait for Functional Design.

**Answer mode** (session notes): questions are asked in the chat one at a time;
each answer is written into its `[Answer]:` tag below, which stays the record.

---

## Plan steps

### Part 1: Planning
- [x] 1. Analyze context: requirements (FR, NFR, SEC-REQ, TST-PBT), 30 stories and 6 misuse stories, execution plan (FR-APR-8 layers in separate modules), tech-env structure
- [x] 2. Draft this plan with checkboxes
- [x] 3. Include the mandatory artifacts (Part 2 below)
- [x] 4. Write design questions (Q1 to Q9 below)
- [x] 5. Collect answers to all questions
- [x] 6. Analyze answers for ambiguity or contradiction; add follow-ups if needed (none needed; see Answer analysis)
- [x] 7. Plan approval (approved 2026-11-24)

### Part 2: Generation (`aidlc-docs/inception/application-design/`)
- [x] 8. `components.md`: each component's name, purpose, responsibilities and interface
  - [x] 8a. Baseline core: models, policy loader, rules, certification, fleet
  - [x] 8b. Engine integration: read client, act client, approval store, evidence adapter (plus act guard and scoring service)
  - [x] 8c. Agent: loop, prompts, LLM client, tool registry (plus fleet narrative writer)
  - [x] 8d. Terminal chat: command loop, device identity resolution, output printer (plus chat session service)
  - [x] 8e. Web: app factory, request security, approval page, fleet page, transcript viewer
  - [x] 8f. Cross-cutting: settings, transcript writer, structured logging and security alerts (plus events and composition root)
  - [x] 8g. Scenario suite runner (outside the package)
- [x] 9. `component-methods.md`: method signatures, purpose, input and output types (no business rules)
- [x] 10. `services.md`: services and orchestration (chat session, scoring, remediation and approval, fleet computation, startup and shutdown)
- [x] 11. `component-dependency.md`: dependency matrix, communication patterns, data-flow diagrams (Mermaid validated, text alternative included)
- [x] 12. `application-design.md`: consolidated design, including:
  - [x] 12a. FR-APR-8 two-layer placement (server layer and harness layer in separate modules)
  - [x] 12b. SECURITY-11 security-critical modules and misuse cases MS-1 to MS-6 mapped to components (plus MS-7 to MS-9 found in design)
  - [x] 12c. Requirement and story traceability to components
- [x] 13. Validate completeness and consistency (every FR and story maps to a component; no cycles in the dependency matrix; the baseline core imports no I/O). One module cycle found and removed
- [x] 14. Extension compliance summary (Security Baseline; PBT Partial). No blocking findings
- [x] 15. Completion message and approval (approved 2026-11-24)

---

## Questions

## Question 1
How should the package be laid out?

A) Keep the `tech-env.md` project structure as the base and add the modules it lacks (for example `security.py`, a terminal chat package, a device identity resolver, an evidence adapter), with no renaming

B) Reorganize into layers (`domain/`, `adapters/`, `services/`, `web/`, `cli/`), mapping each `tech-env.md` module into a layer

C) Reorganize by feature (`check_device/`, `remediate/`, `fleet/`, `transcripts/`), each holding its own models, services and routes

D) Other (please describe after [Answer]: tag below)

[Answer]: A (keep the tech-env.md structure; no renaming)

## Question 2
How should engine responses reach the rule engine? (The forthcoming agent-core MCP template may change the engine's response shapes.)

A) An adapter in the engine-integration layer converts the engine's normalized evidence into the console's own evidence types; the rule engine depends only on console types, so an engine change touches only the adapter

B) The rule engine consumes the engine's normalized evidence schema directly (fewer types, but engine changes reach the rules)

C) Other (please describe after [Answer]: tag below)

[Answer]: A (an evidence adapter keeps the template's timing risk inside one module)

## Question 3
Which type style should the baseline core (evidence, rules, results) use?

A) Frozen dataclasses inside the baseline core, as in the `tech-env.md` example (pure, cheap, strict mypy), with pydantic only at the boundaries (policy file, settings, web forms, engine responses)

B) Pydantic models everywhere, including the rule engine's inputs and outputs

C) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 4
Which engine tools can the model call, and who starts an execution?

A) The model gets the read-server tools and `plan_remediation` (planning changes nothing). Only a terminal command typed by the technician calls `execute_remediation`; the approval page then gates it. The model can never start an execution

B) The model gets the read tools, `plan_remediation` and `execute_remediation`; the approval page is the only gate on execution

C) The model gets read tools only; planning and execution are both terminal commands typed by the technician

D) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 5
Which read-server tools does the agent see?

A) A fixed allowlist in console code (the per-system investigation tools named in requirements section 6); a tool the engine adds later is not offered to the model until it is added to the list

B) Every tool the read server lists at connect time, passed through unchanged

C) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 6
FR-APR-8 asks for the server layer and the harness layer to be shown refusing separately. Beyond tests, should the console cross-check the act server at run time?

A) Yes: the act client wrapper (separate from the approval store) refuses to report success, alerts and records a security event if `execute_remediation` returns an execution for which the approval store holds no accepted answer. The approval store stays the only thing that answers an elicitation

B) No: the server layer is checked only in tests against a misbehaving fake act server; at run time the console trusts the act server's result

C) Other (please describe after [Answer]: tag below)

[Answer]: A (run-time cross-check in the act client wrapper)

## Question 7
How should one chat session be orchestrated?

A) A chat session service owns one session: it resolves the device, runs scoring, hands fixed results to the agent loop, routes remediation commands and holds the session's transcript. The agent loop only runs model turns and tool calls

B) The agent loop is the orchestrator: identity, scoring and remediation are steps inside it

C) The terminal command loop orchestrates directly, calling each component in turn with no session service

D) Other (please describe after [Answer]: tag below)

[Answer]: A (chat session service; the agent loop only runs model turns and tool calls)

## Question 8
How are shared components (settings, MCP clients, approval store, transcript writer) created and passed around?

A) One composition root at startup builds every component once and passes each one explicitly (constructor arguments; the web app reads them from `app.state`). No module-level singletons, so tests build their own with fakes

B) Module-level singletons imported where needed, as in the `tech-env.md` endpoint example (`from itops_console.transcript import transcript`)

C) Other (please describe after [Answer]: tag below)

[Answer]: A (one composition root; no module-level singletons)

## Question 9
Every step must appear in the terminal and the transcript (FR-AGT-2), and security events must be logged and alerted in the terminal (SEC-REQ-11). How should those outputs be produced?

A) One in-process event type: components emit a session event once, and a small dispatcher sends it to the transcript writer, the terminal printer and the logger. Nothing can be printed without being recorded, or the reverse

B) Direct calls: each component calls the transcript writer, the printer and the logger itself

C) Other (please describe after [Answer]: tag below)

[Answer]: A (one session event type, fanned out by a dispatcher)

---

## Answer analysis (step 6)

All nine answers pick a single listed option (Q1 to Q9 = A). None is vague,
combines options or leaves a criterion undefined. Checked for conflicts:

- Q1 (keep the `tech-env.md` structure) and Q8 (composition root, no
  singletons): compatible. Q8 changes how the `tech-env.md` endpoint example
  gets its objects (from `app.state`, not a module import); it renames nothing.
- Q4 (only a technician command starts an execution) and Q6 (run-time
  cross-check): compatible. Q6 adds a check after the act server answers; it
  gives the act client no way to answer an approval.
- Q7 (session service) and Q9 (one event type): compatible. The session
  service and the agent loop both emit events; neither writes to the terminal
  or transcript directly.
- Q2 (evidence adapter) and Q3 (frozen dataclasses in the core): compatible.
  The adapter is the boundary where pydantic engine models become core
  dataclasses.

No follow-up questions needed.
