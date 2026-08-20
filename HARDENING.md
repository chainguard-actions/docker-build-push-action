<!-- markdownlint-disable -->

# Hardening Report: docker--build-push-action/v7.1.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **docker--build-push-action/v7.1.0** was hardened automatically. 3 finding(s) were identified and resolved across 1 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple run: blocks in ci.yml directly interpolate ${{ ... }} expressions inside shell commands (sub-rule a). Examples include: `if [ -z "${{ steps.docker_build.outputs.digest }}" ]` (Check digest steps), `if [ "${{ steps.docker_build.outcome }}" != "failure" ]` (Check steps), `docker buildx imagetools inspect ${{ env.DOCKER_IMAGE }}:${{ steps.meta.outputs.version }}` (Check manifest steps), and `docker image inspect ${{ steps.docker_build.outputs.imageid }}` (Inspect image step). These expressions are expanded by the GitHub Actions template engine before the shell sees them, allowing injection of shell metacharacters.

Locations:

- `.github/workflows/ci.yml:88`
- `.github/workflows/ci.yml:136`
- `.github/workflows/ci.yml:183`
- `.github/workflows/ci.yml:228`
- `.github/workflows/ci.yml:310`
- `.github/workflows/ci.yml:340`
- `.github/workflows/ci.yml:356`
- `.github/workflows/ci.yml:362`
- `.github/workflows/ci.yml:755`
- `.github/workflows/ci.yml:870`
- `.github/workflows/ci.yml:1265`

### script-injection (severity: high)

Multiple run: blocks in .e2e-run.yml directly interpolate ${{ ... }} expressions inside shell commands (sub-rule a). Examples include: `echo -e "[registry.\"${{ env.REGISTRY_FQDN }}\"]..."` (Set up BuildKit config step), `DOCKERD_CONFIG=$(jq '.+{"insecure-registries":["http://${{ env.REGISTRY_FQDN }}"]}' ...)` (Set up Docker daemon step), `docker pull ${SLUG}:${{ steps.meta.outputs.version }}` (Inspect image step), and `docker buildx imagetools inspect ${SLUG}:${{ steps.meta.outputs.version }}` (Check manifest step). These expressions are expanded before the shell sees them.

Locations:

- `.github/workflows/.e2e-run.yml:47`
- `.github/workflows/.e2e-run.yml:54`
- `.github/workflows/.e2e-run.yml:100`
- `.github/workflows/.e2e-run.yml:104`

### github-env-injection (severity: high)

In .e2e-run.yml, the 'Set up env' step writes the contents of a file at a path derived from the untrusted workflow input `inputs.id` directly to $GITHUB_ENV without sanitization: `cat ./.github/e2e/${ID}/env >> $GITHUB_ENV`. The env var ID is set from `${{ inputs.id }}`. File content can contain newlines and key=value pairs that inject arbitrary environment variables into subsequent steps.

Locations:

- `.github/workflows/.e2e-run.yml:42`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all script injection issues in ci.yml and .e2e-run.yml by moving ${{ }} expressions from run: shell commands into step env: blocks, then referencing them as plain environment variables. Fixed github-env-injection in .e2e-run.yml's 'Set up env' step by replacing direct cat-to-GITHUB_ENV with a sanitizing loop that strips carriage returns. Fixed the 'Set up BuildKit config' step to use printf instead of echo -e with interpolated REGISTRY_FQDN. Fixed 'Set up Docker daemon' to use jq --arg to safely pass REGISTRY_FQDN. Fixed 'Inspect image' and 'Check manifest' steps in .e2e-run.yml to use META_VERSION env var instead of inline ${{ steps.meta.outputs.version }}.

