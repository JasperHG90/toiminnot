# Hermes Interactive

A GitHub composite action that enables `@hermes` mentions in issue and PR comments. When a user comments `@hermes <prompt>`, this action gathers context, runs the [Hermes Agent](../hermes/), and posts the response back as a comment.

## How it works

```
@hermes mention in comment
  → eyes reaction (acknowledges)
  → extract prompt from comment
  → gather context (issue/PR body, comments, PR diff)
  → run Hermes agent with context + prompt
  → push changes to branch (optional)
  → post response as comment
  → rocket reaction (done)
```

## Usage

Create a workflow in your repository at `.github/workflows/hermes-interactive.yaml`:

```yaml
name: Hermes Interactive

on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]

permissions:
  contents: read
  issues: write
  pull-requests: write

concurrency:
  group: hermes-interactive-${{ github.event.issue.number || github.event.pull_request.number }}
  cancel-in-progress: false

jobs:
  respond:
    if: >-
      contains(github.event.comment.body, '@hermes') &&
      github.event.comment.user.login != 'github-actions[bot]' &&
      contains(fromJSON('["MEMBER","OWNER","COLLABORATOR"]'),
               github.event.comment.author_association)
    runs-on: ubuntu-latest
    timeout-minutes: 20
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - uses: JasperHG90/toiminnot/actions/hermes-interactive@main
        with:
          api-key: ${{ secrets.OLLAMA_API_KEY }}
          github-token: ${{ github.token }}
```

## Inputs

| Input | Required | Default | Description |
|-------|----------|---------|-------------|
| `trigger-phrase` | no | `@hermes` | The @-mention phrase that activates the action |
| `system-prompt-file` | no | `''` | Path to custom system prompt (absolute or relative to `$GITHUB_WORKSPACE`). Falls back to built-in prompt if empty |
| `api-key` | **yes** | — | Provider API key |
| `github-token` | **yes** | — | GitHub token for reactions and comments |
| `model` | no | `glm-5.1:cloud` | Model identifier |
| `provider` | no | `ollama-cloud` | Hermes inference provider |
| `base-url` | no | `https://ollama.com/v1` | OpenAI-compatible base URL |
| `skills-dir` | no | `''` | Skills directory (`<name>/SKILL.md` subdirs) |
| `max-iterations` | no | `20` | Tool-call iteration cap |
| `extra-env` | no | `''` | Multi-line `KEY=VALUE` pairs exported before the run |
| `include-pr-diff` | no | `'true'` | Include PR diff in context |
| `include-comments` | no | `'true'` | Include prior comments in context |
| `max-comment-count` | no | `'20'` | Max prior comments to include |
| `enabled-toolsets` | no | `''` | Comma-separated toolsets to whitelist |
| `disabled-toolsets` | no | `''` | Comma-separated toolsets to blacklist |
| `quiet-mode` | no | `'true'` | Suppress per-iteration agent output in logs |
| `push-changes` | no | `'false'` | `'true'` to push workspace changes to a branch; `'pr'` to also create a PR |
| `branch-prefix` | no | `'hermes/'` | Prefix for branches created when push-changes is enabled |

## Outputs

| Output | Description |
|--------|-------------|
| `response` | Final agent response text |
| `response-file` | Absolute path to the full response file |
| `comment-id` | ID of the posted GitHub comment |
| `branch` | Branch name if changes were pushed (empty otherwise) |
| `pr-url` | URL of the created PR if `push-changes: 'pr'` (empty otherwise) |
| `skipped` | `'true'` if the action decided not to run (wrong trigger, bot comment) |

## Supported triggers

| Event | Description |
|-------|-------------|
| `issue_comment` | Comments on issues and PRs (main conversation thread) |
| `pull_request_review_comment` | Inline code review comments on PRs |

For PR review comments, the action includes the file path and line number in the context so the agent knows which code the user is asking about.

## Context assembly

The action automatically gathers context for the agent:

1. **Issue/PR metadata** — title, body, labels, author, state
2. **Review comment location** — file path and line number (PR review comments only)
3. **Conversation history** — up to `max-comment-count` prior comments (disable with `include-comments: 'false'`)
4. **PR diff** — when triggered from a PR comment, includes the diff truncated to 50KB (disable with `include-pr-diff: 'false'`)

## Security

- **Authorization** is handled in the workflow `if:` condition via `author_association`. Only members, owners, and collaborators can trigger the action by default.
- **Bot loop prevention** — comments from `github-actions[bot]` are ignored.
- **No `pull_request_target`** — uses `issue_comment` and `pull_request_review_comment`, both of which run in the base branch context.
- **Input sanitization** — the extracted prompt is written to a file, never shell-interpolated.

## Custom system prompt

Override the built-in prompt by pointing to your own:

```yaml
- uses: JasperHG90/toiminnot/actions/hermes-interactive@main
  with:
    api-key: ${{ secrets.OLLAMA_API_KEY }}
    github-token: ${{ github.token }}
    system-prompt-file: .github/prompts/my-hermes-system.md
```

## Trigger phrase

Change the trigger phrase to avoid conflicts with other bots:

```yaml
- uses: JasperHG90/toiminnot/actions/hermes-interactive@main
  with:
    trigger-phrase: '@mybot'
    api-key: ${{ secrets.API_KEY }}
    github-token: ${{ github.token }}
```

## Pushing changes

If the agent modifies files during its run, you can have the action commit and push them:

```yaml
- uses: JasperHG90/toiminnot/actions/hermes-interactive@main
  with:
    api-key: ${{ secrets.OLLAMA_API_KEY }}
    github-token: ${{ github.token }}
    push-changes: 'pr'  # 'true' = push branch only, 'pr' = push + open PR
```

When `push-changes` is enabled:
- The action checks for workspace changes after the agent finishes
- If changes exist: creates a `hermes/{number}-{timestamp}` branch, commits, and pushes
- With `'pr'`: also opens a pull request targeting the default branch
- The response comment includes a link to the branch or PR
- Requires `contents: write` permission in the workflow

## Quoted mentions

The action ignores `@hermes` mentions inside blockquotes (`> @hermes ...`) to prevent re-triggering when someone quotes a previous comment.
