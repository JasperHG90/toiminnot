# Pytest

**Description:** Template for running tests with pytest.
**Inputs:**
- `directory`: Path to tests (required).
- `uv_version`: uv version to use (required).
- `python_version`: Python version to use (default: "3.11").
- `markers`: Pytest markers to use (default: "").

**Example Usage:**
```yaml
- uses: our-organization/data-platform-gha-templates/actions/pytest@main # or use a specific release e.0.1
  with:
    directory: "tests/"
    uv_version: "0.7.x"
    python_version: "3.11"
    markers: "not slow"
```

**Authenticating**

If you're package is using libraries from a private pypi (e.g. GCP artifact registry), then you must supply credentials via environment variables.

For example, if you have the following entry in your pyproject.toml:

```toml
[[tool.uv.index]]
name = "data-platform-packages"
url = "https://oauth2accesstoken@europe-west4-python.pkg.dev/our-organization-data-platform-artifacts-prod/data-platform-python-packages/simple/"
priority = "supplemental"
```

Then you would use in GHA:

```yaml
name: CI

on:
  pull_request:
    branches: [main]
  workflow_dispatch:

concurrency:
  group: ${{ github.workflow }}-${{ github.ref }}
  cancel-in-progress: true

env:
  PYTHON_VERSION: "3.12"
  UV_VERSION: "0.7.x"
  UV_KEYRING_PROVIDER: "subprocess"
  UV_INDEX_DATA_PLATFORM_PACKAGES_USERNAME: "oauth2accesstoken"

permissions:
  contents: read
  id-token: write

jobs:
  unit-tests:
    name: Unit tests
    runs-on: ubuntu-latest
    strategy:
      matrix:
        python-version: ["3.12"]
    steps:
      - name: Set up Google Cloud authentication
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/00000000000000/locations/global/workloadIdentityPools/github/providers/github-provider"
          service_account: "github-actions@our-organization-data-platform-artifacts-prod.iam.gserviceaccount.com"
          project_id: "our-organization-data-platform-artifacts-prod"
          token_format: "access_token" # NB: required to be able to pass UV_INDEX_DATA_PLATFORM_PACKAGES_PASSWORD as access token. See auth docs.
      - name: Run unit tests
        uses: our-organization/data-platform-gha-templates/actions/pytest@main
        env:
          UV_INDEX_DATA_PLATFORM_PACKAGES_USERNAME: ${{ env.UV_INDEX_DATA_PLATFORM_PACKAGES_USERNAME }}
          UV_INDEX_DATA_PLATFORM_PACKAGES_PASSWORD: ${{ steps.auth.outputs.access_token }}
        with:
          directory: tests
          uv_version: ${{ env.UV_VERSION }}
          python_version: ${{ matrix.python-version }}
          markers: "not integration"
```

Note that you must have "id-token: write" permissions for this.

Strictly speaking, you won't need to add 'UV_INDEX_DATA_PLATFORM_PACKAGES_USERNAME' since it has also been set in the URL.
