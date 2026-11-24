# Unit of Work Plan

**Stage**: INCEPTION - Units Generation
**Created**: 2026-11-24
**Inputs**: `application-design/` (components C1 to C31, services S1 to S8,
dependency rules B1 to B9), `stories.md` (30 user stories, 6 misuse
stories), `execution-plan.md`, `requirements.md` (FR-FLT-6, section 7 risks)
**Project shape**: one Python process (a monolith). A unit of work is a
logical group of modules and stories inside that one package, designed and
built in turn; units are not deployed separately.

**Answer mode** (session notes): questions are asked in the chat; each answer
is written into its `[Answer]:` tag below, which stays the record.

---

## Plan steps

### Part 1: Planning
- [x] 1. Analyze the application design, stories and execution plan (likely units sketched in `execution-plan.md`; FR-FLT-6 fleet drops first; unverified fastmcp elicitation and gateway tool calling)
- [x] 2. Draft this plan with checkboxes
- [x] 3. Include the mandatory artifacts (Part 2 below)
- [x] 4. Write decomposition questions (Q1 to Q5 below; category coverage noted)
- [x] 5. Collect answers to all questions
- [x] 6. Analyze answers for ambiguity or contradiction; add follow-ups if needed (none needed; see Answer analysis)
- [x] 7. Plan approval (approved 2026-11-24)

### Part 2: Generation (`aidlc-docs/inception/application-design/`)
- [x] 8. `unit-of-work.md`: each unit's name, purpose, components, services, responsibilities, definition of done
  - [x] 8a. Code organization strategy (greenfield): directory layout, test layout, where each unit's modules live
  - [x] 8b. What dropping the fleet summary means (FR-FLT-6)
- [x] 9. `unit-of-work-dependency.md`: unit dependency matrix, build order, integration and test-support approach, risk spikes
- [x] 10. `unit-of-work-story-map.md`: every user story and misuse story assigned to exactly one primary unit (other units noted)
- [x] 11. Validate: every component in exactly one unit; every story assigned; no cycles between units; the build order respects dependencies; the fleet unit can be dropped without changing another unit
- [x] 12. Extension compliance summary (Security Baseline; PBT Partial). No blocking findings
- [x] 13. Completion message and approval (approved 2026-11-24)

---

## Question category coverage

| Category | Covered by |
|---|---|
| Story grouping | Q1 |
| Dependencies | Q2, Q3 |
| Team alignment | Not asked: solo project with one owner (instructor approval, `README.md`); the engine's work is outside this workflow (Q14 = A), so no unit is shared with another person |
| Technical considerations | Q4. Every unit ships in the same local process, so there are no per-unit scaling or deployment differences to ask about |
| Business domain | Q1 (options follow the domains: scoring, engine and approval, agent and chat, web, fleet, evaluation) |
| Code organization (greenfield, multi-unit) | Q5 |

---

## Questions

## Question 1
How should the console be split into units of work? (Component IDs are from `components.md`.)

A) Six units, with the fleet summary on its own so it can be dropped whole: U1 Baseline core (C1 to C4); U2 Platform and engine integration (C6 to C11 without fleet scoring, plus cross-cutting C26 to C30); U3 Agent and terminal chat (C12 to C15, C17 to C20); U4 Web and approval page (C21 to C23, C25); U5 Fleet summary (C5, C16, C24, `score_fleet`); U6 Scenario suite (C31)

B) Five units as sketched in `execution-plan.md`: baseline core including fleet computation; engine integration with the cross-cutting components; agent and terminal chat including the narrative; web including the fleet page; scenario suite. Dropping the fleet summary means removing parts of three units

C) Three units: baseline core; the rest of the console; scenario suite. Fewest design gates, but one large unit

D) Other (please describe after [Answer]: tag below)

[Answer]: A (six units, fleet summary on its own)

