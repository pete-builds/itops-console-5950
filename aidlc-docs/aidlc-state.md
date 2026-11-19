# AI-DLC State Tracking

## Project Information
- **Project Name**: itops-console
- **Project Type**: Greenfield
- **Start Date**: 2026-11-17T19:05:00Z
- **Current Stage**: INCEPTION - Requirements Analysis (PAUSED until the user's stakeholder call on 2026-11-18. Questions 1-14 answered; a proposed scope change is open (see "Pending decision" in Session Notes); then Question 15 of 31)
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
- [ ] Requirements Analysis (in progress: Q1-Q14 answered, Q15-Q31 open)
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
- **Open items to raise later**: Question 20 must cover device identity resolution by code (from Question 11).
- **Gate before any code**: the user must check in with a named stakeholder before any code is written (console or engine). Do not start Code Generation Part 2 until the user confirms that check-in. (Name kept out of this public repository.) Status 2026-11-17: the stakeholder agreed to a call (not yet held). He said another engineer is building a base template for the team's agent-core MCP servers, not ready yet. The project serves two goals: a class project (INFO 5940) and a contribution to the production rollout.
- **Risk to carry into requirements.md**: the engine's new connectors and evidence tool may have to follow the forthcoming agent-core MCP template, with timing unknown. The console depends only on the MCP tool contract and recorded responses (Q14 = A), which insulates it. Topics for the call: build against the current itops-mcp or wait for the template; who builds the connectors; whether the class demo can run on fixtures alone.
- **Pending decision (resume here)**: the user proposed making the class project a standalone demo with dry runs, separate from the production rollout. Two follow-ups are open: (1) demo backing: A standalone with in-repo fake read/act MCP servers built to the engine's contract (recommended), B standalone with no contract fidelity, C keep the real-engine plan (Q14 = A); (2) does the stakeholder check-in still gate code: A yes for all code, B not for the class demo. If 1 is A or B: change Q14 to C, update the vision wording for features 5 and 10-12 to "an engine-compatible demo server", and revisit Q16 and Q19. Then present Q15 (recommendation A, relative timestamps).
- **Last updated**: 2026-11-17T20:30:00Z
