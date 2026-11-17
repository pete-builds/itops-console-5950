# Requirements Verification Questions

**Project**: itops-console
**Stage**: INCEPTION - Requirements Analysis (comprehensive depth)
**Created**: 2026-11-17T19:05:00Z
**Inputs**: `docs/vision.md`, `docs/tech-env.md`

These questions cover the eight open questions in `docs/vision.md` (marked
"Vision OQ n"), gaps I found while reading both input documents, and the three
AI-DLC extension opt-ins at the end.

**How to answer**: write the letter after each `[Answer]:` tag. If no option
fits, choose the last option (Other) and describe what you want after the tag.
You may add a short comment after any letter. Where a question allows more
than one letter, it says so. A "Note" under a question says where the input
documents already lean; it is context, not a decision made for you.

---

## Question 1: Windows screen-lock threshold (Vision OQ 1)
The official rule passes a Windows timeout from 1 to 1800 seconds (0, meaning
never, fails). The local unit enforces a stricter 900 seconds. Which limit
decides the Windows screen-lock result?

A) The official 1800-second limit decides pass, fail and certification; the local 900-second result is shown beside it as an advisory with no effect on certification

B) The local 900-second limit decides pass, fail and certification; the official result is shown beside it (the console will sometimes fail a device the official dashboard passes)

C) The official 1800-second limit only; no local rule is shown

D) Other (please describe after [Answer]: tag below)

Note: the example policy in `tech-env.md` (`local_max_timeout_secs: 900`,
"reported beside it") implies A.

[Answer]: A

## Question 2: macOS screen-lock timeout (Vision OQ 1)
The official macOS rule judges only whether a screen lock is enabled, and the
publisher marks macOS screen-lock data as inaccurate. What should the console
do with a macOS timeout when one is reported?

A) Judge "enabled" only, as the official rule does; display the reported timeout as information, with no pass or fail of its own

B) Judge "enabled" for the official result, and also apply the local 900-second rule to the macOS timeout as an advisory beside it

C) Judge "enabled" only and do not display the timeout

D) Other (please describe after [Answer]: tag below)

[Answer]: B, modified: the macOS advisory timeout limit is 30 minutes (1800 seconds), not the local 900 seconds. User's words: "b use 30 minutes for the timeoute"

## Question 3: Backup rule (Vision OQ 3)
The official check passes when the backup client has connected within 30
days. The engine's planned CrashPlan connector can also return the last
completed backup per destination, which can be empty.

A) The official rule decides; the last completed backup per destination is always shown beside it, with no pass or fail of its own

B) The official rule decides; a local advisory also flags "no completed backup within N days" or "no completed backup at all" (give N after the letter)

C) A stricter local rule decides: fail unless a backup completed within N days (give N after the letter); the official result is shown beside it

D) Other (please describe after [Answer]: tag below)

Note: the example policy (`report_last_completed: true`, "shown beside it,
never hidden") implies A.

[Answer]: B, N = 30 days. The official rule (connected within 30 days) decides; an advisory flags any device whose last completed backup is older than 30 days, or that has never completed one.

## Question 4: "Enrolled in central patch management"
The vision lists enrollment in central patch management as part of the
baseline, but the example policy has six checks (encryption, screen_lock,
os_patching, third_party_patching, endpoint_protection, backup) and no
enrollment check. How should enrollment be treated?

A) Not a separate check: a device with no record in the patch tool scores no data on third_party_patching (and os_patching where the same tool supplies it), and the agent explains "not enrolled" as the likely cause

B) A separate, seventh scored check with its own freshness window in the policy file

C) Other (please describe after [Answer]: tag below)

