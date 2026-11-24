# Unit of Work Story Map: itops-console

**Stage**: INCEPTION - Units Generation
**Created**: 2026-11-24
**Companion to**: `unit-of-work.md`, `stories.md`

Each story has exactly one **primary** unit, where its acceptance criteria
are met and tested. **Contributing** units build parts the story uses.

---

## Story to unit

| Story | Title | Primary | Contributing |
|---|---|---|---|
| US-1.1 | Identify a device | U3 | U2 (read client) |
| US-1.2 | Scoring runs as soon as the device is known | U3 | U1 (scoring), U2 (scoring service, events, transcript) |
| US-1.3 | Only active devices are scored | U1 | U2 (scope data from the read server), U3 (message) |
| US-1.4 | Device class decides what is scored | U1 | |
| US-1.5 | Encryption result | U1 | |
| US-1.6 | Screen-lock result on both operating systems | U1 | |
| US-1.7 | Patch enrollment and OS patching results | U1 | |
| US-1.8 | Third-party patching result | U1 | |
| US-1.9 | Endpoint protection and backup results | U1 | |
| US-1.10 | Overall device status | U1 | |
| US-1.11 | Explanation with evidence | U3 | U2 (events, transcript) |
| US-1.12 | Recommendation from the catalog | U3 | |
| US-1.13 | Bounded agent and model outages | U3 | U2 (settings) |
| US-2.1 | Plan the endpoint protection reinstall | U3 | U2 (act client, act guard) |
| US-2.2 | Read the whole plan on the approval page | U4 | U2 (approval store), U3 (printer prints the link) |
| US-2.3 | Approve and see the result | U4 | U2 (act guard), U3 (execute command, printer) |
| US-2.4 | Anything but approval means nothing runs | U2 | U3 (execute command), U4 (decline and cancel on the page) |
| US-2.5 | Unexpected approval forms are refused | U2 | |
| US-3.1 | Gaps ranked by check | U5 | U1 (device reports) |
| US-3.2 | Remote fixes ranked by devices certified | U5 | U1 |
| US-3.3 | Devices that need something other than a fix | U5 | U1 (statuses) |
| US-3.4 | Narrative on request | U5 | U3 (LLM client) |
| US-4.1 | Browse past sessions | U4 | U2 (transcript reader) |
| US-4.2 | Complete, append-only records | U2 | |
| US-5.1 | Clean clone to working demo | U2 | All units (README sections); Build and Test (timed run on a second laptop) |
| US-5.2 | Open the web pages from the terminal | U4 | U2 (`main.py` prints the link) |
| US-5.3 | A demo fleet that never goes stale | U2 | Engine (fixture timestamps, external) |
| US-5.4 | Change a threshold without a release | U1 | U2 (`main.py` reads the file) |
| US-5.5 | Run the scenario suite | U6 | Build and Test (live runs) |
| US-5.6 | Model and limits configured, not coded | U2 | U3 (uses the limits) |
| MS-1 | A malicious web page tries to approve a plan | U4 | U2 (approval store) |
| MS-2 | Another local process opens the approval page | U4 | U2 (bind in `main.py`) |
| MS-3 | Replaying an earlier approval | U2 | U4 (route) |
| MS-4 | Prompt injection through device data | U3 | U1 (scores independent of the model), U2 (approval) |
| MS-5 | A tampered policy file | U1 | U2 (startup stops) |
| MS-6 | Fishing for secrets | U2 | U3 (prompts), U4 (pages, error pages) |

Misuse cases found in Application Design (tests only; no story change):
MS-7 misbehaving act server, primary U2; MS-8 terminal control sequences,
primary U3; MS-9 transcript path traversal, primary U4.

---

## Stories per unit

| Unit | Primary user stories | Primary misuse stories | Count |
|---|---|---|---|
| U1 Baseline core | US-1.3, US-1.4, US-1.5, US-1.6, US-1.7, US-1.8, US-1.9, US-1.10, US-5.4 | MS-5 | 9 + 1 |
| U2 Platform and engine | US-2.4, US-2.5, US-4.2, US-5.1, US-5.3, US-5.6 | MS-3, MS-6 | 6 + 2 |
| U3 Agent and chat | US-1.1, US-1.2, US-1.11, US-1.12, US-1.13, US-2.1 | MS-4 | 6 + 1 |
| U4 Web and approval | US-2.2, US-2.3, US-4.1, US-5.2 | MS-1, MS-2 | 4 + 2 |
| U5 Fleet summary | US-3.1, US-3.2, US-3.3, US-3.4 | | 4 + 0 |
| U6 Scenario suite | US-5.5 | | 1 + 0 |
| **Total** | | | **30 + 6** |

Every one of the 30 user stories and 6 misuse stories in `stories.md` has
exactly one primary unit.

## Personas per unit

| Persona | Units that serve them |
|---|---|
| P1 Technician | U1, U2, U3, U4, U6 |
| P2 Endpoint engineer | U5 (fleet), U1 (US-5.4 policy) |
| P3 Compliance or team lead | U4 (transcript viewer), U2 (records) |

If U5 is dropped, P2 keeps only the policy-file story (US-5.4); this is the
accepted cost recorded in FR-FLT-6.
