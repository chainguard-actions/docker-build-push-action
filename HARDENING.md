<!-- markdownlint-disable -->

# Hardening Report: docker--build-push-action/v7.3.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **docker--build-push-action/v7.3.0** was hardened automatically. 4 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `run:` blocks in `.github/workflows/.e2e-run.yml` directly interpolate `${{ }}` expressions into shell commands, violating rule (a). In the 'Set up BuildKit config' step, `${{ env.REGISTRY_FQDN }}` is interpolated directly into an `echo` command: `echo -e "[registry.\"${{ env.REGISTRY_FQDN }}\"]\nhttp = true\ninsecure = true" > /tmp/buildkitd.toml`. In the 'Set up Docker daemon' step, `${{ env.REGISTRY_FQDN }}` is interpolated directly into a `jq` command: `DOCKERD_CONFIG=$(jq '.+{"insecure-registries":["http://${{ env.REGISTRY_FQDN }}"]}' /etc/docker/daemon.json)`. Any `${{ ... }}` expression inside a `run:` block is a script-injection risk because YAML template substitution occurs before the shell ever sees the value.

Locations:

- `.github/workflows/.e2e-run.yml:70`
- `.github/workflows/.e2e-run.yml:78`

### script-injection (severity: high)

Multiple `run:` blocks in `.github/workflows/ci.yml` directly interpolate `${{ }}` expressions into shell commands, violating rule (a). Examples include: `if [ -z "${{ steps.docker_build.outputs.digest }}" ]` (repeated across multiple jobs), `docker image inspect ${{ env.DOCKER_IMAGE }}:${{ steps.meta.outputs.version }}`, `docker buildx imagetools inspect ${{ env.DOCKER_IMAGE }}@${{ steps.docker_build.outputs.digest }}`, `if [[ "${{ matrix.driver }}" = "docker-container" ]]`, `if [ "${{ steps.docker_build.outcome }}" != "failure" ]`, and `docker buildx imagetools inspect ${{ env.DOCKER_IMAGE }}:${{ steps.meta.outputs.version }}`. All of these interpolate workflow context values directly into shell command strings before the shell processes them.

Locations:

- `.github/workflows/ci.yml:99`
- `.github/workflows/ci.yml:148`
- `.github/workflows/ci.yml:213`
- `.github/workflows/ci.yml:268`
- `.github/workflows/ci.yml:330`
- `.github/workflows/ci.yml:356`
- `.github/workflows/ci.yml:400`
- `.github/workflows/ci.yml:415`
- `.github/workflows/ci.yml:430`
- `.github/workflows/ci.yml:870`
- `.github/workflows/ci.yml:960`
- `.github/workflows/ci.yml:990`
- `.github/workflows/ci.yml:1050`
- `.github/workflows/ci.yml:1430`
- `.github/workflows/ci.yml:1700`

### github-env-injection (severity: high)

In `.github/workflows/.e2e-run.yml`, the 'Set up env' step writes the contents of a file path derived from the workflow input `inputs.id` directly to `$GITHUB_ENV` without any sanitization: `cat ./.github/e2e/${ID}/env >> $GITHUB_ENV`. The `ID` variable is set from `${{ inputs.id }}` in the step's `env:` block. A caller of this reusable workflow can supply a crafted `id` input or the referenced env file may contain newline-injected key=value pairs, allowing injection of arbitrary environment variables into subsequent steps. The required sanitization (`printf '%s' ... | tr -d '\n\r'`) is not applied before the write.

Locations:

- `.github/workflows/.e2e-run.yml:62`

### missing-permissions (severity: medium)

The reusable workflow `.github/workflows/.e2e-run.yml` has no top-level `permissions:` key and no job-level `permissions:` key on its only job (`run`). Without explicit permissions, the workflow inherits the default permissions of the calling workflow, which may be overly broad. A top-level or job-level `permissions:` block with minimal specific scopes should be added.

Locations:

- `.github/workflows/.e2e-run.yml:1`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection, missing-permissions

**Notes:**

Fixed all 4 findings across 2 files:

1. .e2e-run.yml - missing-permissions: Added top-level `permissions: contents: read` block.

2. .e2e-run.yml - script-injection (lines 70, 78): Moved `${{ env.REGISTRY_FQDN }}` out of shell commands into step `env:` blocks. In 'Set up BuildKit config', referenced as `${REGISTRY_FQDN}`. In 'Set up Docker daemon', used `jq --arg fqdn "http://${REGISTRY_FQDN}"` to safely pass the value.

3. .e2e-run.yml - github-env-injection (line 62): Added ID validation (alphanumeric/underscore/hyphen only) to prevent path traversal, and replaced `cat file >> $GITHUB_ENV` with a line-by-line loop that strips carriage returns via `tr -d '\r'` before writing to GITHUB_ENV.

4. ci.yml - script-injection (15 locations): Moved all `${{ }}` expressions from shell commands into step `env:` blocks across 9 jobs: git-context, git-context-query, git-context-secret, path-context, example, error, error-buildx, digest, multi, registry-cache, annotations, and call-check. Each expression is now referenced as a plain environment variable in the shell script.

