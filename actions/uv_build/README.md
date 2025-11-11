# Build Python Package with UV

This GitHub Action builds a Python package using [UV](https://github.com/astral-sh/uv) and uploads the resulting distribution files as a workflow artifact.

## Usage

To use this action, create a workflow file (e.g., `.github/workflows/build.yml`) in your repository with the following content:

```yaml
name: Build Python Package

on:
  push:
    branches:
      - main

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - name: Build package
        uses: our-organization/data-platform-gha-templates/actions/uv_build@main
        with:
          python_version: '3.12'
          uv_version: '0.8.x'
```

## Inputs

The action supports the following inputs:

-   `python_version`: The version of Python to use for building the package. Defaults to `"3.11"`.
-   `uv_version`: The version of UV to use. Defaults to `'0.7.x'`.
-   `package`: The name of the package to build. Optional.
-   `twine_version`: The version of twine to install. Defaults to `'6.1.0'`.

## Outputs

This action does not have any outputs.

## Artifacts

The action uploads the built package artifacts from the `dist/` directory. If the `package` input is provided, the artifact is named `dist-<package-name>`; otherwise, it defaults to `dist`. The artifacts are retained for 7 days.
