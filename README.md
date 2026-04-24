# toiminnot

This repository contains reusable GitHub Action templates for common CI/CD tasks.

## Storing actions

Actions can be stored under `/actions`. They must be stored in their own folder, and be called `action.yaml` (or you may use a Dockerfile action).

## Storing workflows

Workflows must be stored under the `.github/workflows` folder (this is a GH requirement). They must be stored in a YAML file and *cannot* be placed in a subfolder.

## Documentation

For *actions*, please place a README in each action folder.

For *workflows*, please amend the README in `.github/workflows`.

## Available actions

| Action | Purpose |
| --- | --- |
| [`actions/git_version`](actions/git_version) | Retrieve a PEP-compatible version string from git history and tags. |
| [`actions/hermes`](actions/hermes) | Run [Hermes Agent](https://github.com/NousResearch/hermes-agent) with a custom system prompt, skills directory, and OpenAI-compatible provider. |
| [`actions/pre_commit`](actions/pre_commit) | Run pre-commit hooks against the repository. |
| [`actions/prek`](actions/prek) | Run [prek](https://github.com/memex-project/prek) formatting/linting. |
| [`actions/pytest`](actions/pytest) | Run pytest with uv, with optional markers and private-index credentials. |
| [`actions/twine_release`](actions/twine_release) | Publish a Python package to PyPI via twine. |
| [`actions/uv_build`](actions/uv_build) | Build a Python package with uv. |

## Consuming an action

Pin to a tag in production — `@main` is a moving target:

```yaml
- uses: JasperHG90/toiminnot/actions/<name>@v0.1.0
  with:
    ...
```
