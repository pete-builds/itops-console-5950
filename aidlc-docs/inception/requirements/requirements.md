# Requirements: itops-console

**Stage**: INCEPTION - Requirements Analysis
**Depth**: Comprehensive
**Created**: 2026-11-19
**Inputs**: `docs/vision.md`, `docs/tech-env.md`,
`aidlc-docs/inception/requirements/requirement-verification-questions.md`
(Questions 1 to 31, all answered)

Requirement IDs are stable and are referenced by later stages. "Q n" points to
the question in the verification file that decided the requirement. "Draft"
marks text the user asked the AI to draft for review (Q5).

---

## 1. Intent analysis

| Item | Value |
|---|---|
| User request | Build itops-console, the local AI assistant described in `docs/vision.md`: a technician asks whether a computer meets the endpoint security baseline and why not; code scores each check; an LLM agent gathers evidence and explains; remediation runs only after a person approves the exact plan |
| Request type | New project (greenfield) |
| Scope estimate | Multiple components in one repository (rule engine, policy loader, agent loop, MCP clients, approval web page, terminal chat, fleet summary, transcript viewer, scenario suite), plus a contract with an external engine (itops-mcp) |
| Complexity estimate | Complex: safety boundary around human approval, deterministic scoring with many rule variants, cross-system device identity, LLM evaluation |
| Requirements depth | Comprehensive |

### 1.1 Context and constraints

- **Two goals.** The project is a class project (INFO 5940) and a contribution
  to the production rollout of the team's endpoint tooling. The class project
  keeps the real-engine plan (decided 2026-11-19).
- **Gate before code.** No code (console or engine) is written until the user
  confirms a check-in with a named stakeholder. Code Generation Part 2 does not
  start until then.
- **Engine work is outside this workflow (Q14 = A).** itops-mcp changes are
  tracked in the engine repository. This document states only the contract the
  console needs (section 6). The console's tests use recorded responses and
  fakes, so they do not wait on the engine.
- **Local only.** One laptop; no cloud hosting; the hosted LLM gateway is the
  only network destination outside the laptop (`tech-env.md`).
- **Stakeholder call, 2026-11-18 (recorded 2026-11-19 on Request Changes).**
  The stakeholder agreed the project is a proof of concept and demo, not a
  campus service. Their advice: start read-only and non-dangerous, then add
  human-in-the-loop approval and audit. They asked whether approval is
  enforced in the harness (the client) or at the server. The forthcoming
  agent-core MCP template is "a week or two" away. Requirements FR-REM-3 and
  FR-APR-8 answer the advice and the question.

### 1.2 Deviations from the input documents

These answers change `docs/vision.md` or `docs/tech-env.md`. The input files are
not edited; this table is the record.

| ID | Input says | Requirement says | Source |
|---|---|---|---|
| DEV-1 | Vision feature 13: model chosen by configuration, hosted gateway or Ollama; "must run on a laptop, including offline" | Hosted gateway only: Claude Opus 5.5 for the product, Claude Sonnet 5.5 for routine scenario-suite runs. No offline mode, no Ollama in the MVP; possible later addition | Q24 |
| DEV-2 | tech-env: LLM offline fallback row, Ollama base URL example, "Small local models call tools poorly" risk | Removed from MVP scope. The demo requires network access to the gateway | Q24, Q25, Q26 |
| DEV-3 | Vision OQ 6 and tech-env: local web page for everything | Terminal for chat; local web page for approvals, fleet summary and transcript viewer | Q13, Q22, Q23 |
| DEV-4 | Example policy: six checks | Seven checks: adds `patch_enrollment`; OS and third-party patching rules redefined | Q4, Q5 |
| DEV-5 | Example policy: macOS screen lock judged "enabled" only | Official result still "enabled" only; a 1800-second macOS advisory is added | Q2 |
| DEV-6 | Example policy: backup "report last completed" only | Advisory flag when the last completed backup is older than 30 days or absent | Q3 |
| DEV-7 | Vision: engine needs a CrashPlan connector | Engine also needs read-only TDX and Sassafras K2 connectors | Q7, Q8 |
| DEV-8 | tech-env dev-dependency list | Adds `hypothesis` (property-based testing) and an SBOM generator | Q31, SECURITY-10 |
| DEV-9 | tech-env: single user, no login | A per-launch access token is required by the web page (SEC-REQ-2), to meet SECURITY-08 on a shared machine | SECURITY-08 |

