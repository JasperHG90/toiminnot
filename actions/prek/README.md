# Prek

**Description:** Template for running prek checks.
**Inputs:**
- `python_version`: Python version to use (default: "3.11").
- `prek_version`: Prek version to use (default: "4.2.0").
- `directory`: Path to directory on which to run prek (default: ".").

**Example Usage:**
```yaml
- uses: our-organization/data-platform-gha-templates/actions/prek@main # or use a specific release e.g. @0.0.1
  with:
    python_version: "3.9"
    directory: "."
```
