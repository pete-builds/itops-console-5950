# itops-console

A local AI assistant that helps IT support staff bring Windows and macOS
computers up to an endpoint security baseline, and that never changes a
computer without a person approving the exact action.

INFO 5940, Fall 2026: mid-semester milestone (AI-DLC Inception) and final
project.

| | |
|---|---|
| **Team** | Pete Stergion, NetID `ps959` (solo project, approved in writing by the course instructor on November 2, 2026) |
| **Coding agent** | Claude Code, running Claude Opus 5.5 |
| **Methodology** | AI-DLC v1.0.1 ([release](https://github.com/awslabs/aidlc-workflows/releases/tag/v1.0.1)) |
| **LLM in the product** | Claude Opus 5.5 through the university's OpenAI-compatible AI gateway; Claude Sonnet 5.5 for test runs; an Ollama model as the offline fallback |
| **Runs** | Locally on one laptop |

## Problem and why an LLM fits

Organizations that manage hundreds of computers usually require each one to
meet a security baseline: endpoint protection installed and checking in,
operating system and third-party patches current, a screen lock within a set
timeout, disk encryption, and a recent backup. A compliance dashboard reports
which check failed, but not why, and the evidence for each check lives in a
different system: the device management consoles (Jamf Pro for Macs, Intune or
Configuration Manager for Windows), a vulnerability scanner, a patch inspector,
the endpoint protection console, and the backup service. Fixing one failing
device means opening five consoles and working out which of several causes
applies. The rules also have traps: "no data" is not "failed", every check has
its own freshness window, some data sources are known to be inaccurate for
certain operating systems, and shared devices follow different rules from a
person's own laptop. The users are desktop support technicians, the endpoint
engineers who plan fleet-wide fixes, and the leads who review what was done.

An LLM is the right tool for the parts of this that are open-ended: deciding
which systems to query for a device, reading their results together,
explaining in plain language why a check fails, and matching the cause to a
reviewed fix, or saying plainly that none exists. It is the wrong tool for the
parts that must be exact, so the design keeps those away from it. Whether a
20-minute screen lock meets a 30-minute rule is arithmetic, done by a rule
engine configured from a policy file, so the result is the same every time and
testable. The model never writes code that runs on a computer, never decides
which machines are in scope, and never approves anything.

## How the LLM is integrated

itops-console is an agent with tools. The agent (Claude Opus 5.5, called
through the OpenAI-compatible gateway with the `openai` client) answers a
technician's question by calling read-only tools on itops-mcp, a governed MCP
server I own and built separately with Claude Code under my direction. The
console's rule engine scores each check as pass, fail, no data, or stale, and
marks results whose source is known to be unreliable. The agent then explains
each failure from the evidence and recommends a fix from a catalog of
reviewed remediation scripts. To act, it asks the itops-mcp act server to plan
the fix, which changes nothing. The act server then requests a person's
approval through MCP form elicitation, and the console shows that request on a
screen listing every target computer and exactly which reviewed script will
run. Only a person's click on that screen lets the action proceed, and the
engine writes its audit record before anything runs. Every tool call the agent
makes is shown in the transcript. Features that depend on the LLM: the
investigation, the explanations, the recommendations, and the fleet summary's
narrative. Features that deliberately do not: scoring, scope, and approval.

This piece is the missing half of itops-mcp. The engine already separates
reading from acting and asks for approval through elicitation, but in tests on
2026-10-02 no available MCP client could carry that approval to a person: one
truncated the form and hid the targets, one did not support the protocol
revision, and the gateway dropped the approval fields. A client built for
this one job is what makes the governed design usable.

## Scope and feasibility

By the final deadline (December 16), the console will run on a laptop against
the engine in demo mode with a fixture fleet of about 20 computers. A
technician will be able to ask about any device and get every check scored
with an explanation, see a fleet summary of which fixes would bring the most
devices into compliance, and take a recommended fix through planning,
approval and execution, with refusals shown verbatim. Success is measured,
not asserted: the rule engine is tested at every boundary, tests prove that
declining, cancelling, or closing the approval never runs anything, and a
ten-scenario evaluation must get the cause and the fix right at least 8 times
in 10. Out of scope: live connections to real systems (credentials never
belong near a public repository), hosting for a team, and anything that lets
the model write scripts or approve actions. The plan is achievable solo
because the hard server-side parts (authentication, scope, the catalog, the
approval protocol and the audit log) already exist in the engine; the
remaining engine work (a CrashPlan backup connector, baseline evidence for
the fixture fleet, and a demo mode) is planned for the first week of
Construction, and the fleet summary is the first feature to drop if time runs
short.

## Collaboration plan

This is a solo project, approved in writing by the course instructor on November 2, 2026 (the approval covered IT-Ops-MCP; this console is its approval half), so the
plan covers how the work is paced, reviewed, and communicated rather than how
it is split. I work in two scheduled sessions a week: the in-class AI-DLC
build sessions (November 19 and 24) and a weekend session, plus short
sessions as needed. Every session ends with a commit and push, and
`aidlc-docs/aidlc-state.md` shows which stage is in progress. Decisions are
mine and are recorded in `aidlc-docs/audit.md` through the AI-DLC question and
approval flow; before approving a major artifact, I ask for an independent
critique in a fresh agent context, as AI-DLC recommends, and record what I
changed because of it. I bring open questions to course office hours and
communicate with course staff through Canvas. The engine lives in its own
repository; changes to it are tracked there and referenced here.

## Repository layout

| Path | What it is |
|---|---|
| `README.md` | This overview |
| `docs/vision.md` | AI-DLC input: what is being built, for whom, MVP in and out, open questions |
| `docs/tech-env.md` | AI-DLC input: stack, local deployment, testing, prohibited libraries, security, example code |
| `CLAUDE.md` | AI-DLC v1.0.1 core workflow rules for Claude Code (a copy of `core-workflow.md`) |
| `.aidlc-rule-details/` | AI-DLC v1.0.1 rule details |
| `aidlc-docs/` | Everything AI-DLC generates, including `audit.md` and `aidlc-state.md` |
| Repository root (from Construction on) | Application code; never inside `aidlc-docs/` |

No API keys or other secrets are committed. Local settings live in `.env`,
which is gitignored.