---

## 2. Users

| Persona | Uses | Key needs |
|---|---|---|
| Desktop support technician | Terminal chat about one device; approval page | Plain-language cause, a fix from the catalog, an approval page that shows everything |
| Endpoint engineer | Fleet summary page | Rankings of gaps and of remote fixes by devices certified |
| Compliance or team lead | Transcript viewer page | Every tool call, approval and engine audit reference for past sessions |

---

## 3. Functional requirements

### 3.1 Baseline policy (policy as data)

- **FR-POL-1.** The baseline is a YAML data file loaded with `yaml.safe_load`
  and validated into pydantic models at startup. An invalid file stops startup
  with the reason.
- **FR-POL-2.** The policy declares: device classes and which are certified;
  the seven checks (FR-RULE-*); each check's thresholds and freshness window;
  per-OS rules; which device classes each check applies to; advisories;
  unreliable-source entries; per-device class overrides (FR-CLS-2).
- **FR-POL-3.** Changing a threshold needs only a policy file change, no code
  release.

### 3.2 Device scope and class

- **FR-SCOPE-1 (Q7).** TDX (TeamDynamix asset records) decides scope. Only
  devices TDX lists as deployed and active are scored and counted.
- **FR-SCOPE-2 (Q7).** Devices present in management tools but not active in
  TDX (retired, disposed or absent) are listed separately as "clean up: remove
  from tools". They are not scored.
- **FR-CLS-1 (Q8).** Device class resolves in this order: a per-device
  override in the policy file, then Sassafras K2 "Computer Use", then the TDX
  asset record.
- **FR-CLS-2 (Q8).** The policy file can override the class per device. TDX
  reports "Unknown" for over half of Library devices, so overrides are a normal
  path, not an edge case.
- **FR-CLS-3 (Q9).** A device with no class from any source is not scored. Its
  only result is "cannot score: device class unknown".
- **FR-CLS-4 (vision feature 3).** Personal productivity devices are scored on
  every check and can be certified. Shared and special-purpose devices are
  never certified, are not scored on screen lock (`not_applicable`), and are
  scored on every other check as baseline hygiene.

### 3.3 Rule engine

General:

- **FR-RULE-0a.** The rule engine is pure: evidence, rule and current time in,
  result out. No I/O, no clock reads, no model calls (`tech-env.md`).
- **FR-RULE-0b.** Every check returns one outcome: `pass`, `fail`, `no_data`,
  `stale`, `not_applicable`, or `unverified` (FR-RULE-0d), with a reason string.
- **FR-RULE-0c.** Freshness windows are inclusive: a report exactly N days old
  is fresh; N days plus any amount is stale. Boundaries are tested.
- **FR-RULE-0d (Q6 = C).** A `fail` whose source is listed as unreliable
  (macOS screen lock; Windows 11 25H2 and later OS patching) becomes
  `unverified`, labeled "source known to be unreliable".
- **FR-RULE-0e.** Advisories are reported beside a check's result and never
  change its outcome or certification.

Checks:

- **FR-RULE-1 Encryption (Draft, Q5).** Pass when every fixed internal volume
  is encrypted with protection on: FileVault on macOS; BitLocker on the
  Windows OS volume and every fixed data volume. Fail when any fixed volume is
  unencrypted or protection is suspended. No encryption record means
  `no_data`. Freshness 30 days.
- **FR-RULE-2 Screen lock, Windows (Q1 = A).** The official limit decides:
  pass for a timeout from 1 to 1800 seconds; 0 (never) fails; above 1800
  fails. Advisory: the local 900-second limit, shown beside it. Freshness 30
  days. Not applicable to non-certified classes.
- **FR-RULE-3 Screen lock, macOS (Q2 = B, modified).** The official result
  judges "enabled" only. Advisory: a reported timeout above 1800 seconds is
  flagged. The source is unreliable (FR-RULE-0d). Freshness 30 days.
- **FR-RULE-4 Patch enrollment (Q4).** Pass when the device has a record in
  the central patch tool and checked in within 30 days. No record means
  "not enrolled", which is `fail` (not `no_data`). A record older than 30 days
  is `stale`.
