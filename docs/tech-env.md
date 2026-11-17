# Technical environment: itops-console

*Greenfield. Local only. Companion to `docs/vision.md`.*

## Deployment model and cloud provider

**The product runs locally on one laptop. Nothing is deployed to a cloud.**

| Process | Where | Address |
|---|---|---|
| itops-console (this repository): web UI, agent loop, rule engine | Python process on the laptop | `http://127.0.0.1:8080` |
| itops-mcp read server (the engine, in its own repository) | Docker container on the laptop, demo mode, fixture data | `http://127.0.0.1:8000/mcp` |
| itops-mcp act server | Docker container on the laptop, demo mode, no-op runner | `http://127.0.0.1:8001/mcp` |
| Token issuer for the demo | The engine's local test issuer, on the laptop | localhost only |
| LLM, default | Claude Opus 5.5 (`claude-opus-5-5`) through the university's OpenAI-compatible AI gateway (hosted, LiteLLM) | `https://api.ai.it.cornell.edu/v1` |
| LLM, scenario-suite runs | Claude Sonnet 5.5 (`claude-sonnet-5-5`), same gateway | same |
| LLM, offline fallback | Ollama on the laptop, OpenAI-compatible endpoint; model chosen during Requirements Analysis | `http://127.0.0.1:11434/v1` |

The hosted gateway is the only network destination outside the laptop. Both
gateway models were confirmed present with `GET /v1/models` on 2026-11-17
(the real key returned 200, a fake key 401). Tool-calling through the gateway
has not been tested yet.

### Cloud services

| Allowed | Constraint |
|---|---|
| The university AI gateway | LLM calls only; key from `.env`; nothing else sent there |

| Not allowed | Reason |
|---|---|
| Any cloud hosting (AWS, Azure, GCP, Vercel, Fly, Render) | The course requires local only, and a hosted approval page would need real authentication |
| Any real device-management, patching, endpoint-protection, or backup API | Real credentials never come near a public course repository; the demo uses fixture data |
| Telemetry, analytics, error-reporting services (Sentry, PostHog, LangSmith, Langfuse cloud) | Transcripts contain device data; they stay on the laptop |

## Languages