[Answer]: B (separate enrollment check), with OS patching rules added. User's words: "lets just do a check for enrollment in central patching, no zero days, and patched up to n - 40 days of latest OS patch . it's ok to be on an older supported version of windows 11 or macos as long as it' enrolled and gettin monthly patches". Confirmed interpretation (2026-11-17, user: "yes, both os and third party, these rules decide"):
1. Patch enrollment is a new, separate check: pass if the device has a record in the central patch tool and checked in within 30 days; no record means "not enrolled" and fails (not "no data").
2. OS patching: fail if a security patch for the device's own OS version has been available for more than 30 days (counted from the patch publish date) without being installed. Revised from 40 days on 2026-11-17 to match the endpoint policy ("apply all security patches within 30 days of publish").
3. Older OS versions (Windows 11, macOS) pass while vendor-supported; an unsupported OS version fails.
4. No zero days: fail if Tenable reports any unpatched vulnerability that is known to be actively exploited (for example, listed in the CISA Known Exploited Vulnerabilities catalog) or has a CVSS score of 9.0 or higher, with no grace period. Applies to both the OS and third-party applications. (CVSS 9.0+ added 2026-11-17.)
5. These rules decide pass, fail and certification (they are not advisories), even where they differ from the official dashboard.

## Question 5: Source of the detailed check definitions
`tech-env.md` says rule values follow a real university's published
compliance definitions, but neither input defines, for example, what
"current" means for OS patching (which update counts as latest, how "n-2
macOS" is determined) or what a failing third-party patch inspection is.
Where should the detailed pass and fail definitions come from?

A) I will provide the published definitions (text or a public link) before requirements.md is written, and requirements will cite them

B) Draft them from the example policy and common practice during Functional Design, for my review

C) Draft them now in requirements.md, for my review

D) Other (please describe after [Answer]: tag below)

[Answer]: Question narrowed during the interactive session to third-party patching beyond zero days. Answer: C (severity-based) with N = 14 days. A third-party critical or high patch fails once it has been available more than 14 days (counted from the patch publish date) without being installed; zero days (KEV or CVSS 9.0+) fail with no grace period; the inspection window applies (7 days Windows, 30 days macOS); other missing third-party patches are listed for information only. Encryption and endpoint protection definitions to be drafted in requirements.md for review. Source: the university's Policy 5.10 summary (endpoints: all security patches within 30 days of publish; macOS within the three most recent releases) and its hardening requirements (critical and high within 14 days).

## Question 6: Unreliable-source results and certification
When a check fails but its source is marked unreliable (macOS screen lock;
Windows 11 25H2 and later OS patching), how should that result count?

A) It counts as a failure for certification and for the fleet ranking, exactly as the official dashboard counts it, and is labeled "source known to be unreliable" wherever it appears

B) It is labeled, counts as a failure on the device view, but is left out of the fleet summary's "fixes that certify the most devices" ranking

C) It is labeled and treated as "unverified" rather than failed; the device is shown as "certified pending verification"

D) Other (please describe after [Answer]: tag below)

[Answer]: C

## Question 7: Deciding that a device has "stopped reporting"
The vision says a device failing every check has usually stopped reporting
and needs retiring, not fixing. How should the console decide that?

A) Deterministically in the rule engine: every applicable check is stale or no data

B) Deterministically: the device's last check-in to its device management system is older than a threshold set in the policy file (give the number of days after the letter)

C) Both A and B; either one sets the flag

D) Not a deterministic flag; the agent infers it from the evidence and says so in its explanation

E) Other (please describe after [Answer]: tag below)

[Answer]: E. User's words: "tdx should have a list of deployed active devices." Clarified (user: "1a . 2a. 3a."):
1. TDX (TeamDynamix asset records) decides scope: only devices TDX lists as deployed and active are scored and counted. Devices present in the tools but not active in TDX (retired, disposed, or absent from TDX) are listed separately as "clean up: remove from tools".
2. A device TDX lists as active is flagged "stopped reporting" by the rule engine when every applicable check is stale or no data.
3. A read-only TDX asset connector is added to the engine (alongside the CrashPlan connector); the demo uses fixture TDX records.

## Question 8: Source of the device class (Vision OQ 2)
Certification applies only to personal productivity devices. Where does a
device's class come from?

A) A field in the device management record (for example a Jamf extension attribute or an Intune device category), returned by the read server

B) The inventory or asset record, returned by the read server