- **FR-RULE-5 OS patching (Q4, Q5).**
  - Fail if a security patch for the device's own OS version has been
    available more than 30 days (counted from its publish date) without being
    installed.
  - Older Windows 11 and macOS versions pass while vendor-supported (macOS:
    the three most recent releases). An unsupported OS version fails.
  - Zero days: fail with no grace period if Tenable reports an unpatched OS
    vulnerability that is in the CISA Known Exploited Vulnerabilities catalog
    or has CVSS 9.0 or higher.
  - Windows 11 25H2 and later results are from an unreliable source
    (FR-RULE-0d). Freshness 30 days.
- **FR-RULE-6 Third-party patching (Q4, Q5).**
  - Fail when a critical or high third-party patch has been available more
    than 14 days (from its publish date) without being installed.
  - Zero days (KEV or CVSS 9.0+): fail with no grace period.
  - Inspection window: 7 days on Windows, 30 days on macOS; older is `stale`.
  - Other missing third-party patches are listed for information only.
- **FR-RULE-7 Endpoint protection (Draft, Q5).** Pass when the endpoint
  protection agent is installed and has checked in within 14 days. Installed
  but no check-in within 14 days is `stale`. No record of the agent is
  `no_data`, explained as "agent missing or broken".
- **FR-RULE-8 Backup (Q3 = B).** The official rule decides: pass when the
  backup client connected within 30 days. Advisory: flag a last completed
  backup older than 30 days, or none ever completed. The last completed backup
  per destination is always shown.

Device-level results:

- **FR-CERT-1 (Q21).** `fail`, `no_data` and `stale` all block certification.
- **FR-CERT-2 (Q6 = C).** A personal productivity device whose only
  non-passing results are `unverified` (and `pass` or `not_applicable`) is
  "certified pending verification".
- **FR-CERT-3.** A personal productivity device with every applicable check
  `pass` is "certified". Non-certified classes show "baseline hygiene: met" or
  "not met", never "certified".
- **FR-CERT-4 (Q7).** A device in scope with every applicable check `stale` or
  `no_data` is flagged "stopped reporting" by the rule engine, and the
  explanation recommends retiring or reconnecting rather than fixing checks.

### 3.4 Terminal chat and device identity

- **FR-CHAT-1 (Q13 = C).** Chat runs in the terminal as a plain line-by-line
  prompt (no terminal UI library). Tool calls and their results print as text.
- **FR-DEV-1 (Q20 = B).** A terminal command lists in-scope devices from the
  read server as a numbered list; the technician picks one to start a chat
  about it. Free text (hostname, serial number or assigned person) is also
  accepted.
- **FR-DEV-2 (Q11, Q20).** Code, not the model, resolves free text to one
  engine device ID. Several matches are shown as a numbered pick-list and never
  silently resolved. No match is reported as such.
- **FR-DEV-3 (Q11 = B).** As soon as a device is identified, code fetches the
  normalized evidence and scores it. The scoring step is printed and recorded
  in the transcript as a console-initiated step (evidence fetched, results
  produced). The results are given to the model as fixed facts.

### 3.5 Agent

- **FR-AGT-1 (vision 5, Q10 = C).** The agent investigates causes through the
  engine read server's per-system tools. It never decides pass or fail.
- **FR-AGT-2 (vision 6).** Every model turn, tool call and tool result appears
  in the terminal and the transcript.
- **FR-AGT-3 (vision 7).** For each non-passing check, the agent explains the
  likely cause and cites the tool calls behind it, covering `no_data`,
  `stale`, "stopped reporting" and "source known to be unreliable".
- **FR-AGT-4 (vision 8).** For each non-passing check, the agent recommends a
  catalog remediation, or says "no reviewed fix exists" and gives the manual
  next step.
- **FR-AGT-5.** Step and token limits per question stop the loop
  (`AGENT_MAX_STEPS`, `AGENT_MAX_TOKENS`). Reaching a limit is reported.
- **FR-AGT-6.** The agent loop is hand-written on the `openai` client. Agent
  frameworks are prohibited (`tech-env.md`).

### 3.6 Remediation and approval

