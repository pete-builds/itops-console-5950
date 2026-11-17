# AI-DLC State Tracking

## Project Information
- **Project Name**: itops-console
- **Project Type**: Greenfield
- **Start Date**: 2026-11-17T19:05:00Z
- **Current Stage**: INCEPTION - Requirements Analysis (collecting answers: Questions 1-11 answered, next is Question 12 of 31)
- **AI-DLC Version**: v1.0.1
- **Rule Details Directory**: `.aidlc-rule-details/`

## Workspace State
- **Existing Code**: No
- **Programming Languages**: None yet (Python 3.12 planned, per `docs/tech-env.md`)
- **Build System**: None yet (uv planned, per `docs/tech-env.md`)
- **Project Structure**: Empty (documentation and AI-DLC rules only)
- **Reverse Engineering Needed**: No
- **Workspace Root**: /Users/ps959/code/itops-console

## Input Documents
- `docs/vision.md`: product vision, MVP in and out, risks, eight open questions
- `docs/tech-env.md`: local deployment, stack, prohibited libraries, security, testing, example code

## Code Location Rules
- **Application Code**: Workspace root (NEVER in aidlc-docs/)
- **Documentation**: aidlc-docs/ only
- **Structure patterns**: See code-generation.md Critical Rules

## Extension Configuration
| Extension | Enabled | Decided At |
|---|---|---|
| Security Baseline | Pending (Question 29) | Requirements Analysis |
| Resiliency Baseline | Pending (Question 30) | Requirements Analysis |
| Property-Based Testing | Pending (Question 31) | Requirements Analysis |

## Stage Progress
### 🔵 INCEPTION PHASE
- [x] Workspace Detection (2026-11-17)
- [-] Reverse Engineering (skipped: greenfield)
- [ ] Requirements Analysis (in progress: Q1-Q11 answered, Q12-Q31 open)
- [ ] User Stories
- [ ] Workflow Planning
- [ ] Application Design
- [ ] Units Generation

### 🟢 CONSTRUCTION PHASE
- [ ] To be determined in Workflow Planning

### 🟡 OPERATIONS PHASE
- [ ] Operations (placeholder)

## Session Notes (Requirements Analysis)
- **Answer mode**: at the user's request, questions are asked in the chat one at a time (the user may dictate answers by voice). Each answer is written into its `[Answer]:` tag in `aidlc-docs/inception/requirements/requirement-verification-questions.md`, which remains the record, and logged in `audit.md`.
- **Free-form answers**: match to an option, read the interpretation back, and ask short lettered follow-ups (for example "1 A, 2 B") when anything is ambiguous or contradicts an earlier answer.
- **Public repository**: redact internal URLs, hostnames and other organization-private details before writing user input to `audit.md` or any committed file; say so when doing it.
- **Open items to raise later**: Question 14 must address engine scope, which now needs three new read connectors (CrashPlan, TDX, Sassafras K2) plus a normalized evidence tool, fixture data and demo mode. Question 20 must cover device identity resolution by code (from Question 11).
- **Last updated**: 2026-11-17T19:43:00Z
