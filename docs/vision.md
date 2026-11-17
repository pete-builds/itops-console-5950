# Vision: itops-console

*Working name. Greenfield product. Runs locally.*

## Executive summary

itops-console is a local AI assistant that helps IT support staff bring
Windows and macOS computers up to their organization's endpoint security
baseline: endpoint protection installed and reporting, enrolled in central
patch management, operating system and third-party patches current, screen
lock within the required timeout, disk encrypted, and backed up. A technician
asks "is this laptop compliant, and if not, why?" An LLM agent gathers the
evidence by calling read-only tools across the organization's device
management and vulnerability systems, a deterministic rule engine decides
pass or fail for each check, and the agent explains the result and recommends
a fix from a catalog of reviewed remediations. Nothing changes on a computer
until the technician approves the exact action on a screen that shows every
target and exactly what will run. The console is the missing human half of
itops-mcp, an existing governed MCP engine whose approval step no tested MCP
client can currently carry to a person.

## Problem statement

Many organizations require every managed computer to meet a security
baseline, and they score it per device. Compliance is reported by a dashboard
that says a check failed, but not why, and the evidence for each check lives
in a different system: endpoint protection status in one console, encryption
and screen lock in the device management system (Jamf Pro for Macs, Intune or
MECM for Windows), patch status in a patch tool, vulnerabilities in Tenable,
backup in a backup console. Fixing one failing device means opening four or
five consoles, finding the same device in each, and working out which of
several possible causes applies. A device only counts as compliant when its
last failing check is fixed, so the order of work matters, and that is hard
to see from a per-check dashboard.

Four things make this harder than it looks. "No data" is not the same as
"failed": a device with no record from a tool usually has a missing or broken
agent, and the fix is different. Every check has a freshness window (30 days
of inventory for most, 14 days of check-in for endpoint protection, 7 days for
third-party patch inspection), so a healthy device that has not reported
fails anyway, and a device failing every check has usually stopped reporting
altogether and needs retiring, not fixing. The source data is not equally
reliable: the publisher of one real baseline marks its macOS screen-lock data
and its newest Windows patch data as inaccurate, so a failure there may not
be real. And the baseline is not uniform: rules differ by operating system,
and devices assigned to one person (personal productivity devices) are the
only ones certified, while shared and special-purpose devices are not
certified but must still meet every check except screen lock.

The obvious shortcut, letting an AI agent act on devices directly, is unsafe.
An agent that reads device data or vendor output can be steered by a prompt
injection, and an agent that can both read untrusted text and change a
computer can be turned against the fleet. itops-mcp, the engine this product
builds on, solves the server side: reading and acting are separate servers,
every action is a reviewed script chosen from a catalog, scope comes from
policy, and an audit row is written before any action runs. Its act server
asks a person to approve each action through MCP form elicitation (protocol
revision 2026-07-28).

The gap is the client. In tests run on 2026-10-02, no available client could
carry that approval to a person: one general-purpose coding agent shows the
approval form truncated after two lines, hiding the targets and the approval
code; another negotiates an older protocol revision without the feature; and
the LLM gateway in use drops the approval fields in transit. So the engine can
plan an action but no person can safely approve it. itops-console closes that
gap with an agent built for this one job.

## Why an LLM is the right tool, and where it is not

- **Gathering evidence across systems is open-ended.** Deciding which tools to
  call, for which device, in which systems, and reconciling a device that
  appears under different names, is reasoning a fixed script handles badly.
- **Explaining a failure is the valuable part.** The dashboard already says
  "failed". The technician needs to know why (agent not reporting, profile not
  scoped, policy linked to the wrong group, device offline for weeks) and what
  to do next. The LLM writes that explanation from the evidence.
- **Matching a failure to a fix is judgment.** The agent decides which catalog
  remediation, if any, addresses the cause, and says plainly when none does.
- **Pass or fail is NOT the LLM's call.** Whether a 20-minute screen lock meets
  a 15-minute rule is arithmetic. A rule engine, configured from a policy file,
  decides every check, so the result is the same every time and testable.
- **Also never the LLM's call:** writing code that runs on a computer,
  choosing which machines are in scope, or approving anything. Those stay with
  the catalog, the policy, and the person.

## Target users

| User | What they do with the console | What they need |
|---|---|---|
| Desktop support technician | Checks one device against the baseline, reads why it fails, approves or declines a recommended fix | Plain-language reasons, a fix they can trust, an approval screen they can read in full |
| Endpoint engineer | Looks across the fleet for the fixes that would bring the most devices into full compliance | A ranked list of fixes by devices certified, and correlation across systems without switching consoles |
| Compliance or team lead | Reviews which devices fail which checks, and what was recommended and approved | A per-check summary, a transcript of every tool call, and the audit reference for every action |

