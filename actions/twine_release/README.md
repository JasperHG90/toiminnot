# Release Python Package with Twine

This GitHub Action releases a Python package to a specified repository using Twine.

## Usage

To use this action, you need to configure your workflow to authenticate with your package repository. Below is an example for releasing to Google Artifact Registry.

```yaml
name: Release Python Package

on:
  release:
    types: [published]

jobs:
  release:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      id-token: write
    steps:
      - name: Set up Google Cloud authentication
        id: auth
        uses: google-github-actions/auth@v2
        with:
          workload_identity_provider: "projects/0000000000/locations/global/workloadIdentityPools/github/providers/github-provider"
          service_account: "github-actions@our-organization-data-platform-artifacts-prod.iam.gserviceaccount.com"
          project_id: "our-organization-data-platform-artifacts-prod"
          token_format: "access_token" # Required to pass the TWINE_PASSWORD env variable
      - name: Release package
        uses: our-organization/data-platform-gha-templates/actions/twine_release@main
        with:
          python_version: '3.12'
          uv_version: '0.8.x'
        env:
          TWINE_REPOSITORY_URL: "https://europe-west4-python.pkg.dev/your-project/your-repo/"
          TWINE_USERNAME: "oauth2accesstoken"
          TWINE_PASSWORD: ${{ steps.auth.outputs.access_token }}
```

## Inputs

The action supports the following inputs:

-   `python_version`: The version of Python to use. Defaults to `"3.11"`.

## Environment Variables

To publish the package, you must configure the following environment variables for Twine:

-   `TWINE_REPOSITORY_URL`: The URL of the package repository.
-   `TWINE_USERNAME`: The username for authentication.
-   `TWINE_PASSWORD`: The password or token for authentication.

## Outputs

This action does not have any outputs.