- **FR-REM-1 (Q16 = A).** One remediation is demonstrable end to end (plan,
  approve, execute): reinstall the endpoint protection agent. Screen-lock
  policy and stuck update cache stay "explain only".
- **FR-REM-2 (vision 10).** Planning uses the act server's
  `plan_remediation`, which changes nothing.
- **FR-REM-3 (stakeholder call).** Non-dangerous by design: in the MVP every
  remediation, approved or not, runs only on the engine's no-op demo runner
  against the fixture fleet. Nothing in the MVP can change a real computer.
  The console shows "demo runner: no device is changed" on the approval page
  and with every execution result. Running against real devices is out of
  MVP scope and would need the stakeholder's review first.
- **FR-APR-1 (Q17 = A).** The approval page is purpose-built for the
  itops-mcp approval elicitation schema. It shows every target, the
  remediation, the script identity and the approval code in full.
- **FR-APR-2 (Q17 = A).** An elicitation that does not match that schema is
  declined automatically, with the reason shown in the terminal and
  transcript. This is the only automatic answer, and it is always "decline".
- **FR-APR-3 (Q18 = A).** No target limit: every target is always shown in
  full, with no truncation, scrolling or collapsing. Tests cover 25 targets and
  100 targets.
- **FR-APR-4 (vision 11).** Approve requires an explicit action with no
  default. Decline and cancel are always available.
- **FR-APR-5.** Decline, cancel, closing the page, timeout
  (`APPROVAL_TIMEOUT_SECS`) and a replayed answer all mean nothing executes.
- **FR-APR-6.** The terminal prints the approval page link when an approval
  opens and the outcome when it closes.
- **FR-APR-7.** The console states plainly that it cannot prove a person
  clicked (the engine records `human_verified: false`).
- **FR-APR-8 (stakeholder call).** Approval is enforced at both layers, so
  neither is the only control: (1) server: the engine's act server will not
  execute without an accepted elicitation answer and writes its audit row
  first; (2) harness: the console has no code path that answers an approval
  except a person on the approval page (and the automatic decline in
  FR-APR-2). Tests show each layer refusing on its own: a fake act server
  that executes without an answer is caught by the console's tests, and the
  console cannot answer on a person's behalf.
- **FR-EXE-1 (vision 12).** After approval, the act server executes. The
  result and the engine's audit reference are shown and recorded. Server
  refusals are shown verbatim.

### 3.7 Fleet summary

- **FR-FLT-1 (Q22 = A).** The fleet summary is its own web page, computed by
  code over all in-scope devices.
- **FR-FLT-2 (Q21 = C), per check.** For each check: the number of personal
  productivity devices whose only blocking result is this check (one fix from
  certified), and total non-passing results.
- **FR-FLT-3 (Q21 = C), per remediation.** For each catalog remediation: the
  number of devices applying it would certify, and the number it would
  improve. Checks with no remote fix are listed separately.
- **FR-FLT-4.** The page also lists "stopped reporting", "cannot score: device
  class unknown" and "clean up: remove from tools" devices.
- **FR-FLT-5 (Q22 = A).** An agent-written narrative is generated only on
  request, from the computed numbers, and labeled as model-written.
- **FR-FLT-6 (vision risks).** The fleet summary is the first feature to drop
  if time runs short.

### 3.8 Transcripts

- **FR-TRN-1.** Each chat session writes one JSON Lines file: model turns,
  console-initiated scoring steps, tool calls, results, approvals, answers and
  engine audit references.
- **FR-TRN-2 (Q23 = B).** A read-only web page lists past sessions and shows
  each transcript with its tool calls, approvals and audit references. It
  cannot edit or delete anything.
- **FR-TRN-3.** Transcript files are opened for append only; no code path
  rewrites or deletes them.

### 3.9 Model provider

- **FR-LLM-1 (Q24, DEV-1).** The console calls Claude Opus 5.5
  (`claude-opus-5-5`) through the hosted OpenAI-compatible gateway, using the
  `openai` client with `base_url` from settings.
- **FR-LLM-2.** Scenario-suite runs use Claude Sonnet 5.5 (`claude-sonnet-5-5`)
  routinely and Opus 5.5 before each milestone.
- **FR-LLM-3.** No offline mode in the MVP. If the gateway is unreachable, the
  console says so and scoring still works (it needs no model).

