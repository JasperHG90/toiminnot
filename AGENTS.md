# AGENTS.md

Guidance for LLM agents contributing to this repository. Human contributors should follow the same conventions.

## Repository purpose

`toiminnot` is a library of reusable GitHub Actions. Every commit here is consumed elsewhere, so treat backwards compatibility as a first-class concern: renaming inputs, removing defaults, or changing output semantics is a breaking change and should go out on a new tag.

## Directory layout

```
.
├── actions/
│   └── <action_name>/
│       ├── action.yaml          # REQUIRED. Entry point. Never name it action.yml.
│       ├── README.md            # REQUIRED. See "README contract" below.
│       ├── Dockerfile           # OPTIONAL. Only for `using: docker` actions.
│       ├── entrypoint.sh        # OPTIONAL. For Dockerfile actions.
│       └── <supporting files>   # OPTIONAL. Source code, scripts, lockfiles.
├── .github/
│   └── workflows/               # Repo-internal CI only. No nested folders (GH limitation).
├── README.md                    # Top-level index of available actions.
├── AGENTS.md                    # This file.
├── pyproject.toml               # Repo-wide ruff config.
└── .pre-commit-config.yaml      # Runs on every commit; do not skip.
```

### Hard rules

- **One action per folder** under `actions/`. The folder name is part of the public API (`JasperHG90/toiminnot/actions/<folder>@<ref>`); pick it carefully and do not rename it casually.
- **The action file MUST be named `action.yaml`** — not `action.yml`. GitHub accepts both, but the repo picks one.
- **Every action folder MUST contain a `README.md`.** Pre-commit will not catch a missing README, so reviewers must.
- **Do not place workflows under `actions/`** and do not place actions under `.github/workflows/`.

## Adding a new action

1. Create `actions/<name>/` with a snake_case name. The name should describe the capability, not the first caller.
2. Write `action.yaml` first. Start from an existing action of the same type (composite vs. Dockerfile) — do not freehand it.
3. Write the README next, before adding supporting code. If the README is hard to write, the interface is probably wrong.
4. Commit and push. Pre-commit must pass locally before pushing.
5. Cut a tag (e.g. `v0.2.0`) when the new action is ready for consumption.

## `action.yaml` conventions

### Metadata

- Always set both `name:` and `description:`. `description:` is what surfaces on the Marketplace and in workflow pickers.
- Keep `description:` to one line. Move prose to the README.

### Inputs

- All input names MUST be `snake_case` (`uv_version`, `python_version`, `repo_path`, `system_prompt_file`). This is enforced by convention, not by tooling — reviewers must catch violations.
- Every input MUST have a `description:`. Include the unit (`seconds`, `KB`), expected format (`comma-separated`, `@file-prefix`), and an example when the shape is not obvious.
- Mark `required: true` only when there is no sensible default. Prefer a sensible default over forcing the caller to pass one.
- All defaults MUST be strings (`default: '3'`, not `default: 3`) — GitHub coerces them to strings anyway and an unquoted value can surprise you for `true/false/yes/no`.

### Outputs

- Output names MUST be `snake_case` as well (`response_file`, `trajectory_file`, `version`).
- Give every step you want to expose a stable `id:` and reference it via `steps.<id>.outputs.<name>`.
- Document outputs in the README with the same rigor as inputs (what, when, format, size limits).
- Large blobs (> ~1 MB) must go to a file path output, not a string output — GitHub truncates step outputs around 1 MB per line.

### Runs

- Use `using: composite` by default. Pick `using: docker` only when you need a pinned toolchain that would be slow or fragile to install from the runner (e.g. `git_version` uses a Dockerfile to avoid installing bash + gh on every runner).
- Composite steps MUST set `shell:` explicitly.
- Composite `run:` blocks MUST start with `set -euo pipefail` when written in bash.
- Reference the action's own files via `${{ github.action_path }}` — never assume the caller's `$GITHUB_WORKSPACE` layout.

### Third-party action pinning

- Pin by major version (`actions/checkout@v4`, `astral-sh/setup-uv@v5`) for GitHub-maintained actions.
- Pin by SHA for any action outside `actions/` or `astral-sh/`. A moving tag on a third-party action is a supply-chain risk.

## README contract (per action)

Every `actions/<name>/README.md` must contain, in this order:

1. `# <Human-readable name>`
2. `**Description:**` — one paragraph. Link to upstream projects where relevant.
3. `**Inputs:**` — bulleted list. Mirror the `action.yaml` exactly. Call out `required`, defaults, and units.
4. `**Outputs:**` — bulleted list (omit only if the action has no outputs).
5. `**Example Usage:**` — at least one fenced `yaml` block showing the minimal happy path. Use `JasperHG90/toiminnot/actions/<name>@main # or pin to a tag, e.g. @v0.1.0` in the `uses:` line.
6. Follow-up sections (auth, provider variants, gotchas) as needed.

Match the tone and formatting of `actions/pytest/README.md` and `actions/git_version/README.md` — they are the reference implementations.

## Python code in actions

Some actions ship a Python runner (currently `hermes`). When you add or modify Python code:

- Put source under `actions/<name>/runner/src/<package>/` and tests under `actions/<name>/runner/tests/`.
- Use `uv` for dependency management. Commit `pyproject.toml` AND `uv.lock` together.
- Python `>= 3.12`. `ruff` formatting and lint must pass (it runs in pre-commit repo-wide).
- `pyright` must pass (also in pre-commit). Prefer explicit types over `Any`.
- Use `pydantic-settings` with an `env_prefix` for action→runner parameter passing, so tests can drive the runner by setting env vars instead of shelling out.

## Workflows (`.github/workflows/`)

- Scope workflows in this repo to testing the actions themselves (lint, smoke tests, release tagging). Do not add unrelated CI here.
- Reference local actions by relative path (`uses: ./actions/<name>`) in test workflows so you exercise the checked-out version, not a published tag.

## Pre-commit

`.pre-commit-config.yaml` runs `check-ast`, `check-toml`, `check-yaml`, trailing-whitespace, end-of-file-fixer, `ruff`, `ruff-format`, and `pyright` on every commit. Never bypass with `--no-verify`. If a hook is failing, fix the underlying issue; if the hook itself is wrong, fix the hook config and land that change separately.

## Commits and PRs

- Use conventional-commit prefixes: `feat(hermes):`, `fix(pytest):`, `docs:`, `chore(ci):`, `refactor:`.
- One logical change per commit. A new action + its README is one commit; tweaking CI is another.
- PR descriptions should show the action being consumed from a downstream repo (a snippet of the caller's workflow) — this is the fastest way to review interface changes.

## Releases

- Tag releases as `vMAJOR.MINOR.PATCH` on `main`.
- Any breaking change to an existing action's inputs/outputs bumps MAJOR. Adding an input with a default is MINOR. Bug fixes / doc changes are PATCH.
- Also move the matching `v<MAJOR>` float tag (e.g. `v0`) to the new release so callers pinning `@v0` track it.

## What not to do

- Don't add actions that merely wrap a single `run:` — callers can inline that themselves.
- Don't introduce shared helper scripts across actions. Each action stays self-contained so it can be vendored or forked independently.
- Don't commit secrets, tokens, or `.env` files. There is no secret management in this repo; actions receive secrets from the caller at runtime.
- Don't remove an action without a deprecation period. Mark it deprecated in the README for at least one release first.
