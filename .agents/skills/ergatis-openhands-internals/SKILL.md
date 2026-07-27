---
name: ergatis-openhands-internals
description: >
  Working guide for modifying this OpenHands fork (github.com/YugmaF/OpenHands) as the
  second coding engine of the Ergatis platform. You MUST consult this skill whenever you
  change anything in this repository — agent loop behavior, conversation orchestration,
  hooks, sandbox services, events, LLM settings, or the frontend — and whenever the task
  mentions the Ergatis verifier loop, acceptance tests, per-stage models, the quality
  loop, or wiring OpenHands to the Ergatis platform. It defines which layer a change
  belongs to, the Ergatis product contract you must not break, and what is current
  state vs. future MVP target.
triggers:
  - ergatis
  - verifier
  - acceptance tests
  - agent loop
  - app_server
  - conversation
  - sandbox
  - quality loop
---

# Ergatis × OpenHands Internals

## Why this fork exists

This is Ergatis's fork of OpenHands. Ergatis (the parent repo, two levels up at the
git-submodule mount `runners/openhands/`) is a multi-tenant SaaS that runs coding agents
against customer repos, metered by credits. OpenHands is being wired in as the **second
engine** behind Ergatis's frozen `CodingEnginePort`. Platform-side code (the adapter,
tenancy, budgets, APIs) lives in the parent repo's `platform/` — **never here**. What
lives HERE is agent-loop **orchestration**: the changes to how OpenHands runs a task.
Fork workflow: feature branch in this repo → verify → merge to this fork's `main`;
the parent repo bumps its submodule pointer as a separate commit. Follow the root
`AGENTS.md` (install pre-commit hooks first; pre-commit + lint must pass).

## The three layers — decide where your change belongs BEFORE coding

| Layer | Where | What it is | Modifiable? |
|---|---|---|---|
| **Application server** | `openhands/app_server/` (this repo) | FastAPI app: conversation lifecycle (`app_conversation/`), events (`event/`, `event_callback/`), sandbox launch (`sandbox/` — docker/remote/process services), settings & LLM profiles (`settings/llm_profiles.py`), integrations, MCP | ✅ yes — primary place for orchestration changes |
| **Agent server** (in-sandbox) | external pkg `openhands-agent-server==1.35.0` | the server that runs inside a sandbox and executes runs | ⚠️ pinned PyPI dep — not in this repo |
| **Agent loop / SDK** | external pkgs `openhands-sdk==1.35.0`, `openhands-tools==1.35.0` | the actual agent loop, LLM class (litellm-based), tools | ⚠️ pinned PyPI dep — not in this repo |

Consequences:

- **The agent loop itself is NOT in this repository.** `pyproject.toml` exact-pins the
  three `openhands-*` packages (no path/git overrides). If a change truly requires
  editing loop internals, that is a *new decision* — forking agent-sdk or vendoring —
  which must be surfaced to the user, not made silently. Prefer expressing loop changes
  as **orchestration in `app_server`**: `app_conversation/app_conversation_start_task_service.py`
  (task start flows), `app_conversation/hook_loader.py` and `skill_loader.py` (extension
  seams), `event_callback/` (react to run events).
- Version bumps of the pinned SDK packages are explicit and deliberate (see the
  pyproject comments) — never float them.

## The Ergatis product contract — never break these

The user experience is fixed and lives in the Ergatis platform, not here:

