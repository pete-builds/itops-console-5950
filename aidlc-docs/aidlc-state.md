# AI-DLC State Tracking

## Project Information
- **Project Name**: itops-console
- **Project Type**: Greenfield
- **Start Date**: 2026-11-17T19:05:00Z
- **Current Stage**: INCEPTION - Application Design (next; Workflow Planning approved 2026-11-19). Milestone: stop after Units Generation, before Construction.
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
| Security Baseline | Yes (Q29 = A) | Requirements Analysis |
| Resiliency Baseline | No (Q30 = B) | Requirements Analysis |
| Property-Based Testing | Partial (Q31 = B: enforced rules PBT-02, 03, 07, 08, 09; others advisory) | Requirements Analysis |

## Stage Progress
### 🔵 INCEPTION PHASE
- [x] Workspace Detection (2026-11-17)
- [-] Reverse Engineering (skipped: greenfield)
- [x] Requirements Analysis (approved 2026-11-19)
- [x] User Stories (approved 2026-11-19)
- [x] Workflow Planning (approved 2026-11-19 after one Request Changes)
- [ ] Application Design - EXECUTE
- [ ] Units Generation - EXECUTE

### 🟢 CONSTRUCTION PHASE
- [ ] Functional Design - EXECUTE (per unit)
- [ ] NFR Requirements - EXECUTE (per unit, light)
- [ ] NFR Design - EXECUTE (per unit)
- [-] Infrastructure Design - SKIP (local only, no cloud)
- [ ] Code Generation - EXECUTE (Part 2 gated on stakeholder check-in)
- [ ] Build and Test - EXECUTE

### 🟡 OPERATIONS PHASE
- [ ] Operations (placeholder)

## Session Notes (Requirements Analysis)
- **Answer mode**: at the user's request, questions are asked in the chat one at a time (the user may dictate answers by voice). Each answer is written into its `[Answer]:` tag in `aidlc-docs/inception/requirements/requirement-verification-questions.md`, which remains the record, and logged in `audit.md`.
- **Free-form answers**: match to an option, read the interpretation back, and ask short lettered follow-ups (for example "1 A, 2 B") when anything is ambiguous or contradicts an earlier answer.
- **Public repository**: redact internal URLs, hostnames and other organization-private details before writing user input to `audit.md` or any committed file; say so when doing it.
- **Gate before any code**: the user must check in with a named stakeholder before any code is written (console or engine). Do not start Code Generation Part 2 until the user confirms that check-in. (Name kept out of this public repository.) Status 2026-11-17: the stakeholder agreed to a call. UPDATE 2026-11-19: the call was held 2026-11-18 (proof of concept and demo agreed; start read-only and non-dangerous; see audit.md). Whether the gate is cleared is still the user's call; ask before Code Generation Part 2. He said another engineer is building a base template for the team's agent-core MCP servers, not ready yet. The project serves two goals: a class project (INFO 5940) and a contribution to the production rollout.
- **Risk to carry into requirements.md**: the engine's new connectors and evidence tool may have to follow the forthcoming agent-core MCP template, with timing unknown. The console depends only on the MCP tool contract and recorded responses (Q14 = A), which insulates it. Topics for the call: build against the current itops-mcp or wait for the template; who builds the connectors; whether the class demo can run on fixtures alone.
- **Resolved decision (2026-11-19)**: class project keeps the real-engine plan (Q14 = A); the stakeholder check-in gates all code. Q15 = A. Q16 = A. Q17 = A. Q18 = A. Q19 = A. Q20 = B (device list from the read server plus free text; code-side identity resolution). Q21 = C (assumption confirmed). Q22 = A. Q23 = B. Q24 = drop offline mode (gateway-only Claude; deviation from vision feature 13 and tech-env Ollama row). Q25 = N/A, Q26 resolved by Q24 (gateway path only). Q27 = A. Q28 = A. Q29 = A, Q30 = B, Q31 = B. requirements.md generated; awaiting approval. Next: User Stories (three personas, user-facing).
- **Last updated**: 2026-11-17T20:30:00Z