### 3.10 Demo, time and tokens

- **FR-DEMO-1 (vision 14).** The demo runs against a fixture fleet of about 20
  devices covering every check, both device classes plus a class-unknown
  device, a no-data device, a stopped-reporting device, an unreliable-source
  device and a TDX-inactive device.
- **FR-DEMO-2 (Q15 = A).** The engine's demo mode generates fixture timestamps
  relative to its start time, so device ages are the same on any day. The
  console always scores with the real current time.
- **FR-TOK-1 (Q19 = A).** The person runs the engine's local issuer before a
  session and puts the read and act tokens in `.env`. The console refuses to
  start without them, and reports clearly when the engine rejects an expired
  token. The console never mints or validates tokens.

### 3.11 Scenario suite

- **FR-EVAL-1 (Q27 = A).** The AI drafts ten scenarios from the fixture fleet
  (device, expected cause, expected fix or "no reviewed fix"). The user
  reviews and approves them before any run.
- **FR-EVAL-2.** `evals/run.py` runs the suite and commits results per model
  and date. It is not run in CI.

---

## 4. Non-functional requirements

### 4.1 Quality targets (from the vision's success metrics)

| ID | Requirement | Target |
|---|---|---|
| NFR-Q-1 | Rule engine correctness | Every check on every fixture device matches its expected outcome; boundary tests for every threshold and freshness window; 100% branch coverage of `baseline/rules.py` |
| NFR-Q-2 | Device class respected | Tests per class: non-certified classes never certified, never scored on screen lock, scored on everything else |
| NFR-Q-3 | Explanation accuracy | Correct cause in at least 8 of 10 scenarios |
| NFR-Q-4 | Recommendation accuracy | Correct catalog entry or correct "no reviewed fix" in at least 8 of 10 scenarios |
| NFR-Q-5 | Approval page completeness | 100% of targets, remediation name and script identity visible for 25- and 100-target plans |
| NFR-Q-6 | No action without a person | 0 executions on decline, cancel, close, timeout, replay and schema mismatch; a control test with the approval check removed must fail |
| NFR-Q-7 | Transparency | Transcript matches the server audit log for every tool call |
| NFR-Q-8 | Setup time (Q26) | Clean clone to working demo in under 15 minutes on a second laptop, following the README (gateway path) |
| NFR-Q-9 | Line coverage | 85% overall, excluding `evals/` |

### 4.2 Usability and accessibility (Q28 = A)

- **NFR-A-1.** Web pages use semantic HTML (headings, lists, tables with
  header cells, labeled form controls).
- **NFR-A-2.** The approval page is fully operable by keyboard, with a visible
  focus indicator.
- **NFR-A-3.** Text and controls meet sufficient color contrast; meaning is
  never conveyed by color alone (outcomes are written as words).
- **NFR-A-4.** No formal conformance target.

### 4.3 Performance and limits

- **NFR-P-1.** One async event loop serves the web pages, agent loop and MCP
  clients; an open approval blocks nothing else.
- **NFR-P-2.** The fleet summary over the fixture fleet renders in under 5
  seconds on the demo laptop.

### 4.4 Maintainability and portability

- **NFR-M-1.** Python 3.12, uv with a hashed lock file, the libraries and
  prohibitions in `tech-env.md`.