## Question 2
In what order should the units be built? (Unit names from Q1 option A; adjusted if Q1 differs.)

A) U1 core, U2 platform and engine, U3 agent and chat, U4 web and approval page, U6 scenario suite, U5 fleet summary last. Read-only first and approval next, as the stakeholder advised; the scenario suite before the final milestone; the fleet last because it is the first to drop

B) U1, U2, U4, U3, U6, U5: the approval page before the agent, since it can be tested with a scripted plan

C) A walking skeleton across every unit first (one check, one tool, a bare approval page), then each unit in depth

D) Other (please describe after [Answer]: tag below)

[Answer]: A (U1, U2, U3, U4, U6, then U5 last)

## Question 3
How should units test against each other before the engine's demo mode exists?

A) One shared test-support package built in U2 and reused by later units: in-process fake read and act MCP servers (the act fake records every execution), recorded engine responses, a scripted fake model and the shared Hypothesis strategies (TST-PBT-4). The live engine is used only in Build and Test

B) Each unit builds its own fakes and recorded responses

C) Other (please describe after [Answer]: tag below)

[Answer]: A (shared test-support package built in U2)

## Question 4
How should the two unverified integration risks be retired: fastmcp 4 answering form elicitation (revision 2026-07-28), and tool calling through the gateway?

A) A short spike at the start of Construction, before U2's Functional Design: two throwaway scripts kept outside the package, with the results recorded in `aidlc-docs/`. The spike is code, so it waits for the same stakeholder gate

B) Verify each one inside the unit that needs it: elicitation in U2's Code Generation, gateway tool calling in U3's

C) Other (please describe after [Answer]: tag below)

[Answer]: A (spike at the start of Construction, before U2's Functional Design; gated like code)

## Question 5
How should the code be organized on disk? (AI-DLC's default for a multi-unit monolith is `src/{unit-name}/` and `tests/{unit-name}/`; Application Design Q1 kept the `tech-env.md` structure.)

A) Keep the `tech-env.md` layout: the `itops_console/` package at the repository root, each unit a group of modules inside it, and tests by kind as in `tech-env.md` (`tests/unit/`, `tests/agent/`, `tests/mcp/`, `tests/web/`) plus `tests/support/` (Q3) and `tests/architecture/` (rules B1 to B9). The difference from the AI-DLC default is recorded in `unit-of-work.md`

B) The AI-DLC default: `src/itops_console/` with one sub-package per unit and `tests/{unit-name}/`. This renames `tech-env.md` modules, which conflicts with Application Design Q1 = A

C) A `src/` layout (`src/itops_console/`) that keeps the `tech-env.md` module names, with tests per unit (`tests/{unit-name}/`)

D) Other (please describe after [Answer]: tag below)

[Answer]: A (keep the `tech-env.md` layout; record the difference from the AI-DLC default)

---

## Answer analysis (step 6)

All five answers pick a single listed option (Q1 to Q5 = A). None is vague or
combines options. Checked for conflicts:

- Q1 (approval store and act guard in U2) and Q2 (read-only first, approval
  next): consistent. U2 builds and tests the approval store and act guard
  against fakes only. No person can reach an execution until U3 adds the
  technician's execute command and U4 adds the approval page, so the order
  of what a user can do is read-only first, then approval.
- Q1 (fleet summary as U5) and Q2 (U5 last): consistent with FR-FLT-6. U5
  adds `score_fleet` to the scoring service (U2) and two routes to `app.py`
  (U4) without changing either unit's existing behavior; skipping U5 leaves
  them as built.
- Q3 (shared test support in U2) and Q2: consistent; U2 is built before every
  unit that reuses the fakes.
- Q4 (spike before U2's Functional Design) and the stakeholder gate: the
  spike is code, so it waits for the gate. U1's Functional Design (no code)
  can start before it.
- Q5 (`tech-env.md` layout) and Application Design Q1 = A: consistent.

No follow-up questions needed.
