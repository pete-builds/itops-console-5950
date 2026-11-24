# Units of Work: itops-console

**Stage**: INCEPTION - Units Generation
**Created**: 2026-11-24
**Plan**: `aidlc-docs/inception/plans/unit-of-work-plan.md` (Q1 to Q5 = A, approved 2026-11-24)
**Inputs**: `application-design.md` and its companion documents (components C1 to C31), `stories.md`, `execution-plan.md`

itops-console is one Python process (a monolith). Each unit of work below is a
group of modules and stories inside the one `itops_console` package. Units are
designed and built one at a time in the Construction phase, each through
Functional Design, NFR Requirements, NFR Design and Code Generation; they are
not deployed separately.

---

## Summary

| Unit | Name | Components | Primary stories | Size | Build order |
|---|---|---|---|---|---|
| U1 | Baseline core | C1 to C4 | 9 user stories, MS-5 | Large | 1 |
| U2 | Platform and engine integration | C6 to C11 (device scoring), C26 to C30 | 6 user stories, MS-3, MS-6 | Large | 2 (after the spike) |
| U3 | Agent and terminal chat | C12 to C15, C17 to C20 | 6 user stories, MS-4 | Medium | 3 |
| U4 | Web and approval page | C21 to C23, C25 | 4 user stories, MS-1, MS-2 | Medium | 4 |
| U6 | Scenario suite | C31 | 1 user story | Small | 5 |
| U5 | Fleet summary (droppable) | C5, C16, C24, plus fleet additions to C11 and C21 | 4 user stories | Small to medium | 6 (last) |

Every component belongs to exactly one unit. Three modules are created by one
unit and extended by later ones, always by adding, never by changing what was
built: `main.py` (created in U2; each later unit adds its wiring), `scoring.py`
(created in U2; U5 adds `score_fleet`) and `app.py` (created in U4; U5 adds the
two fleet routes).

**Before U2: risk spike** (Q4 = A). Two throwaway scripts retire the two
unverified integration risks before U2's Functional Design. See
`unit-of-work-dependency.md`.

---

## U1 Baseline core

- **Purpose**: the deterministic heart of the product: policy, the seven
  checks, device class, certification. Pure code with no I/O, no clock reads
  and no model calls.
- **Components**: C1 core types, C2 policy parser, C3 check rules, C4 device
  scoring and status.
- **Modules**: `baseline/models.py`, `baseline/policy.py`,
  `baseline/rules.py`, `baseline/certification.py`;
  `policy/baseline.example.yaml`.
- **Also delivers** (first unit built): repository scaffolding:
  `pyproject.toml` (Python 3.12, capped major versions), `uv.lock` with
  hashes, ruff and mypy configuration (strict on `baseline/`), the CI workflow
  with the `tech-env.md` gates (ruff, format, mypy, pytest with coverage,
  pip-audit, gitleaks, forbidden-terms check, SBOM), `tests/support/` with
  the shared Hypothesis domain strategies (TST-PBT-4), and
  `tests/architecture/` with rules B1 and B9.
- **Requirements**: FR-POL-1 to FR-POL-3, FR-SCOPE-2 (clean-up status),
  FR-CLS-1 to FR-CLS-4, FR-RULE-0a to FR-RULE-0e, FR-RULE-1 to FR-RULE-8,
  FR-CERT-1 to FR-CERT-4; NFR-Q-1, NFR-Q-2; TST-PBT-1 to TST-PBT-5;
  SEC-REQ-8 (CI), SEC-REQ-9 (safe YAML).
- **Functional Design depth**: full. This unit holds most of the product's
  business logic and its "Testable Properties" section (PBT-01).
- **Definition of done** (in addition to the common one below): 100% branch
  coverage of `baseline/rules.py`; a boundary test for every threshold and
  freshness window; per-class tests; the invariant and policy round-trip
  properties; MS-5 refusal tests.

## U2 Platform and engine integration

- **Purpose**: everything the other units stand on: settings, events,
  transcripts, logs, the composition root, both MCP clients, the evidence
  adapter, device scoring with I/O, and both approval layers (built and tested
  against fakes).
- **Components**: C6 read client, C7 act client, C8 evidence adapter, C9
  approval store, C10 act guard, C11 scoring service (`score_device`), C26
  settings, C27 events and dispatcher, C28 transcript store, C29 logging and
  security alerts, C30 composition root.
- **Modules**: `mcp/clients.py`, `mcp/evidence.py`, `mcp/approvals.py`,
  `mcp/act_guard.py`, `scoring.py`, `settings.py`, `events.py`,
  `transcript.py`, `logs.py`, `main.py`; `.env.example`.