## Success metrics

| Metric | Target | How it is measured |
|---|---|---|
| Rule engine is correct | Every check on every fixture device matches the expected pass, fail, no-data, or stale result | Unit tests over the fixture fleet, including boundary values (timeout exactly at the limit and one second over, a report exactly at the freshness window and one day past it, a timeout of 0) |
| Device class respected | Shared and special-purpose devices are never reported as certified, are never failed on screen lock, and are failed on every other check like any device | Unit tests with devices of each class |
| Explanation accuracy | The agent names the correct cause in at least 8 of 10 scripted scenarios | Scenario suite against fixture data, results recorded per model |
| Recommendation accuracy | Correct catalog entry, or a correct "no reviewed fix exists", in at least 8 of 10 scenarios | Same suite |
| Approval screen shows the whole plan | 100% of targets, the remediation name, and its script identity visible with no truncation, for plans up to 25 targets | Automated UI test with a 25-target plan |
| No action without a person | 0 executions when the person declines, cancels, or closes the screen | Automated tests for each path, with a control proving the test fails if approval is bypassed |
| Transparency | Every tool call the agent makes appears in the transcript with its result | Test comparing the transcript to the server audit log |
| Runs on a laptop | A new machine goes from clean clone to working demo in under 15 minutes by following the README | Timed run on a second laptop |
| No secrets in the repository | Zero secrets in full history | gitleaks scan with a planted-secret control |

## Full scope vision

1. **Baseline checks:** every check the organization scores, for every device,
   from live data in every connected system.
2. **Fleet view:** compliance by check and by device group, and the fixes that
   would bring the most devices into full compliance, with a trend over time.
3. **Explanation:** for each failing check, the likely cause and the evidence.
4. **Remediation:** catalog-bound fixes for each check that can be fixed
   remotely, with approval, execution, and a re-check afterwards.
5. **Records:** transcripts linked to the engine's audit rows, exportable.
6. **Organization deployment:** single sign-on, per-person roles, hosted for an
   IT department, with the baseline policy kept in the organization's private
   deployment.
7. **Ticketing:** open a ticket for a device that needs hands-on work.

## MVP scope: features IN

| # | Feature | Rationale |
|---|---|---|
| 1 | Baseline policy as a data file: the checks, their thresholds (such as the screen-lock timeout), each check's freshness window, per-OS rules, which device classes each check applies to, and known data-quality caveats per source | The organization's rules are configuration, not code, so they can change without a release |
| 2 | Rule engine that scores a device on each check as pass, fail, no data, or stale (no report inside the freshness window), and marks a result whose source is known to be unreliable | Deterministic and testable; keeps pass or fail away from the model |
| 3 | Device class handling: personal productivity devices are scored for certification on every check; other devices are not certified, but are still scored on every check except screen lock and reported as baseline hygiene | Shared and special-purpose devices still need patching, encryption, endpoint protection and backup, but a screen lock would get in the way of how they are used |
| 4 | Chat interface where a technician asks about a device | The core interaction |
| 5 | Agent loop that gathers evidence through the itops-mcp read server's tools | The LLM's investigation job, grounded in real tool output |
| 6 | Visible tool calls: each call and its result shown in the transcript | Lets the user check the agent's work |
| 7 | Per-check explanation of why a device fails, including the "no data", "stale", "stopped reporting", and "this source is known to be unreliable" cases | The part the dashboard does not give |
| 8 | Recommendation of a catalog remediation per failing check, or an explicit "no reviewed fix exists" with the manual next step | The LLM's judgment job, bounded by the catalog |
| 9 | Fleet summary over the fixture fleet: failures per check and the fixes that would certify the most devices | Shows where effort pays off; computed by code, explained by the agent |
| 10 | Planning through the act server's `plan_remediation`, which changes nothing | Produces the frozen plan the person will approve |
| 11 | Approval screen implementing MCP form elicitation: every target, the remediation and its script identity, the approval code, approve with no default, decline, cancel | The reason this product exists |
| 12 | Execution through `execute_remediation` after approval, with the result and the audit reference shown; server refusals shown verbatim | Closes the loop; a refusal is information |
| 13 | Model provider chosen by configuration: a hosted OpenAI-compatible gateway, or Ollama locally | Must run on a laptop, including offline |
| 14 | Demo against a fixture fleet of about 20 devices covering every check, both device classes, a no-data device, and a device that stopped reporting, with a runner that changes nothing real | Demonstrable with no access to any real organization's systems |

