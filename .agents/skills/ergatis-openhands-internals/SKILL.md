---
name: ergatis-openhands-internals
description: >
  Working guide for the OpenHands fork (github.com/YugmaF/OpenHands) as it relates to the
  Ergatis platform. You MUST consult this skill whenever you change anything in this
  repository, or when a task mentions the OpenHands agent loop, the Ergatis verifier loop,
  acceptance tests, hooks, the quality loop, or wiring OpenHands to Ergatis. It states what
  this fork is (and is not) on the run path, where loop orchestration actually lands, and
  the product contract you must not break. Corrected 2026-08-17: the previous version said
  orchestration lands in this repo's app_server — it does not.
triggers:
  - ergatis
  - verifier
  - acceptance tests
  - agent loop
  - app_server
  - agent-server
  - hooks
  - quality loop
---

# Ergatis × OpenHands Internals

## What this fork is — and what runs in production

Ergatis (parent repo, two levels up; this is the submodule `runners/openhands/`) is a
multi-tenant SaaS that runs coding agents against customer repos, metered by credits.
OpenHands is its second engine behind the frozen `CodingEnginePort`.

**Production does not run this repository.** The platform adapter
(`platform/src/main/java/com/ergatis/platform/engine/adapter/openhands/`) runs the upstream
**`openhands-agent-server`** binary from Ergatis's own sandbox image
(`runners/openhands-sandbox/Dockerfile`, `FROM ghcr.io/openhands/agent-server:1.37.1-python`)
and drives it over REST. It **deliberately bypasses this repo's `app_server`** (a second
control plane — DB, users, settings, sandbox lifecycle — that would sit inside ours and break
`stop()` and the budget kill-switch). See `docs/25-openhands-architecture.md` in the parent.

| Layer | Where | On the Ergatis run path? | Modifiable? |
|---|---|---|---|
| Application server (`openhands/app_server/`, frontend) | this repo | **No** | yes, but pointless for Ergatis |
| Agent server (in-sandbox REST) | PyPI `openhands-agent-server==1.37.1`, inside our image | **Yes** | pinned dep — not here |
| Agent loop / tools / SDK | PyPI `openhands-sdk==1.37.1`, `openhands-tools==1.37.1` | **Yes** | pinned dep — not here |
| **Conversation start body** (`AgentServerClient.body()`) | parent `platform/…/engine/adapter/openhands/` | **Yes** | ✅ primary orchestration seam |
| **Sandbox image** (hooks, verifier, guards, MCP servers, recorder) | parent `runners/openhands-sandbox/` | **Yes** | ✅ primary orchestration seam |

Consequences:

- **This fork carries no code changes and should stay that way** unless an SDK change is
  genuinely unavoidable. That is a **new decision to surface to the user** (fork
  `agent-sdk` / vendor a package / upstream PR), never a silent step. Keep the fork synced
  with `upstream/main` so a first real change lands on a current base; last sync
  2026-07-27.
- To **read the loop source**, do not hunt in this repo — clone the SDK at the pinned tag:
  `git clone --depth 1 --branch v1.37.1 https://github.com/OpenHands/agent-sdk`
  (`openhands-sdk`, `openhands-tools`, `openhands-agent-server` in one repo). The image
  ships the SDK as a compiled binary; there is no `.py` to read inside it.
- **Where loop changes land**: the start body + the sandbox image, in the parent repo,
  under the parent skill `.agents/skills/ergatis-platform-integration/` (read
  `references/engine-adapters.md`, sections *The Ergatis loop around agent-server* and
  *Start-body contract: prove every field landed*). The analysis behind this —
  weaknesses of the 1.37.1 loop for industrial repos and the recommended sequence — is
  `docs/26-openhands-agent-loop-findings.md`.
- Version bumps of the pinned `openhands-*` packages are explicit and deliberate — never
  float them, and re-run the parent's live-server contract test on every bump.

## The loop primitives already in SDK 1.37.1 (use these, do not reinvent)

All accepted by `POST /api/conversations` on agent-server:

- `hook_config` — Claude-Code-compatible **command hooks** (JSON on stdin; `exit 2` or
  `{"decision":"deny","reason":…}` blocks; `exit 1` does **not** block). Events:
  `PreToolUse, PostToolUse, UserPromptSubmit, SessionStart, SessionEnd, Stop`. The **Stop
  hook** fires on every finish (finish tool *and* prose reply) and a denial re-arms the loop
  with the reason injected — this is the bounded verify→fix primitive. Hooks are plain
  subprocesses in the sandbox; they call no model.
- `confirmation_policy` (`AlwaysConfirm | NeverConfirm | ConfirmRisky`) + `security_analyzer`,
  answered externally via `POST …/events/respond_to_confirmation` — per batch, accept/reject-all.
- `agent.agent_context.load_project_skills=true` — loads the repo's `AGENTS.md`/`CLAUDE.md`/
  `.openhands/skills` (default **false**).
- `agent.condenser` (`LLMSummarizingCondenser` with `max_size` and, importantly, `max_tokens`
  and its own `llm`), `agent_launch_additions.system_message_suffix_append`, `agent.mcp_config`
  (stdio MCP servers), `stuck_detection`, `agent.critic` (experimental; do not build the gate
  on it — its in-loop `git_patch` is always `None` and it misses prose finishes).