| Category | Language | Version | Use |
|---|---|---|---|
| Required | Python | 3.12 (matches the engine's container) | Everything server-side |
| Permitted | HTML and CSS | n/a | Jinja2 templates |
| Permitted | YAML | n/a | Baseline policy and scenario files, loaded with `yaml.safe_load` only |
| Permitted | JSON | n/a | Fixtures, transcripts (JSON Lines), MCP payloads |
| Prohibited | JavaScript or TypeScript application code | n/a | One vendored HTMX file is the only script; no build step |

## Package manager

**uv.** Dependencies are declared in `pyproject.toml` with each major version
capped (`>=X,<X+1`), resolved into `uv.lock`, which is committed and records a
sha256 hash for every package. Install with `uv sync --locked`, so a tampered
or changed package fails the install instead of running. Run with
`uv run itops-console`.

## Frameworks and libraries

### Required

| Library | Version range | Purpose |
|---|---|---|
| fastapi | `>=0.115,<1` | Web app: pages, approval form, health route |
| uvicorn | `>=0.30,<1` | ASGI server bound to 127.0.0.1 |
| jinja2 | `>=3.1,<4` | Server-rendered templates, autoescape on |
| htmx | one pinned file, vendored in `static/`, sha256 recorded | Partial page updates without a JavaScript build |
| openai | `>=1.50,<2` | One client for the gateway and Ollama, both OpenAI-compatible; tool calling |
| fastmcp | `>=4,<5` (the engine's major) | MCP client for the read and act servers, including the elicitation handler that receives the approval form |
| pydantic | `>=2.7,<3` | Validation of settings, form input, policy file, tool arguments |
| pyyaml | `>=6,<7` | Baseline policy and scenario files |
| httpx | `>=0.28,<1` | Async HTTP (also used by fastmcp and openai) |

To verify early in Construction: that the fastmcp 4 client negotiates MCP
revision 2026-07-28 and answers form elicitation from the engine's act server.
The engine's own approval tests exercise that path and are the reference.

### Development only

| Library | Purpose |
|---|---|
| pytest, pytest-asyncio, pytest-cov | Tests, async tests, coverage |
| ruff | Lint and format |
| mypy | Type checking, strict on `baseline/` |
| pip-audit | Known-vulnerability scan of the locked dependencies |
| gitleaks (binary, in CI) | Secret scan over full history |

### Prohibited libraries

| Prohibited | Reason | Use instead |
|---|---|---|
| `requests` | Synchronous: one slow call blocks the event loop that also serves the approval page | `httpx.AsyncClient` |
| LangChain, LangGraph, LlamaIndex, CrewAI, AutoGen | They hide the tool loop and the approval pause behind abstractions, and the loop is exactly what this product must show, test and audit | A short hand-written agent loop on the `openai` client (about 100 lines) |
| `anthropic`, `google-genai`, other provider-specific SDKs | Tie the product to one provider; the gateway and Ollama both speak the OpenAI format | `openai` client with `base_url` from settings |
| Streamlit, Gradio, Chainlit | Their rerun and session models cannot hold an approval request open while the MCP call waits for the person, and they limit control over showing all targets | FastAPI, Jinja2 and HTMX |
| React, Angular, Vue, any npm toolchain | A second language and a build pipeline for a handful of pages | Server-rendered Jinja2 with HTMX |
| Flask, Django | Synchronous by default (Flask) or far more than needed (Django) | FastAPI |
| paramiko, asyncssh, fabric, `subprocess` aimed at endpoints | The product never touches a computer directly; the engine acts only through the device management system | The itops-mcp act server |
| SQLAlchemy, any database server | Single local user with append-only records | JSON Lines files written with the standard library |
| `pickle`, `shelve` | Loading them runs arbitrary code | `json` |
| PyJWT, python-jose, any token minting or validation in the console | Authentication is the engine's job; a second implementation would drift | Pass the demo issuer's short-lived tokens through unchanged |
| `yaml.load` with the default loader | Can construct arbitrary Python objects | `yaml.safe_load` |

## Architecture and patterns

- **Async throughout.** One event loop runs the web app, the agent loop, and
  the MCP clients, so an approval can wait for a person without blocking
  anything else.
- **The rule engine is pure.** `baseline/rules.py` takes evidence, a rule and
  the current time, and returns a result. It imports nothing that does I/O,
  calls a model, or reads the clock. Pass or fail is never the model's call.
- **The model only sees tools and evidence.** It calls read tools, receives
  their results as data, and proposes plans. The rule engine's results are
  given to it as facts to explain, not questions to decide.
- **Approval is a pause, not a value.** When the act server sends its approval
  form, the console stores it as an open approval (an `asyncio.Future` keyed by
  id) and shows it. Only the approval route, driven by a person, resolves it.
  There is no code path that answers it automatically. Unanswered approvals
  time out as cancelled.
- **Every tool call is recorded.** The transcript (JSON Lines) records each
  model turn, tool call, result, approval and answer, with the engine's audit
  reference where one exists.
- **Configuration from the environment.** Settings load from `.env` into a
  pydantic model at startup; the console refuses to start if a required value
  is missing.

### Project structure

```
itops_console/
  app.py               FastAPI app and routes
  settings.py          settings from the environment, validated at startup
  llm.py               openai client for gateway or Ollama
  agent/loop.py        tool-calling loop, step and token limits
  agent/prompts.py     system prompt
  mcp/clients.py       read and act MCP clients
  mcp/approvals.py     open approvals (elicitation requests) awaiting a person
  baseline/models.py   evidence and result types
  baseline/policy.py   load and validate the baseline policy
  baseline/rules.py    pure scoring functions, one per check
  baseline/fleet.py    fleet summary: fixes that certify the most devices
  transcript.py        JSON Lines transcript
  templates/           Jinja2 pages
  static/htmx.min.js   vendored, hash recorded
policy/baseline.example.yaml
evals/scenarios/       ten scenario files
evals/run.py           scenario runner (not run in CI; it calls a model)
evals/results/         committed results per model and date
tests/unit/            rule engine, policy, fleet summary
tests/agent/           agent loop against a scripted fake model
tests/mcp/             clients against recorded responses and a fake act server
tests/web/             routes, approval page, CSRF and origin checks
docs/                  vision.md, tech-env.md
.env.example           every variable, no values
```

### Settings

| Variable | Example | Purpose |
|---|---|---|
| `LLM_BASE_URL` | `https://api.ai.it.cornell.edu/v1` or `http://127.0.0.1:11434/v1` | Provider |
| `LLM_API_KEY` | (secret) | Gateway key; any placeholder for Ollama |
| `LLM_MODEL` | `claude-opus-5-5` | Model for the product |
| `LLM_EVAL_MODEL` | `claude-sonnet-5-5` | Model for routine scenario-suite runs |
| `ITOPS_READ_URL`, `ITOPS_ACT_URL` | `http://127.0.0.1:8000/mcp`, `http://127.0.0.1:8001/mcp` | Engine servers |
| `ITOPS_READ_TOKEN`, `ITOPS_ACT_TOKEN` | (secret, short-lived) | Bearer tokens from the demo issuer |
| `BASELINE_POLICY_PATH` | `policy/baseline.example.yaml` | Baseline rules |
| `AGENT_MAX_STEPS`, `AGENT_MAX_TOKENS` | `15`, `60000` | Per-question limits |
| `APPROVAL_TIMEOUT_SECS` | `300` | An unanswered approval is cancelled |

### Baseline policy (example)

Rule values follow a real university's published compliance definitions.

```yaml
version: 1
device_classes:
  personal_productivity: {certify: true}
  shared: {certify: false}
  special_purpose: {certify: false}
non_certified_classes_skip: [screen_lock]   # still scored on everything else
checks:
  encryption:
    freshness_days: 30            # all fixed volumes encrypted
  screen_lock:
    freshness_days: 30
    max_timeout_secs: 1800        # official limit; 0 means never and fails
    local_max_timeout_secs: 900   # stricter local rule, reported beside it
    macos_enabled_only: true      # official macOS rule judges "enabled" only
  os_patching:
    freshness_days: 30
    unsupported_os_fails: true    # n-2 macOS, out-of-support Windows 10
  third_party_patching:
    windows_inspection_days: 7
    macos_freshness_days: 30
  endpoint_protection:
    checkin_days: 14
  backup:
    last_connected_days: 30       # official rule
    report_last_completed: true   # shown beside it, never hidden
unreliable_sources:
  - {check: screen_lock, os: macos, note: "publisher marks macOS screen-lock data inaccurate"}
  - {check: os_patching, os: windows, min_release: "25H2", note: "publisher marks Windows 11 25H2+ patch status inaccurate"}
```

## Security

### Basics

- **Authentication.** The console is single-user and binds to `127.0.0.1`
  only, so it has no login. Every POST checks the `Origin` header and a CSRF
  token, because any web page the user visits can send a request to
  localhost. Authentication to the engine uses short-lived bearer tokens from
  the engine's demo issuer; the console never mints or validates tokens.
- **Input validation.** Every form, setting, policy file and tool argument
  is parsed into a pydantic model; anything that fails is refused with the
  reason. YAML is loaded with `safe_load` only.
- **Secrets.** In `.env`, which is gitignored. `.env.example` is tracked and
  lists every variable with no values. Secrets are never logged, rendered,
  or put in a prompt. The gateway key is sent only to the gateway.
- **Output handling.** Jinja2 autoescape is on. Tool output and model text
  are rendered as text, never as HTML.
- **Approvals.** No automatic answer path exists. The approve box has no
  default. Decline, cancel, closing the page, a timeout, and a replayed answer
  all mean nothing runs. The console states plainly that it cannot prove a
  person clicked; the engine records `human_verified: false`.
- **Supply chain.** `uv.lock` with hashes, `pip-audit` in CI, GitHub Actions
  pinned to commit SHAs, the vendored HTMX file checked against its recorded
  hash.
- **Data.** Fixture data only. No real hostnames, serial numbers, or people
  in the repository. A CI check fails on any term from a list supplied out of
  band (the same pattern as the engine's forbidden-terms check), and gitleaks
  scans full history with a planted-secret control.

### Compliance framework: OWASP Top 10 for LLM Applications (2025)

| Category | Control in this product |
|---|---|
| LLM01 Prompt injection | Device data reaches the model only as tool results, labeled as data; the system prompt says tool output is never instructions; the act server reads nothing; every action needs a person |
| LLM02 Sensitive information disclosure | Fixture data only; secrets never in prompts or transcripts; transcripts stay on the laptop |
| LLM03 Supply chain | Hash-locked dependencies, pip-audit, SHA-pinned Actions, vendored HTMX with a recorded hash |
| LLM04 Data and model poisoning | Not applicable: no training or fine-tuning |
| LLM05 Improper output handling | Model output is escaped text; it never becomes code, HTML, a shell command, or a script |
| LLM06 Excessive agency | The model can read and propose only; actions are reviewed catalog scripts; a person approves each one; scope comes from policy |
| LLM07 System prompt leakage | The system prompt holds no secrets or scope rules, so leaking it exposes nothing |
| LLM08 Vector and embedding weaknesses | Not applicable: no retrieval index |
| LLM09 Misinformation | Pass or fail comes from the rule engine, not the model; results from unreliable sources are labeled; every claim links to the tool call behind it |
| LLM10 Unbounded consumption | Step and token limits per question; approval timeout; routine test runs on the cheaper model |

## Testing

| Type | Tooling | What it proves | Target |
|---|---|---|---|
| Rule engine | pytest | Every check, both operating systems, every device class, boundary values for timeouts and freshness windows | 100% branch coverage of `baseline/rules.py` |
| Policy loader | pytest | Invalid or unsafe policy files are refused with a reason | All refusal paths |
| Agent loop | pytest-asyncio, a scripted fake model | Tool calls are made and recorded; step and token limits stop the loop; no network | All loop exits |
| MCP clients | pytest-asyncio, recorded engine responses, a fake act server | Refusals surface verbatim; tokens are sent; nothing calls a live upstream | All tools used |
| Approval safety | pytest-asyncio, fake act server that records executions | Decline, cancel, close, timeout and replay cause zero executions; a control test with the approval check removed must fail | 0 unapproved executions |
| Web | FastAPI test client | A 25-target plan renders every target and the script identity; POSTs without a valid origin and CSRF token are refused | Every route |
| Scenario suite | `evals/run.py` | The ten scenarios: correct cause and correct fix or "no reviewed fix" | 8 of 10 or better; Sonnet 5.5 routinely, Opus 5.5 before each milestone; results committed |

Overall line coverage target: 85%, excluding `evals/`. Unit tests never call a
model or a live server.

**CI gates** (GitHub Actions, every push and pull request): `ruff check`,
`ruff format --check`, `mypy`, `pytest` with coverage thresholds,
`pip-audit`, gitleaks over full history, and the forbidden-terms check. The
scenario suite is not in CI because it calls a paid model.

## Example code

### A typical function: score one check

```python
# itops_console/baseline/rules.py
from dataclasses import dataclass
from datetime import datetime, timedelta
from enum import StrEnum


class Outcome(StrEnum):
    PASS = "pass"
    FAIL = "fail"
    NO_DATA = "no_data"
    STALE = "stale"
    NOT_APPLICABLE = "not_applicable"


@dataclass(frozen=True)
class ScreenLockRule:
    max_timeout_secs: int          # 1800 is the official 30-minute limit
    freshness: timedelta           # inventory older than this is stale
    macos_enabled_only: bool       # official macOS rule judges "enabled" only
    applies_to: frozenset[str]     # device classes scored on this check


@dataclass(frozen=True)
class ScreenLockEvidence:
    os: str                        # "windows" or "macos"
    device_class: str
    enabled: bool | None
    timeout_secs: int | None
    reported_at: datetime | None


@dataclass(frozen=True)
class CheckResult:
    check: str
    outcome: Outcome
    reason: str
    source_reliable: bool = True


def evaluate_screen_lock(
    ev: ScreenLockEvidence,
    rule: ScreenLockRule,
    now: datetime,
    unreliable_os: frozenset[str] = frozenset(),
) -> CheckResult:
    """Score screen lock for one device. Pure: no I/O, no clock, no model."""
    reliable = ev.os not in unreliable_os

    def result(outcome: Outcome, reason: str) -> CheckResult:
        return CheckResult("screen_lock", outcome, reason, reliable)

    if ev.device_class not in rule.applies_to:
        return result(Outcome.NOT_APPLICABLE, f"not scored for {ev.device_class} devices")
    if ev.reported_at is None or ev.enabled is None:
        return result(Outcome.NO_DATA, "no screen-lock record in inventory")
    age = now - ev.reported_at
    if age > rule.freshness:
        return result(Outcome.STALE, f"last inventory {age.days} days ago")
    if not ev.enabled:
        return result(Outcome.FAIL, "screen lock is not enabled")
    if ev.os == "macos" and rule.macos_enabled_only:
        return result(Outcome.PASS, "screen lock is enabled")
    if ev.timeout_secs is None:
        return result(Outcome.NO_DATA, "screen lock enabled but no timeout reported")
    if ev.timeout_secs <= 0 or ev.timeout_secs > rule.max_timeout_secs:
        return result(
            Outcome.FAIL,
            f"timeout {ev.timeout_secs}s is outside 1 to {rule.max_timeout_secs}s",
        )
    return result(Outcome.PASS, f"timeout {ev.timeout_secs}s")
```

### A typical endpoint: a person answers an approval

```python
# itops_console/app.py (excerpt)
from typing import Annotated, Literal

from fastapi import Form, HTTPException, Request
from fastapi.responses import HTMLResponse

from itops_console.mcp.approvals import approvals
from itops_console.security import require_csrf, require_same_origin
from itops_console.transcript import transcript


@app.post("/approvals/{approval_id}", response_class=HTMLResponse)
async def answer_approval(
    request: Request,
    approval_id: str,
    action: Annotated[Literal["approve", "decline", "cancel"], Form()],
    csrf_token: Annotated[str, Form()],
) -> HTMLResponse:
    """The only code path that can approve an action, and only a person drives it."""
    require_same_origin(request)
    require_csrf(request, csrf_token)

    pending = approvals.get(approval_id)
    if pending is None:
        raise HTTPException(status_code=404, detail="No open approval with that id")

    if action == "approve":
        # Validate against the schema the act server sent; an unticked
        # approve box or a missing field raises 422 and nothing resolves.
        content = pending.validate(await request.form())
        pending.accept(content)
    elif action == "decline":
        pending.decline()
    else:
        pending.cancel()

    transcript.record("approval_answered", approval_id=approval_id, action=action)
    return templates.TemplateResponse(
        request, "approval_answered.html", {"approval": pending, "action": action}
    )
```

### A typical test: boundaries, freshness and device class

```python
# tests/unit/test_screen_lock.py
from dataclasses import replace
from datetime import datetime, timedelta, timezone

import pytest

from itops_console.baseline.rules import (
    Outcome,
    ScreenLockEvidence,
    ScreenLockRule,
    evaluate_screen_lock,
)

NOW = datetime(2026, 11, 17, 12, 0, tzinfo=timezone.utc)
OFFICIAL = ScreenLockRule(
    max_timeout_secs=1800,
    freshness=timedelta(days=30),
    macos_enabled_only=True,
    applies_to=frozenset({"personal_productivity"}),
)
LOCAL = replace(OFFICIAL, max_timeout_secs=900)


def evidence(**changes: object) -> ScreenLockEvidence:
    base = ScreenLockEvidence(
        os="windows",
        device_class="personal_productivity",
        enabled=True,
        timeout_secs=900,
        reported_at=NOW - timedelta(days=1),
    )
    return replace(base, **changes)


@pytest.mark.parametrize(
    ("timeout", "expected"),
    [(0, Outcome.FAIL), (1, Outcome.PASS), (1800, Outcome.PASS), (1801, Outcome.FAIL)],
)
def test_windows_timeout_boundaries(timeout: int, expected: Outcome) -> None:
    assert evaluate_screen_lock(evidence(timeout_secs=timeout), OFFICIAL, NOW).outcome is expected


def test_twenty_minutes_passes_official_but_fails_local() -> None:
    ev = evidence(timeout_secs=1200)
    assert evaluate_screen_lock(ev, OFFICIAL, NOW).outcome is Outcome.PASS
    assert evaluate_screen_lock(ev, LOCAL, NOW).outcome is Outcome.FAIL


def test_freshness_window_is_inclusive_of_day_30() -> None:
    on_day_30 = evidence(reported_at=NOW - timedelta(days=30))
    on_day_31 = evidence(reported_at=NOW - timedelta(days=31))
    assert evaluate_screen_lock(on_day_30, OFFICIAL, NOW).outcome is Outcome.PASS
    assert evaluate_screen_lock(on_day_31, OFFICIAL, NOW).outcome is Outcome.STALE


def test_shared_device_is_not_scored_on_screen_lock() -> None:
    ev = evidence(device_class="shared", enabled=False)
    assert evaluate_screen_lock(ev, OFFICIAL, NOW).outcome is Outcome.NOT_APPLICABLE


def test_macos_judged_on_enabled_only_and_flagged_unreliable() -> None:
    ev = evidence(os="macos", timeout_secs=3600)
    result = evaluate_screen_lock(ev, OFFICIAL, NOW, unreliable_os=frozenset({"macos"}))
    assert result.outcome is Outcome.PASS
    assert result.source_reliable is False
```