1. **The Task Console is the only user entry point.** Users submit the prompt, watch the
   live timeline, approve/deny gated actions, and steer — all through Ergatis's
   `/api/v1/tasks/{id}` API family (events poll, `messages`, `decisions`). Any
   interaction your change introduces must surface through that channel (via the
   platform's `CodingEnginePort` adapter), never through a separate UI a tenant user
   would need.
2. **The final deliverable is the run recording.** A task ends with a screen recording
   uploaded to the platform (`POST /api/v1/ingest/tasks/{id}/recording` with a per-task
   capability token) that the user watches to verify everything was done as asked. Loop
   changes must keep the run observable and the recording pipeline intact — a verifier
   iteration that happens invisibly off-screen defeats the product.
3. **All model traffic goes through the injected endpoint.** The run receives an
   OpenAI-compatible `base_url` + per-task `api_key` (Ergatis's LiteLLM gateway; virtual
   key with budget). Never call a provider or openrouter.ai directly, never embed
   provider keys, never log the key. Stage models are selected by **model name** only
   (gateway aliases).
4. **Budgets end runs.** The platform's kill-switch can revoke the key and stop the
   sandbox mid-run; loop code must tolerate hard stops without corrupting state.

Platform-side details: `.agents/skills/ergatis-platform-integration/` in the parent
repo — especially `references/engine-adapters.md` and `references/llm-gateway.md`.

## Current state vs. TARGET (do not confuse them)

**Current (2026-07):** the Ergatis platform SaaS is nearly done (tasks, budgets,
kill-switch, Task Console, recording pipeline — live with the Claude Code engine). The
OpenHands platform adapter does not exist yet; this fork is unmodified upstream. The
work is *beginning* here.

**Target MVP — Phase 1 (near-term): the pre-PR acceptance-verifier loop**

1. At task start, generate acceptance tests from the task's acceptance criteria —
   before the agent codes, never from its diff — and **freeze them** as an independent
   oracle.
2. **Anti-gaming:** the coding agent must not be able to edit or delete the acceptance
   suite (outside the writable tree, or diff-guarded).
3. **Deterministic verdicts:** pass/fail = the real per-language test-runner exit code
   (`mvn test` / `pytest` / `go test` / `npm test`). The LLM only generates tests and
   judges genuinely unmechanizable criteria.
4. **Bounded fix loop:** fail + iterations left → resume the agent with minimal failure
   feedback (failing test, expected vs actual, repro — not raw logs); pass → open the
   PR; exhausted → hand to a human via the platform (`NEEDS_ATTENTION`), never silent
   success. Platform vocabulary already exists for this: `TaskStatus.VERIFYING`,
   `EngineRunSpec.maxIterations`.

**Target MVP — Phase 2 (deferred, do NOT build yet):** post-PR quality loop — PR-review
agent runs on PR; the **org's existing Sonar gate** (coverage/security/quality) is
consumed via its webhook/API — never bundle a Sonar server into the loop or sandbox —
and its verdict decides re-enter-the-loop (failed, with findings) vs stage done
(passed); Java unit-test tool for coverage; human merge → CD to perf/UAT. The review
tool, the Java test tool, and the org Sonar server are **org infrastructure outside
both repos** — confirm location with the user before wiring. The acceptance oracle
(Phase 1, per-task, never ships) and coverage tests (Phase 2, ship with code) are
different artifacts — never conflate them.

**Per-stage models (target):** stages pick price/quality via gateway model aliases.
Selection is **deterministic, config-driven** — a fixed stage→alias mapping, never a
runtime/LLM choice, never boxed to one vendor model. Optimization target: **fewer
issues per task** (minimize rework first, cost second); put model quality where the
oracle is created (planning, test generation). ⚠️ Prerequisite in the platform, not
here: today Ergatis mints the virtual key scoped to ONE model — the mint must be
widened to the stage-alias set first.

## Dev workflow (from the root `AGENTS.md` — follow it fully)

```bash
make install-pre-commit-hooks       # ALWAYS first, before any change
make build                          # full setup (backend + frontend), when needed

# after backend changes:
pre-commit run --config ./dev_config/python/.pre-commit-config.yaml

# after frontend changes:
cd frontend && npm run lint:fix && npm run build
```

Pre-commit and lint MUST pass before pushing. Prefer `git add <file>` over `git add .`;
pin third-party GitHub Actions to full SHAs. Local full-app run recipe and its
troubleshooting notes are in the root `AGENTS.md`.

## Common failure modes

| Mistake | Correct behavior |
|---|---|
| Hunting for the agent loop in this repo | It's in the pinned `openhands-sdk` package — orchestrate from `app_server`, or escalate the SDK-fork decision to the user |
| Building UI here for tenant-facing interaction | Tenant interaction goes through the Ergatis Task Console; this fork's frontend is not the tenant surface |
| Calling providers/OpenRouter directly from loop code | Only the injected `base_url` + per-task key; models by alias name |
| Implementing Phase 2 (review agent / Sonar) now | Phase 1 verifier loop first; Phase 2 is explicitly deferred |
| Letting the agent touch the acceptance suite | The oracle is frozen and out of the agent's reach — that is the point |
| Treating a failed gate/verdict as terminal | Failures re-enter the bounded loop as feedback; only exhaustion goes to a human |
| Floating or upgrading `openhands-*` pins as a side effect | Exact pins are deliberate; bump explicitly and alone |
| Committing to `main` of the fork | Feature branch → verify → merge; parent repo bumps the submodule separately |