- **Also delivers**: the shared test support (Q3 = A): in-process fake read
  and act MCP servers (the act fake records every execution and can be told
  to misbehave), recorded engine responses, and a scripted fake model for U3;
  architecture rules B2 (fastmcp), B3, B6 and B8.
- **Requirements**: FR-DEV-3 (scoring step), FR-SCOPE-1, FR-APR-2, FR-APR-5,
  FR-APR-6 (event), FR-APR-8, FR-EXE-1 (guard), FR-REM-3 (runner check),
  FR-TRN-1, FR-TRN-3, FR-TOK-1, FR-DEMO-2 (real clock); NFR-Q-6, NFR-Q-7;
  SEC-REQ-1 (settings), SEC-REQ-4, SEC-REQ-10 to SEC-REQ-14.
- **Functional Design depth**: full for the approval store (states and
  transitions, timeout, single answer, automatic declines) and the act guard;
  standard for the rest.
- **Definition of done**: the approval safety suite against the recording
  fake act server: zero executions on decline, cancel, timeout, replay,
  schema mismatch and unexpected elicitation; a control test with the approval
  check removed fails (NFR-Q-6); the act guard flags a fake server that
  executes without an accepted answer (FR-APR-8 server layer); settings
  refusal tests naming each problem; the transcript round-trip property;
  secrets absent from logs and transcripts (MS-6, partial).
- **Open items decided here**: 3 (`.env` loading), 5 (fastmcp elicitation
  shape, informed by the spike), 8 (engine contract fields, raised in the
  engine repository), 9 (engine unreachable at startup).

## U3 Agent and terminal chat

- **Purpose**: the technician's read-only experience: identify a device, see
  it scored, ask why, get an explanation and a catalog recommendation; plan a
  fix (changes nothing); type the execute command.
- **Components**: C12 LLM client, C13 tool registry, C14 agent loop, C15
  prompts, C17 chat session service, C18 device identity resolver, C19
  terminal command loop, C20 terminal printer.
- **Modules**: `llm.py`, `agent/tools.py`, `agent/loop.py`,
  `agent/prompts.py`, `chat/session.py`, `chat/identity.py`,
  `chat/terminal.py`, `chat/printer.py`; wiring added to `main.py`.
- **Also delivers**: architecture rules B2 (openai), B4, B5 and B7.
- **Requirements**: FR-CHAT-1, FR-DEV-1 to FR-DEV-3, FR-AGT-1 to FR-AGT-6,
  FR-REM-1, FR-REM-2, FR-LLM-1, FR-LLM-3; SEC-REQ-15.
- **Functional Design depth**: standard (session states, command syntax,
  identity decision, loop exits).
- **Definition of done**: the agent loop against the scripted fake model
  covers every exit (answered, step limit, token limit, model unavailable);
  the registry refuses tools outside the allowlist and offers no execute tool;
  `decide` and `sanitize` properties; the MS-4 injected-instructions fixture
  leaves scores unchanged and answers no approval.
- **Open items decided here**: 1 (read-tool allowlist), 10 (command syntax).
- **Note**: until U4 exists, no route can answer an approval, so an execute
  command can only time out as cancelled. End-to-end approval is tested in
  U4.

## U4 Web and approval page

- **Purpose**: the human-in-the-loop step and the record: the approval page,
  the transcript viewer, and the request security around every page.
- **Components**: C21 web app factory and routes, C22 request security, C23
  approval page, C25 transcript viewer.
- **Modules**: `app.py`, `security.py`; `templates/home.html`,
  `approval.html`, `approval_answered.html`, `transcripts.html`,
  `transcript.html`, `error.html`; wiring added to `main.py`.
- **Also delivers**: architecture rule B3 completed (the approval route is
  the only caller of `answer`).
- **Requirements**: FR-APR-1, FR-APR-3, FR-APR-4, FR-APR-7, FR-REM-3 (page
  notice), FR-EXE-1 (end to end), FR-TRN-2; NFR-Q-5, NFR-A-1 to NFR-A-4;
  SEC-REQ-1 to SEC-REQ-7.
- **Functional Design depth**: standard; NFR Design carries most of the work
  (session, CSRF, headers, CSP).
- **Definition of done**: every route except `/health` and `/start` refuses
  without the session; foreign origin, missing CSRF and replayed answers are
  refused with alerts and zero executions (MS-1, MS-3); 25- and 100-target
  plans render every target, the remediation, the script identity and the
  approval code (NFR-Q-5); keyboard operation and semantic markup checks;
  security headers on every HTML response; docs routes absent; generic error
  page; transcript ID traversal refused (MS-9); the full approve, decline,
  cancel and timeout flow end to end with the fakes.
