# Hermes Agent

**Description:** Run [Hermes Agent](https://github.com/NousResearch/hermes-agent) inside a workflow with a custom ephemeral system prompt, an optional skills directory, and a configurable OpenAI-compatible provider. Defaults to `ollama-cloud` with `glm-5.1:cloud`.

The action bundles a small Python runner (`actions/hermes/runner`) that wires GitHub Actions inputs into `HERMES_RUNNER_*` environment variables, invokes `AIAgent.run_conversation`, and writes the final response, a truncated `response` output, and a full JSON message trajectory to disk.

**Inputs:**
- `system_prompt_file`: Path (relative to `$GITHUB_WORKSPACE`) to a Markdown file holding the agent's ephemeral system prompt (required).
- `user_message`: Message passed to the agent. Prefix with `@` to load from a file, e.g. `@${{ runner.temp }}/diff.patch` (required).
- `api_key`: Provider API key. Wired into both `OPENAI_API_KEY` and `HERMES_RUNNER_API_KEY` (required).
- `skills_dir`: Directory of skills — each subdirectory should contain a `SKILL.md`. Copied into `$HERMES_HOME/skills/` before the run (default: `""`).
- `model`: Model identifier. When `base_url` is set, Hermes passes this verbatim to the endpoint's `/chat/completions`, so do **not** include a provider prefix for Ollama Cloud — just the tag, e.g. `glm-5.1:cloud` (default: `glm-5.1:cloud`).
- `provider`: Hermes inference provider identifier (default: `ollama-cloud`).
- `base_url`: OpenAI-compatible base URL for the provider (default: `https://ollama.com/v1`).
- `extra_env`: Multi-line `KEY=VALUE` pairs exported before the run. Escape hatch for provider-specific env vars (default: `""`).
- `max_iterations`: Cap on Hermes tool-call iterations per conversation (default: `40`).
- `enabled_toolsets`: Comma-separated toolsets to whitelist; empty means Hermes default (default: `""`).
- `disabled_toolsets`: Comma-separated toolsets to blacklist (default: `""`).
- `output_file`: If set, the agent's final response is also written to this workspace-relative path (default: `""`).
- `quiet_mode`: Pass `quiet_mode=True` to `AIAgent`. Set to `"false"` to see iteration-by-iteration agent output in the step log (default: `"true"`).
- `python_version`: Python version for `setup-python` (default: `"3.12"`).
- `uv_version`: Version spec for `setup-uv` (default: `0.10.x`).

**Outputs:**
- `response`: Final agent response, truncated to ~900 KB to stay under GitHub's step-output size limits. For the full response use `response_file`.
- `response_file`: Absolute path to the full response on disk (`$RUNNER_TEMP/hermes-response.txt`).
- `trajectory_file`: Absolute path to the JSON message trajectory (`$RUNNER_TEMP/hermes-trajectory.json`).

**Example Usage (PR review):**
```yaml
name: Hermes Code Review

on:
  pull_request:
    types: [opened, synchronize, reopened]

permissions:
  contents: read
  pull-requests: write

jobs:
  review:
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Fetch PR diff
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh pr diff "${{ github.event.pull_request.number }}" > "${{ runner.temp }}/pr.patch"

      - id: hermes
        uses: JasperHG90/toiminnot/actions/hermes@main # or pin to a tag, e.g. @v0.1.0
        with:
          system_prompt_file: .github/prompts/review-system.md
          skills_dir: .claude/skills
          user_message: '@${{ runner.temp }}/pr.patch'
          api_key: ${{ secrets.OLLAMA_API_KEY }}
          max_iterations: '12'
          quiet_mode: 'false'

      - name: Post review comment
        env:
          GH_TOKEN: ${{ github.token }}
        run: gh pr review "${{ github.event.pull_request.number }}" --comment -F "${{ steps.hermes.outputs.response_file }}"
```

**Using Gemini (OpenAI-compatible endpoint):**
```yaml
- uses: JasperHG90/toiminnot/actions/hermes@main
  with:
    system_prompt_file: .github/prompts/review-system.md
    user_message: '@${{ runner.temp }}/task.md'
    api_key: ${{ secrets.GEMINI_API_KEY }}
    provider: gemini
    model: gemini-3-flash-preview
    base_url: https://generativelanguage.googleapis.com/v1beta/openai/
```

**Notes:**
- The only supported providers are **Ollama Cloud** (default) and **Gemini** via its OpenAI-compatible endpoint. Both are configured with an `api_key`, a `provider`, a `model`, and a `base_url`.
- The runner is installed via `uv sync` against `actions/hermes/runner/pyproject.toml`; the environment is cached by `astral-sh/setup-uv` keyed on the checked-out lockfile.
- `skills_dir` expects a layout like `my-skills/<skill-name>/SKILL.md`. Each `<skill-name>/` subdirectory is copied into `$HERMES_HOME/skills/` before the run; anything outside that structure is ignored.
- `extra_env` is appended line-by-line to `$GITHUB_ENV`, so the values are available to any subsequent step in the job — not just to Hermes.
- The `response` step-output is truncated at ~900 KB. Always prefer `response_file`/`trajectory_file` when the body might be large or binary.