- **NFR-M-2.** A later port of the web pages to TypeScript and Angular (the AI
  hub's preference) is possible but out of MVP scope (Q13).

### 4.5 Security requirements (Security Baseline extension enabled, Q29 = A)

`tech-env.md` already sets most controls. The items below make each applicable
SECURITY rule explicit. Some rules are written for cloud deployments; where a
rule is adapted to a single local laptop, the adaptation is stated.

- **SEC-REQ-1 (SECURITY-07).** The web server binds to `127.0.0.1` only.
  Outbound traffic goes only to the engine on localhost and to the gateway.
- **SEC-REQ-2 (SECURITY-08, SECURITY-12, new).** At each launch the console
  creates a random access token and prints the web page link carrying it in
  the terminal. The first request exchanges it for a session cookie
  (`HttpOnly`, `SameSite=Strict`, and `Secure` if the target browsers accept it
  on `127.0.0.1`, to verify in Construction). Every route except a health
  route requires that session; other local users and processes cannot open
  approvals. The session ends when the console stops.
- **SEC-REQ-3 (SECURITY-08).** Every POST checks the `Origin` header and a
  CSRF token. No CORS headers are sent (same-origin only).
- **SEC-REQ-4 (SECURITY-08).** An approval answer is accepted only for an
  approval that is currently open, once.
- **SEC-REQ-5 (SECURITY-04).** Every HTML response sets
  `Content-Security-Policy` (at least `default-src 'self'`, no `unsafe-inline`
  or `unsafe-eval`), `X-Content-Type-Options: nosniff`,
  `X-Frame-Options: DENY` and `Referrer-Policy: strict-origin-when-cross-origin`.
  `Strict-Transport-Security` is not sent: the server is plain HTTP on
  loopback, where browsers ignore HSTS (documented exception).
- **SEC-REQ-6 (SECURITY-05).** Every route, form, setting, policy file and
  tool argument is validated by a pydantic model with explicit string length
  limits; request body size is capped. Jinja2 autoescape is on; tool output and
  model text render as text.
- **SEC-REQ-7 (SECURITY-09).** FastAPI's `/docs`, `/redoc` and
  `/openapi.json` are disabled. Error pages are generic (no stack traces or
  paths). No default credentials. No directory listing for static files.
- **SEC-REQ-8 (SECURITY-10).** `uv.lock` with hashes, `uv sync --locked`,
  `pip-audit` in CI, GitHub Actions pinned to commit SHAs, no unused
  dependencies, an SBOM (CycloneDX) generated in CI.
- **SEC-REQ-9 (SECURITY-13).** JSON and `yaml.safe_load` only; no `pickle`.
  The vendored HTMX file is checked against its recorded sha256 at startup and
  in CI. No CDN resources.
- **SEC-REQ-10 (SECURITY-03, adapted).** Structured JSON logs (timestamp,
  session or request ID, level, message) to one local log directory, the
  single central store for this local product (telemetry services are
  prohibited). Secrets, tokens and the access token never appear in logs,
  pages, prompts or transcripts.
- **SEC-REQ-11 (SECURITY-14, adapted).** Security events (bad or missing
  session, origin or CSRF refusal, replayed or unknown approval answer, schema
  mismatch decline) are logged at WARNING and printed in the terminal as an
  alert. Logs and transcripts are append-only from the application; local logs
  are kept at least 90 days. The engine's audit log is the authoritative
  tamper-evident record of actions.
- **SEC-REQ-12 (SECURITY-15).** All external calls (MCP, gateway, file I/O)
  have explicit error handling and fail closed: any error during approval or
  execution means nothing runs. A global error handler logs and returns a
  generic page.
- **SEC-REQ-13 (SECURITY-01, adapted).** Transcripts and logs hold fixture
  data only and stay on the laptop; the README requires the demo laptop to
  have full-disk encryption (FileVault or BitLocker). The gateway connection
  uses TLS. Localhost engine traffic is plain HTTP on loopback (documented
  exception).
- **SEC-REQ-14 (SECURITY-11).** Security-critical logic lives in dedicated
  modules: approvals (`mcp/approvals.py`), request security (`security.py`),
  scoring (`baseline/`). Design documents cover at least these misuse cases:
  prompt injection through device data, a malicious local web page posting to
  the console, a replayed approval, an elicitation with an unexpected schema.
- **SEC-REQ-15 (OWASP LLM Top 10).** The control mapping in `tech-env.md`
  applies unchanged, except LLM10 no longer mentions Ollama.

### 4.6 Testing requirements (Property-Based Testing extension, Partial, Q31 = B)

- **TST-PBT-1 (PBT-09).** Hypothesis is the PBT framework, added as a
  development dependency (DEV-8).
- **TST-PBT-2 (PBT-03).** Invariant properties for the rule engine and fleet
  summary, for example: an outcome is always one of the allowed values; a
  non-certified class is never certified; screen lock is always
  `not_applicable` for non-certified classes; any `fail`, `no_data` or
  `stale` blocks certification; moving a report date further into the past
  never turns `stale` into `pass`; fleet counts never exceed the number of
  in-scope devices.
- **TST-PBT-3 (PBT-02).** Round-trip properties for transcript records
  (write then read gives the same record) and policy models (load then dump
  then load gives the same model).
- **TST-PBT-4 (PBT-07, PBT-08).** Domain generators for devices, evidence and
  policies live in one shared test module; Hypothesis shrinking stays on; the
  seed is printed on failure and CI uses a fixed seed profile.
- **TST-PBT-5.** Example-based tests still pin every boundary in NFR-Q-1. PBT
  rules other than PBT-02, 03, 07, 08 and 09 are advisory (Partial mode).

---

## 5. Out of scope for the MVP

Everything in the vision's "features OUT" table, plus:

- Offline mode and any local model (DEV-1).
- Remote fixes other than reinstalling the endpoint protection agent (Q16).
- Any remediation run against a real device; the MVP uses only the engine's
  no-op demo runner (FR-REM-3).
- Sassafras "Computer Current Use" as a class source (superseded by K2
  "Computer Use", Q8).
- A generic renderer for other elicitation schemas (Q17).
- A TypeScript or Angular web front end (Q13).

---

## 6. Engine contract (work tracked in the itops-mcp repository, Q14 = A)

The console depends only on this contract and on recorded responses of it.

| Item | Needed by |
|---|---|
| Read tool: list in-scope devices (TDX active), with engine device ID, hostname, serial, assigned person, OS | FR-DEV-1, FR-SCOPE-1 |
| Read tool: resolve free text to candidate device IDs | FR-DEV-2 |
| Read tool: normalized per-check evidence for one device (all seven checks, report timestamps, device class from K2 and TDX, TDX status) | FR-DEV-3, FR-RULE-* |
| Evidence for the whole fleet (a batch tool, or the per-device tool for about 20 devices) | FR-FLT-* |
| List of devices present in tools but not active in TDX | FR-SCOPE-2 |
| Per-system read tools for investigation (device management, Tenable, patch tool, endpoint protection, CrashPlan, TDX, K2) | FR-AGT-1 |
| Read-only connectors: CrashPlan (last connected, last completed per destination), TDX assets, Sassafras K2 "Computer Use" | FR-RULE-8, FR-SCOPE-1, FR-CLS-1 |
| Act tools `plan_remediation` and `execute_remediation`; catalog entry for reinstalling the endpoint protection agent | FR-REM-*, FR-EXE-1 |
| Approval through MCP form elicitation (revision 2026-07-28), with a stable schema listing every target, remediation, script identity and approval code | FR-APR-* |
| Demo mode: fixture fleet (FR-DEMO-1), timestamps relative to start (FR-DEMO-2), no-op runner, local token issuer | FR-DEMO-*, FR-TOK-1 |

---

## 7. Risks carried forward

| Risk | Mitigation |
|---|---|
| Engine connectors and evidence tool may have to follow the forthcoming agent-core MCP server template, timing unknown | Console depends only on the contract (section 6) and recorded responses |
| Tool calling through the gateway is untested | Verify early in Construction with a minimal tool call |
| fastmcp 4 client support for elicitation revision 2026-07-28 is unverified | Verify early in Construction against the engine's approval tests |
| Demo needs network access to the gateway (DEV-1) | Scoring, approval and fleet numbers work without the model; README states the requirement |
| Stakeholder advice to start read-only and non-dangerous | FR-REM-3: every remediation runs on the no-op demo runner; real devices out of MVP scope |
| Drafted encryption and endpoint protection rules (FR-RULE-1, FR-RULE-7) may differ from the official definitions | User review of this document; check against official results before organizational use |
| Solo developer, about three weeks | Fleet summary drops first (FR-FLT-6) |

---

## 8. Items for the user's review

1. FR-RULE-1 and FR-RULE-7 are drafts (Q5).
2. SEC-REQ-2 (per-launch access token for the web page) is new. It comes from
   the Security Baseline, not from an answer.
3. FR-DEV-1 puts the device list in the terminal, because chat lives there
   (Q13). The fleet summary page shows device IDs but does not start chats.
4. FR-APR-2: a schema mismatch is declined automatically. That is the only
   automatic answer.
5. SEC-REQ-5, 10, 11 and 13 adapt cloud-oriented SECURITY rules to a local
   laptop, as stated in each item.