- **Open items decided here**: 4 (`Secure` cookie on `127.0.0.1`), 7 (launch
  token rules).

## U5 Fleet summary (droppable)

- **Purpose**: the endpoint engineer's fleet view and the optional narrative.
  First feature to drop if time runs short (FR-FLT-6).
- **Components**: C5 fleet computation, C16 fleet narrative writer, C24 fleet
  page; additions to C11 (`score_fleet`, `CatalogSource`) and C21 (`/fleet`,
  `/fleet/narrative`).
- **Modules**: `baseline/fleet.py`, `agent/narrative.py`;
  `templates/fleet.html`, `templates/fleet_narrative.html`;
  `static/htmx.min.js` with its recorded sha256 and the startup check
  (HTMX is used only by this page).
- **Requirements**: FR-FLT-1 to FR-FLT-6; NFR-P-2; SEC-REQ-9 (HTMX hash).
- **Functional Design depth**: standard (ranking rules, catalog source).
- **Definition of done**: fleet invariants (counts never exceed in-scope
  devices; class-unknown and stopped-reporting devices never counted as one
  fix from certified); render under 5 seconds on the fixture fleet; no model
  text until requested, then labeled model-written.
- **Open items decided here**: 2 (catalog source; an engine catalog tool is
  preferred because it needs no change to U1's policy model).
- **What dropping U5 means**: none of the modules above is created, and
  `score_fleet` and the two routes are not added. U1 to U4 and U6 are
  unchanged; HTMX and its hash check are never added. FR-FLT-1 to FR-FLT-5 and
  US-3.1 to US-3.4 are then recorded as dropped under FR-FLT-6.

## U6 Scenario suite

- **Purpose**: measure the agent's explanations and recommendations.
- **Components**: C31 scenario suite runner.
- **Modules**: `evals/run.py`, `evals/scenarios/` (ten files),
  `evals/results/`.
- **Requirements**: FR-EVAL-1, FR-EVAL-2, FR-LLM-2; NFR-Q-3, NFR-Q-4.
- **Functional Design depth**: minimal (scenario file format, grading method).
- **Definition of done**: the ten scenarios drafted from the fixture fleet and
  approved by the user before any run (FR-EVAL-1); the runner tested against
  the scripted fake model; never executes a remediation. Real runs (Sonnet 5.5
  routinely, Opus 5.5 before each milestone) need the live engine and the
  gateway, so they happen in Build and Test, with results committed per model
  and date.
- **Open items decided here**: 6 (grading method).

---

## Common definition of done (every unit)

1. Functional Design, NFR Requirements and NFR Design approved for the unit.
2. Code Generation plan approved; Part 2 starts only after the stakeholder
   gate is cleared (`aidlc-state.md`, Session Notes).
3. `ruff check`, `ruff format --check`, `mypy` and `pytest` pass; the unit's
   architecture rules are active; no test calls a model or a live server.
4. Coverage moves toward 85% overall (NFR-Q-9), with each unit's own target
   above.
5. Security Baseline and PBT (Partial) compliance summary with no blocking
   findings.
6. README updated for anything the unit adds to setup or use.
7. Commit and push at the unit's approval gate.

---

## Code organization strategy (greenfield, Q5 = A)

The repository keeps the `tech-env.md` layout. Each unit is a group of
modules inside the one `itops_console/` package; tests are grouped by kind as
in `tech-env.md`, plus two shared folders.

```
itops_console/
  baseline/models.py, policy.py, rules.py, certification.py ... U1
  baseline/fleet.py ........................................... U5
  mcp/clients.py, evidence.py, approvals.py, act_guard.py ..... U2
  settings.py, events.py, transcript.py, logs.py .............. U2
  scoring.py .................................. U2 (U5 adds score_fleet)
  main.py ......................... U2 (U3, U4, U5 add their wiring)
  llm.py ...................................................... U3
  agent/tools.py, loop.py, prompts.py ......................... U3
  agent/narrative.py .......................................... U5
  chat/session.py, identity.py, terminal.py, printer.py ....... U3
  app.py ............................ U4 (U5 adds the fleet routes)
  security.py ................................................. U4
  templates/ (home, approval, approval_answered,
              transcripts, transcript, error) ................. U4
  templates/ (fleet, fleet_narrative) ......................... U5
  static/htmx.min.js .......................................... U5
policy/baseline.example.yaml .................................. U1
evals/run.py, evals/scenarios/, evals/results/ ................ U6
spikes/ ............................ risk spike, never imported
tests/unit/ ............ pure functions (U1; pure helpers of U2, U3)
tests/mcp/ ............. clients, adapter, approvals, act guard (U2)
tests/agent/ ........... loop, tools, session, terminal (U3)
tests/web/ ............. routes and pages (U4, U5)
tests/support/ ......... strategies (U1); fakes, recordings (U2)
tests/architecture/ .... import-boundary rules B1 to B9 (U1 onward)
pyproject.toml, uv.lock, .github/workflows/ ................... U1
.env.example .................................................. U2
```

**Difference from the AI-DLC default**: for a multi-unit monolith,
`code-generation.md` suggests `src/{unit-name}/` and `tests/{unit-name}/`.
This project keeps the `tech-env.md` layout instead (Application Design
Q1 = A, Units Generation Q5 = A): modules keep their `tech-env.md` names and
the console script stays `itops-console`. Code Generation plans must use the
paths above. Application code never goes in `aidlc-docs/`.

---

## Refinements to the Application Design made in this stage

Both follow from this stage's answers and were applied to the approved
design documents:

1. The fleet-only types (`FleetSummary`, `CheckGap`, `RemediationImpact`,
   `RemediationCatalogEntry`, `DeviceRef`) move from `baseline/models.py`
   (C1) to `baseline/fleet.py` (C5), so dropping U5 leaves no unused types in
   U1 (Q1 = A).
2. `tests/support/` starts in U1 with the Hypothesis domain strategies, which
   U1's property tests need before U2 exists; U2 adds the fakes and recorded
   responses (Q3 = A).

---

## Validation (plan step 11)

| Check | Result |
|---|---|
| Every component in exactly one unit | Yes. U1: C1 to C4. U2: C6 to C11, C26 to C30. U3: C12 to C15, C17 to C20. U4: C21 to C23, C25. U5: C5, C16, C24. U6: C31. All 31 assigned once; C11, C21 and C30 are extended additively by later units |
| Every story assigned | Yes: 30 user stories and 6 misuse stories, each with one primary unit (`unit-of-work-story-map.md`) |
| No cycles between units | Yes. Nothing depends on U5 or U6; U4 does not depend on U3 (`unit-of-work-dependency.md`) |
| Build order respects dependencies | Yes: U1, spike, U2, U3, U4, U6, U5; each unit's dependencies are built before it |
| Fleet summary droppable | Yes. Skipping U5 changes no module another unit built; HTMX is never added |
| Security-critical modules each owned by one unit | Yes: `baseline/` U1; `mcp/approvals.py` and `mcp/act_guard.py` U2; `agent/tools.py` U3; `security.py` U4 |
| Diagrams | One Mermaid flowchart checked by hand (alphanumeric IDs, quoted labels, valid edge syntax) with a text alternative; the code-organization tree and the text alternative use plain ASCII only |

## Extension compliance (plan step 12)

### Security Baseline (enabled)

| Rule | Status | Rationale |
|---|---|---|
| SECURITY-10 Supply chain | Compliant (planned) | U1's scaffolding delivers the lock file with hashes, pip-audit, SBOM, SHA-pinned Actions and gitleaks before any other unit's code |
| SECURITY-11 Secure design | Compliant | Each security-critical module belongs to one unit, and that unit's definition of done includes its misuse tests (MS-1 to MS-9 assigned in the story map) |
| SECURITY-13 Integrity | Compliant (planned) | CI pipeline in U1; HTMX and its hash check arrive with U5, the only unit that uses it |
| SECURITY-01 to 09, 12, 14, 15 | N/A at this stage | No new security controls are designed here; each is carried by the unit that owns its components, as mapped in `application-design.md` section 6.3 |

No blocking security findings.

### Property-Based Testing (Partial)

| Rule | Status | Rationale |
|---|---|---|
| PBT-02 Round-trip | Compliant | Policy round trip in U1; transcript round trip in U2 |
| PBT-03 Invariants | Compliant | Rule and certification invariants in U1; `decide` and `sanitize` in U3; fleet count invariants in U5 |
| PBT-07 Generator quality | Compliant (planned) | Shared domain strategies in `tests/support/` from U1 (TST-PBT-4) |
| PBT-08 Shrinking and reproducibility | Compliant (planned) | Fixed-seed CI profile and seed printing set up in U1's scaffolding |
| PBT-09 Framework | Compliant | Hypothesis added as a development dependency in U1 |
| PBT-01, 04, 05, 06, 10 | Advisory | Property lists are written in each unit's Functional Design |

### Resiliency Baseline

Disabled (Q30 = B); skipped.