## MVP scope: features OUT

| Feature | Reason | Target phase |
|---|---|---|
| Live connections to real device management, patching, endpoint protection, or backup systems | Real credentials can never be near a public course repository | Organization deployment, after the course |
| Reading the organization's compliance dashboard or its exports | That data names people and devices and stays private | Organization deployment |
| Remote fixes for disk encryption | Turning on encryption needs the user present and a restart; the console explains the steps instead | Re-evaluate after the course |
| Exemptions, or changing the baseline itself | Policy decisions belong to the people who own the baseline | Never in this product |
| The model writing or editing remediation scripts | The model picks a reviewed script, it never composes one | Never |
| Any approval made by the agent, a scheduled job, or a hook instead of a person | The approval is the safety boundary | Never |
| Remote shell or SSH actions | The engine acts only through the device management system's logged channel | Never in this product |
| Single sign-on, multiple users, hosting for a team | A local single-user demo is the course requirement | Organization deployment |
| Re-checking a device after a fix | Needs live data to mean anything | After the course |
| Trend over time | Needs repeated live snapshots | After the course |

## Risks

| Risk | Impact | Mitigation |
|---|---|---|
| The engine has no backup connector, no read tool that returns baseline evidence for the fixture fleet, and no demo mode | Backup cannot be checked, and there is nothing to demo against | Add a CrashPlan read connector to the engine (its API reports both when a device last connected and when it last completed a backup), and build the fixture data and demo mode, early in Construction, in the engine repository; the console's tests use recorded responses so they do not wait on it |
| The baseline rules in the policy file drift from the official program | The console reports a device compliant that the dashboard fails | Policy as data; check the rule engine against a sample of official results before any organizational use |
| The agent names the wrong cause or the wrong fix | A person could approve a fix that does not help | Pass or fail comes from the rule engine, fixes come from the catalog, the plan shows exactly what runs, and the person decides |
| Prompt injection through device data | The agent could be steered | Device data is read by the read server only; the act server reads nothing; every action needs a person |
| The approval screen cannot prove a human clicked | An automated client could approve | Stated plainly: the engine records `human_verified: false`; the console has no automatic answer path, and tests prove decline and cancel never execute |
| Small local models call tools poorly | The offline fallback may give weak answers | Run the scenario suite per model and report the results honestly |
| Solo developer, about three weeks of build time | Scope overrun | MVP limited to the features above; the fleet summary (feature 9) is the first to drop if time runs short |

## Open questions

These are deliberately unresolved, for Requirements Analysis to settle.

1. **Screen-lock threshold.** The organization-wide rule passes a Windows
   timeout from 1 second to 30 minutes (0, meaning never, fails) and judges
   macOS only on whether a lock is enabled. The local unit enforces a stricter
   15 minutes. Should the console report against the official limit, the local
   one, or both, and should it check the macOS timeout even though the
   official rule does not?
2. **Device class source.** Certification applies only to personal
   productivity devices; other devices get every check except screen lock,
   without certification. Where does the device class come from: a field in
   the device management system, the inventory record, or the policy file?
   And what does the console do when the field is missing?
3. **Backup evidence.** The official check passes when the backup client has
   connected within 30 days, which is not the same as a completed backup. The
   engine's new CrashPlan connector can return both: last connected, and last
   completed backup per destination (which can be empty). Should the console
   fail on the official rule only, and report the last completed backup
   beside it, or apply a stricter local rule as it does for screen lock?
4. **Which checks get a remote fix in the MVP?** Candidates: reinstall the
   endpoint protection agent, apply the screen-lock policy, clear a stuck
   update cache. Which are safe to demonstrate, and which stay "explain only"?
5. **Agent runtime.** Should the agent loop run in-process on the laptop, or
   on a hosted managed-agent service the developer already operates? Unknown:
   whether that service can reach MCP servers on a laptop or home network, and
   whether it passes MCP form elicitation through to a person.
6. **Interface.** Terminal interface or a local web page? The approval screen
   must show up to 25 targets and a script identity without truncation.
7. **Authentication to the engine in the demo.** The engine refuses to execute
   without a bearer token. Use a local throwaway token issuer, or something
   else?
8. **Offline model.** The hosted model is Claude Opus 5.5 through an
   OpenAI-compatible gateway, and routine scenario-suite runs use Claude
   Sonnet 5.5 to control cost. Which Ollama model is the offline fallback,
   given tool-calling quality?
