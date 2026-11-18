# Egress Ward

Pi editor configuration for this repository.

## Project Shape

This is a monorepo scaffold for NemoClaw agent projects with OpenShell sandbox
policies. Each agent or service is isolated under its own component directory
(`agents/<name>/` or `services/<name>/`) and owns its own manifest, policy,
and tests, sharing one workspace-wide `uv.lock`.

## Core Rule

One owner per top-level component directory. A worktree building agent `X`
touches only its own component directory: `agents/business-agent`,
`agents/security-agent`, `agents/hello-agent` under `agents/`, and
`services/ingestion`, `services/processing`, `services/dashboard` under
`services/`.

`contracts/` and the root `uv.lock` (plus root `pyproject.toml` and the root
`justfile`) are jointly-owned, review-gated shared surfaces. Do not modify them while implementing a specific component
unless the user explicitly asks for a shared change. If `uv.lock` conflicts,
see CLAUDE.md's "uv.lock conflicts" section — regenerate it with `just
relock`, never hand-merge it.

## Commands

Use the repo recipes when available:

```bash
just a <name> check     # one agent
just s <name> check     # one service
just each check         # everything, across agents/ and services/
```

If `just` is not installed, run the equivalent commands inside the target
project:

```bash
uv sync
uv run ruff check .
uv run ruff format --check .
uv run mypy src
uv run pytest
```

Run the relevant check before calling work done.

## Pi Model

For this repository, Pi should use the local LiteLLM OpenAI-compatible
endpoint on the `hack` host (model `Qwen3.6-27B-FP8`). Do not fall back to a
cloud provider if the LiteLLM endpoint or Qwen model is unavailable; report the
local inference failure instead.

## Skills

Available repo skills (canonical copies under `libs/skills/`):

- `nemoclaw-fanout`: reason about subagent fan-out and concurrency limits.
- `openshell-data-boundary`: reason about sandbox filesystem boundaries.
- `openshell-egress-audit`: inspect and tighten OpenShell network egress.
- `spark-inference`: inspect or switch local Dell Pro Precision GB10 inference routes.

Treat `libs/skills/` as canonical. If a skill needs to change, edit the shared
copy there rather than forking skill text.

## Development Guidance

- Start by reading the target agent's existing files before editing.
- Make the smallest correct change.
- Preserve per-agent isolation and avoid central registries.
- Do not add broad network policy exceptions; widen sandbox policy only from
  observed denials.
- Prefer local inference. Do not add cloud fallback unless explicitly requested.
