# Story Generation Plan

**Stage**: INCEPTION - User Stories, Part 1 (Planning)
**Created**: 2026-11-19
**Input**: `aidlc-docs/inception/requirements/requirements.md` (approved 2026-11-19)
**Assessment**: `aidlc-docs/inception/plans/user-stories-assessment.md`

**How to answer**: as in Requirements Analysis, questions are asked in the chat
one at a time and each answer is written into its `[Answer]:` tag here.

---

## Breakdown approaches considered

| Approach | Strength | Weakness here |
|---|---|---|
| User journey-based | Reads like the demo: identify device, see scores, read cause, approve fix | Shared rules (scoring, security) get split across journeys |
| Feature-based | Maps cleanly to modules and requirement IDs | Loses the persona's point of view |
| Persona-based | Clear ownership per persona | The technician owns most stories, so groups are unbalanced |
| Domain-based | Separates scoring, agent, approval, records | Similar to feature-based |
| Epic-based | Hierarchy that scales; epics can follow journeys | Needs a rule for what makes an epic |

---

## Questions

## Question 1: Story breakdown approach
How should the stories be organized?

A) Epics by user journey (for example "Check one device", "Fix a device with approval", "Review the fleet", "Review past sessions", "Set up and run the demo"), with stories under each epic

B) Feature-based: one group per feature area (rule engine, chat, agent, approval, fleet, transcripts)

C) Persona-based: one group per persona

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 2: Story size
How fine-grained should stories be?

A) Medium: about 20 to 30 stories, each covering a coherent slice (for example "screen-lock scoring for both operating systems" is one story)

B) Fine: about one story per functional requirement (about 60 stories)

C) Coarse: about 10 stories, with detail carried in acceptance criteria

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 3: Acceptance criteria format
A) Given / When / Then scenarios for every story

B) Checklist bullets for every story

C) Given / When / Then for scoring and approval stories (where exact behavior is tested); checklist bullets for the rest

X) Other (please describe after [Answer]: tag below)

[Answer]: C

## Question 4: Personas
A) The three personas from the vision: desktop support technician, endpoint engineer, compliance or team lead

B) The three, plus a fourth: the course evaluator who clones the repository, runs the demo and reads the scenario results

C) The three, plus the evaluator and the production stakeholder who decides whether the console joins the rollout

X) Other (please describe after [Answer]: tag below)

[Answer]: A

## Question 5: Security and misuse stories
A) Include misuse stories written from the attacker's side (for example "a malicious web page tries to approve a plan"), each with acceptance criteria that the attack fails

B) No separate misuse stories; security behavior appears only as acceptance criteria on normal stories

X) Other (please describe after [Answer]: tag below)

[Answer]: A

---

## Generation checklist (Part 2, executed after plan approval)

- [x] 1. Generate `aidlc-docs/inception/user-stories/personas.md` per the Question 4 answer: goals, context, pain points, surfaces used
- [x] 2. Generate `aidlc-docs/inception/user-stories/stories.md` organized per Question 1, sized per Question 2
  - [x] 2.1 Story template: "As a [persona], I want [capability], so that [benefit]"
  - [x] 2.2 Acceptance criteria per Question 3
  - [x] 2.3 Each story lists the requirement IDs it covers (FR-*, NFR-*, SEC-REQ-*, TST-PBT-*)
  - [x] 2.4 Misuse stories per Question 5
- [x] 3. Check every story against INVEST (Independent, Negotiable, Valuable, Estimable, Small, Testable) and record the check
- [x] 4. Add a persona-to-story map in `stories.md`
- [x] 5. Add a coverage table: every functional requirement in `requirements.md` maps to at least one story; list any requirement with no story and why
- [x] 6. Security Baseline compliance check for the stage (SECURITY-11 misuse scenarios in particular)
