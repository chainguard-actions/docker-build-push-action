<!-- markdownlint-disable -->

# Hardening Report: docker--build-push-action/v7.0.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **docker--build-push-action/v7.0.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Sub-rule (a): Direct expression interpolation in run: blocks. In .e2e-run.yml, attacker-controllable workflow_call inputs are interpolated directly into shell commands: `cat ./.github/e2e/${{ inputs.id }}/env >> $GITHUB_ENV` (line 59), `if [ "${{ inputs.type }}" = "local" ]` (line 63), `echo -e "[registry.\"${{ env.REGISTRY_FQDN }}\"]...` (line 64), `DOCKERD_CONFIG=$(jq '.+{"insecure-registries":["http://${{ env.REGISTRY_FQDN }}"]}' ...)` (line 74), and `sudo -E bash ./.github/e2e/${{ inputs.id }}/install.sh` (line 81). In ci.yml, multiple run: blocks interpolate ${{ steps.docker_build.outputs.digest }}, ${{ steps.docker_build.outcome }}, ${{ steps.docker_build.conclusion }}, ${{ env.DOCKER_IMAGE }}, ${{ steps.meta.outputs.version }}, ${{ matrix.driver }}, ${{ matrix.load }}, ${{ matrix.push }}, and ${{ steps.docker_build.outputs.imageid }} directly into shell commands.

Locations:

- `.github/workflows/.e2e-run.yml:59`
- `.github/workflows/.e2e-run.yml:63`
- `.github/workflows/.e2e-run.yml:64`
- `.github/workflows/.e2e-run.yml:74`
- `.github/workflows/.e2e-run.yml:81`
- `.github/workflows/ci.yml:68`
- `.github/workflows/ci.yml:113`
- `.github/workflows/ci.yml:175`
- `.github/workflows/ci.yml:215`
- `.github/workflows/ci.yml:228`

### github-env-injection (severity: high)

In .e2e-run.yml, the 'Set up env' step writes the contents of a file whose path is derived from the workflow_call input ${{ inputs.id }} directly to $GITHUB_ENV without any sanitization: `cat ./.github/e2e/${{ inputs.id }}/env >> $GITHUB_ENV`. An attacker controlling the `id` input (or the file contents) can inject arbitrary environment variable definitions, including newline-based key=value pairs that override subsequent steps' environment.

Locations:

- `.github/workflows/.e2e-run.yml:59`

### unpinned-uses (severity: high)

Multiple workflow files reference external actions using mutable version tags instead of immutable 40-character commit SHAs, making them vulnerable to supply-chain attacks if the tag is moved. Failing references include: actions/checkout@v6, docker/setup-buildx-action@v4, docker/setup-qemu-action@v4, docker/metadata-action@v5, docker/login-action@v4, docker/bake-action@v6, docker/bake-action/subaction/list-targets@v6, codecov/codecov-action@v5, actions/create-github-app-token@v2, actions/publish-immutable-action@v0.0.4.

Locations:

- `.github/workflows/ci.yml:32`
- `.github/workflows/.e2e-run.yml:53`
- `.github/workflows/e2e.yml:1`
- `.github/workflows/test.yml:18`
- `.github/workflows/update-dist.yml:14`
- `.github/workflows/validate.yml:14`
- `.github/workflows/publish.yml:14`

### missing-permissions (severity: medium)

The following workflow files have no top-level `permissions:` key and no job-level `permissions:` keys on any job, meaning they run with the default (potentially broad) token permissions: ci.yml, .e2e-run.yml, e2e.yml, test.yml, update-dist.yml, and validate.yml.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/.e2e-run.yml:1`
- `.github/workflows/e2e.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`
- `.github/workflows/validate.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, unpinned-uses, missing-permissions

**Notes:**

Fixed all four findings across 7 workflow files:

1. script-injection: Moved all ${{ }} expressions from run: shell commands to env: blocks in both .e2e-run.yml and ci.yml. Affected steps: Set up BuildKit config, Set up Docker daemon, Install, Inspect image, Check manifest (in .e2e-run.yml); Check digest, Inspect image, Check manifest, Check (error/error-buildx/call-check), Check digest/manifest/image ID/Inspect image (digest job) in ci.yml.

2. github-env-injection: Fixed the 'Set up env' step in .e2e-run.yml to sanitize inputs.id before using as a path component (stripping non-alphanumeric chars via sed), and to strip carriage returns from env file contents before writing to $GITHUB_ENV.

3. unpinned-uses: Pinned all 11 action references to full 40-character commit SHAs across ci.yml, .e2e-run.yml, test.yml, update-dist.yml, validate.yml, and publish.yml.

4. missing-permissions: Added top-level 'permissions: contents: read' to ci.yml, .e2e-run.yml, e2e.yml, test.yml, validate.yml, and update-dist.yml. The update-dist job that pushes commits gets a job-level 'permissions: contents: write' override.