C) The policy file: an explicit device-to-class mapping or naming-pattern rules

D) A device management field (as in A), which the policy file can override per device

E) Other (please describe after [Answer]: tag below)

[Answer]: D, with TDX as the default source (options were re-presented in the interactive session with TDX in place of the device management field). User's words: "D. TDX is the data of record for personal productivity device status, so use it by default. It is Unknown on over half of our Library devices, so the policy file must be able to override it per device. Longer term, Sassafras "Computer Current Use" is the better source, because it separates kiosk, loaner and shared devices. It is still being populated, so leave it out of the first version."
Recorded: device class comes from the TDX asset record by default; the policy file can override it per device; TDX reports "Unknown" for over half of Library devices, so the Unknown case (Question 9) is common, not an edge case; Sassafras "Computer Current Use" is a future source, out of scope for the MVP.
REVISED 2026-11-17 (user: "we are tagging ppd devices in itham k2 "computer use field"", then "1b"): Sassafras K2's "Computer Use" field IS an MVP source, because personal productivity devices are being tagged there. Order: K2 "Computer Use" first, then TDX, and a per-device override in the policy file wins over both. The engine needs a read-only K2 connector (fixture K2 records in the demo).

## Question 9: Missing device class (Vision OQ 2)
What should the console do when a device's class is missing?

A) Treat it as personal productivity: score every check including screen lock, allow certification, and flag "device class missing"

B) Score every check including screen lock, but never certify; show "device class missing: certification not determined"

C) Do not score; report "cannot score: device class missing" as the only result

D) Other (please describe after [Answer]: tag below)

[Answer]: C. A device with no class from K2, TDX or a policy-file override is not scored; the console reports "cannot score: device class unknown" as its only result. (User: "2c".)

## Question 10: How system data becomes rule-engine evidence
The rule engine takes typed evidence. The vision's risk table says the engine
has no read tool yet that returns baseline evidence. If the model extracted
evidence from raw tool output, pass or fail would depend on the model. Which
path should evidence take?

A) The engine adds a read tool that returns normalized per-check evidence for a device; console code (not the model) passes it to the rule engine; the model receives the results as facts

B) The console calls the engine's existing per-system read tools and deterministic console adapters map each tool's output to evidence; the model receives the results as facts

C) Both: a normalized evidence tool (as in A) is used for scoring, and the per-system read tools are available to the agent for investigating causes

D) Other (please describe after [Answer]: tag below)

Note: A, B and C all keep the model out of scoring. C matches the vision's
description of the agent investigating across systems.

[Answer]: C

## Question 11: When scoring runs during a chat
A) The console offers the model a local tool (for example `score_device`); the tool fetches evidence and runs the rule engine in code, and the call and its results are recorded in the transcript like any other tool call

B) The console scores the device in code as soon as the device is identified, before the agent investigates, and gives the results to the model as fixed facts at the start of its turn

C) Other (please describe after [Answer]: tag below)

[Answer]: B, with an addition, accepted on the AI's recommendation (user asked "what do ou think?", then "ok"): the console scores the device in code as soon as it is identified, and records the scoring step in the transcript as a console-initiated step (evidence fetched, results produced), so it is as visible as a tool call. Device identity for scoring is resolved by code to an engine device ID, not guessed by the model; ambiguous matches are put to the technician (see Question 20).

## Question 12: Agent runtime (Vision OQ 5)
A) In-process on the laptop: one Python async process runs the web app, agent loop, rule engine and MCP clients

B) A hosted managed-agent service the developer already operates

C) Other (please describe after [Answer]: tag below)

Note: `tech-env.md` assumes A and forbids cloud hosting. B would also need
that service to reach MCP servers on the laptop and to pass form elicitation
through to a person, both untested.

[Answer]: 

## Question 13: Interface (Vision OQ 6)
A) A local web page: FastAPI, Jinja2 and HTMX on `127.0.0.1:8080`

B) A terminal interface

C) Both: a terminal for chat and a local web page for approvals

D) Other (please describe after [Answer]: tag below)

