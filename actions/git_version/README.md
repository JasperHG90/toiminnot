# Git Version

**Description:** GitHub action to retrieve a PEP-compatible version based on git history and tags. This repository is inspired by [poetry-git-version-plugin](https://pypi.org/project/poetry-git-version-plugin/). Unlike that package, this action has been written in bash and requires no additional dependencies.

**Inputs:**
- `repo_path`: Repository path. Defaults to root repo (default: ".").

**Outputs:**
- `version`: The PEP-compatible version string.

**Example Usage:**
```yaml
jobs:
  test_git_version_action:
    runs-on: ubuntu-latest
    name: 'Test GHA'
    steps:
      - name: checkout
        uses: actions/checkout@v4
        with:
          fetch-depth: 0 # NB: required to get all tags
      - name: 'Get version'
        id: version
        uses: our-organization/data-platform-gha-templates/actions/git_version@main # or use a specific release e.g. @0.0.1
      - name: 'Print version'
        run: echo "Version is ${{ steps.version.outputs.version }}"
```

The action will return a version based on the **release tag**. E.g. 'v1.0.0' will become '1.0.0'.

If HEAD is not tagged, then the script will take the last tag that is formatted as 'v*', compute the distance from that tag and the current commit, and add that distance to the version with a short commit SHA. For example: `1.0.0a50+7342fed`.

You can use this action for docker images as well, but that requires an additional step:

```yaml
jobs:
   ...
    - name: "Modify version for docker"
      id: modify-version
      run: |
        VERSION="${{ steps.version.outputs.version }}"
        VERSION="${VERSION//+/\.}" # Replace + with .

        echo "VERSION=$VERSION" >> $GITHUB_ENV
        echo "VERSION=$VERSION" >> $GITHUB_OUTPUT
```