**Fail closed.** Nested server models drop unknown fields silently and still answer `201`.
Every gating field must be **echo-asserted** via `GET /api/conversations/{id}` right after
start, and the hooks proven armed at runtime (SessionStart marker file) before any verdict
is trusted. A dropped `hook_config` is a run that *looks* verified.

## The Ergatis product contract — never break these

1. **The Task Console is the only user entry point** — prompt, live timeline, approvals,
   steering all go through Ergatis's `/api/v1/tasks/{id}` API family via the adapter. Never
   a separate UI a tenant user would need (this repo's frontend is not the tenant surface).
2. **The final deliverable is the run recording** (uploaded with a per-task capability
   token). Loop changes must keep the run observable — a verifier iteration that happens
   invisibly defeats the product; verify attempts are posted to the checks ingest so the
   timeline and recording show them.
3. **All model traffic goes through the injected endpoint** — OpenAI-compatible `base_url`
   + per-task `api_key` (Ergatis's LiteLLM gateway, budget-capped virtual key). Never a
   provider or openrouter.ai directly, never embed keys, never log the key. Stage models are
   chosen by **alias name** only; the key already covers the profile's whole alias set.
4. **Budgets end runs.** The kill-switch revokes the key and stops the sandbox mid-run
   (`docker stop -t 180`); everything you add (hooks, verifier) must tolerate SIGTERM
   without corrupting state — write verdicts `tmp`+rename, flush in the wrapper's `cleanup()`.

## Current state vs. TARGET (do not confuse them)

**Current (2026-08):** platform SaaS live with both engines; OpenHands adapter shipped
(start, events by webhook, steer, resume-after-cap, publish/PR, recording). The start body
sends **only** workspace, agent{tools, llm, condenser(max_size)}, initial_message, max_iterations
— no hooks, no confirmation policy, no project skills, no critic. `FINISHED → SUCCEEDED`
unconditionally; the evidence ledger is display-only; `planMode` is ignored on this engine.

**Target — Phase 1 (near-term): the pre-PR acceptance-verifier loop**

1. Generate acceptance tests from the task's criteria **before** the agent codes (separate
   stage, `VERIFICATION`/`PLANNING` alias, different model than the coder) and **freeze**
   them (sha recorded) as an independent oracle. Never let the coding agent propose them.
2. **Anti-gaming**: suite lives outside `/workspace/project` (root-owned, 0555 — agent-server
   runs as uid 10001), copied in at verify time; `PreToolUse` deny on protected paths;
   diff-guard against `.git/ergatis_base_sha`.
3. **Deterministic verdicts**: the Stop hook runs the real runner (`mvn test` / `pytest` /
   `go test` / `npm test`); exit code decides. LLM judges only genuinely unmechanisable
   criteria.
4. **Bounded fix loop**: fail + attempts left → deny with minimal feedback (failing test,
   expected vs actual, repro — never raw logs); pass → allow → PR; exhausted → allow but
   `verdict=exhausted` → platform `NEEDS_ATTENTION`, never silent success. Platform
   vocabulary exists: `TaskStatus.VERIFYING` (unused today), phase `verifying` (reserved),
   `EngineRunSpec.maxIterations`.

**Target — Phase 2 (deferred, do NOT build yet):** post-PR quality loop — PR-review agent,
the **org's existing Sonar gate** consumed via webhook/API (never bundled), Java unit-test
tool for coverage. Acceptance oracle (per-task, never ships) ≠ coverage tests (ship) —
never conflate.

## Dev workflow for this repo (only if you really must change it)

```bash
make install-pre-commit-hooks       # ALWAYS first
pre-commit run --config ./dev_config/python/.pre-commit-config.yaml   # after backend changes
```

Feature branch in this repo → verify → merge to the fork's `main`; the parent bumps the
submodule pointer as its own commit. Prefer `git add <file>` over `git add .`.

## Common failure modes

| Mistake | Correct behavior |
|---|---|
| Hunting for the agent loop in this repo | It's in the pinned `openhands-sdk` — clone `agent-sdk` at `v1.37.1` to read it |
| Putting orchestration in `app_server` / `hook_loader.py` / `skill_loader.py` here | Dead in Ergatis's topology — the adapter never runs `app_server`. Use the start body + sandbox image in the parent repo |
| Sending a new start-body field and trusting the `201` | Echo-assert via `GET /api/conversations/{id}`; fail the start if it did not land |
| Building the gate on `critic` | Use the Stop hook; critic misses prose finishes and gets `git_patch=None` |
| Letting the coding agent write or edit the acceptance suite | Frozen, outside its writable tree, generated by a separate stage |
| Treating a failed verdict as terminal | Failures re-enter the bounded loop; only exhaustion goes to a human |
| Building UI here for tenant interaction | Task Console only |
| Floating `openhands-*` pins | Exact pins; bump explicitly, alone, with the contract test |
| Committing to the fork's `main` | Feature branch → verify → merge |