Note: `tech-env.md` already specifies A, and its library list has no terminal
UI library.

[Answer]: 

## Question 14: Engine work and this workflow
The console needs engine changes that land in the itops-mcp repository: a
CrashPlan read connector, baseline evidence for the fixture fleet (and the
normalized evidence tool, if Question 10 is A or C), and a demo mode with a
no-op runner. How should this AI-DLC workflow treat them?

A) Track the engine work outside this workflow, in the engine repository; these requirements only state the contract the console needs (tools, fields, demo behavior), and the console's tests use recorded responses

B) Include the engine work as its own unit of work in this workflow (designed and planned here; code committed in the engine repository)

C) Avoid engine changes for the MVP: the console ships its own fake read and act servers for the demo

D) Other (please describe after [Answer]: tag below)

Note: C conflicts with vision features 5, 10, 11 and 12, which go through the
real engine.

[Answer]: 

## Question 15: Time in the demo
Every check has a freshness window. If fixture report dates are fixed, the
demo drifts: a month later every device is stale. How should demo time work?

A) The engine's demo mode generates fixture timestamps relative to the time it starts, so every device's age is the same on any day

B) Fixture dates are fixed; the console has an "as of" setting used as the current time for scoring in demo mode, shown on every page

C) Other (please describe after [Answer]: tag below)

Note: with B, the agent still sees the raw fixture dates and the real date,
so its explanations may disagree with the scores unless it is also told the
"as of" time.

[Answer]: 

## Question 16: Remote fixes in the MVP (Vision OQ 4)
Which remediations should be demonstrable end to end (plan, approve,
execute)? All others stay "explain only". **More than one letter allowed.**

A) Reinstall the endpoint protection agent

B) Apply the screen-lock policy (assign the configuration profile or policy to the device)

C) Clear a stuck update cache

D) Other (please describe after [Answer]: tag below)

If any remediation you choose is not yet in the engine's catalog, name it
after the letters.

[Answer]: 

## Question 17: Rendering the approval form
The act server sends an MCP form elicitation: a message plus a JSON schema
for the answer. How should the console render it?

A) A purpose-built screen for the itops-mcp approval schema (every target, the remediation, the script identity and the approval code laid out in full); any elicitation that does not match that schema is declined and the person is told why

B) A generic renderer that draws any form elicitation schema

C) The purpose-built screen for the approval schema, and a generic renderer for any other schema

D) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 18: Plans with more than 25 targets
The success metric covers plans up to 25 targets. What happens above 25?

A) No limit in the console: every target is always shown in full; tests cover 25 targets and a larger plan (for example 100)

B) The console declines any plan with more than 25 targets and tells the person to narrow the scope

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 19: Engine tokens in the demo (Vision OQ 7)
The engine refuses to execute without a bearer token. `tech-env.md` says
tokens come from the engine's local test issuer, are short-lived, and the
console never mints or validates them.

A) The person runs the issuer before each session and puts the tokens in `.env`; the console refuses to start without them and shows a clear message when a token has expired

B) The console requests tokens from the local issuer at startup and again when they expire (it still never mints or validates them)

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 20: Picking a device
A) Free text in the chat only (hostname, serial number, or assigned person); the agent resolves it across systems

B) A device list from the read server that the technician can pick from to start a chat about that device, with free text also accepted

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 21: Fleet summary ranking (MVP feature 9)
"The fixes that would bring the most devices into full compliance." Which
computation should the fleet summary use?

A) Per check: the number of personal productivity devices whose only non-passing check is this one (one fix from certified), plus total failures per check

B) Per catalog remediation: the number of devices that applying it would certify, plus the number it would improve; checks with no remote fix are listed separately

C) Both A and B

D) Other (please describe after [Answer]: tag below)

Assumption to confirm or correct in your answer: "no data" and "stale" block
certification exactly as "fail" does.

[Answer]: 

## Question 22: Where the fleet summary appears
A) On its own page, computed by code, with an agent-written narrative on request

B) Only in the chat, when the technician asks

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 23: Reviewing past transcripts (compliance or team lead)
A) JSON Lines files only, one per chat session; no viewer in the MVP

B) JSON Lines files plus a read-only page that lists past sessions and shows each transcript with its tool calls, approvals and engine audit references

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 24: Offline model (Vision OQ 8)
A) Choose by measurement: in Construction, run the scenario suite against two or three tool-calling Ollama models that fit the laptop, pick the best, and commit the results

B) I will name the model (write it after the letter)

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 25: Memory of the demo laptops
This limits which Ollama models can run. If the two laptops differ, give the
smaller one.

A) 16 GB

B) 24 to 36 GB

C) 48 GB or more

D) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 26: Scope of the 15-minute setup target
The vision targets a clean clone to a working demo in under 15 minutes. Does
that include the offline path?

A) No: it covers the hosted-gateway path only; offline setup (installing Ollama and pulling the model) is documented and timed separately

B) Yes: it includes installing Ollama and downloading the offline model

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 27: Writing the ten evaluation scenarios
A) The AI drafts the ten scenarios (device, expected cause, expected fix or "no reviewed fix") from the fixture fleet, and you review and approve them before any run

B) You write them

C) Other (please describe after [Answer]: tag below)

[Answer]: 

## Question 28: Accessibility
A) Semantic HTML, a fully keyboard-operable approval screen, and sufficient color contrast; no formal conformance target

B) WCAG 2.2 AA conformance as a tested requirement

C) No specific requirement

D) Other (please describe after [Answer]: tag below)

[Answer]: 

---

## Question 29: Security Extensions
Should security extension rules be enforced for this project?

A) Yes — enforce all SECURITY rules as blocking constraints (recommended for production-grade applications)

B) No — skip all SECURITY rules (suitable for PoCs, prototypes, and experimental projects)

X) Other (please describe after [Answer]: tag below)

Note: `tech-env.md` already sets many security controls (origin and CSRF
checks, hash-locked dependencies, secret scanning, OWASP LLM Top 10 mapping).
This extension would add its rules as blocking checks at every stage.

[Answer]: 

## Question 30: Resiliency Extensions
Should the resiliency baseline be applied to this project?

**What this extension is.** Enabling it applies a set of **directional, design-time best practices** for building resilient systems, derived from the **AWS Well-Architected Framework (Reliability Pillar)** and resilience-review guidance. It steers requirements, design, and code toward fault tolerance, high availability, observability, and recoverability — covering 15 practice areas across business goals, change management, observability, high availability, disaster recovery, and continuous improvement.

**What this extension is NOT.** Enabling it does **not** make your workload production-ready, nor does it certify or guarantee any availability, RTO, or RPO target. It is a **starting point** that scaffolds good resiliency decisions early — it is not a substitute for a formal **AWS Well-Architected Review** of the built system.

Treat the output as a well-grounded **first draft of your resiliency posture** to build on and validate — not a finished, production-certified result.

A) Yes — apply the resiliency baseline as directional best practices and design-time guidance (recommended for business-critical workloads, as an informed starting point that you can validate and harden before go-live)

B) No — skip the resiliency baseline (suitable for PoCs, prototypes, and experimental projects where rapid iteration matters more than reliability)

X) Other (please describe after [Answer]: tag below)

Note: the product is a single-user demo on one laptop with no cloud
deployment.

[Answer]: 

## Question 31: Property-Based Testing Extension
Should property-based testing (PBT) rules be enforced for this project?

A) Yes — enforce all PBT rules as blocking constraints (recommended for projects with business logic, data transformations, serialization, or stateful components)

B) Partial — enforce PBT rules only for pure functions and serialization round-trips (suitable for projects with limited algorithmic complexity)

C) No — skip all PBT rules (suitable for simple CRUD applications, UI-only projects, or thin integration layers with no significant business logic)

X) Other (please describe after [Answer]: tag below)

Note: the pure rule engine (`baseline/rules.py`) and policy loader are good
PBT candidates. Enabling PBT would add `hypothesis` as a development
dependency; it is not in the `tech-env.md` list today.

[Answer]: 
