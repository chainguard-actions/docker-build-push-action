<!-- markdownlint-disable -->

# Hardening Report: docker--build-push-action/v6

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **docker--build-push-action/v6** was hardened automatically. 4 finding(s) were identified and resolved across 2 iteration(s).

## Findings Fixed

### unpinned-uses (severity: high)

Multiple workflow files reference external actions using mutable version tags instead of pinned 40-character SHA hashes, making them vulnerable to supply-chain attacks. Affected references include: `actions/checkout@v6`, `docker/setup-buildx-action@v3`, `docker/setup-qemu-action@v3`, `docker/metadata-action@v5`, `docker/login-action@v3`, `docker/bake-action@v6`, `docker/bake-action/subaction/list-targets@v6`, `codecov/codecov-action@v5`, `actions/cache@v5`, `actions/create-github-app-token@v2`, `actions/publish-immutable-action@v0.0.4`.

Locations:

- `.github/workflows/ci.yml:33`
- `.github/workflows/ci.yml:36`
- `.github/workflows/.e2e-run.yml:53`
- `.github/workflows/.e2e-run.yml:87`
- `.github/workflows/.e2e-run.yml:89`
- `.github/workflows/.e2e-run.yml:97`
- `.github/workflows/.e2e-run.yml:103`
- `.github/workflows/test.yml:17`
- `.github/workflows/test.yml:20`
- `.github/workflows/test.yml:24`
- `.github/workflows/update-dist.yml:14`
- `.github/workflows/update-dist.yml:22`
- `.github/workflows/update-dist.yml:30`
- `.github/workflows/validate.yml:16`
- `.github/workflows/validate.yml:20`
- `.github/workflows/validate.yml:33`
- `.github/workflows/publish.yml:12`
- `.github/workflows/publish.yml:14`

### missing-permissions (severity: medium)

These workflow files have no top-level `permissions:` key and no job-level `permissions:` keys on any job. Without explicit permissions, workflows run with the default (potentially broad) token permissions.

Locations:

- `.github/workflows/ci.yml:1`
- `.github/workflows/e2e.yml:1`
- `.github/workflows/test.yml:1`
- `.github/workflows/update-dist.yml:1`
- `.github/workflows/validate.yml:1`
- `.github/workflows/.e2e-run.yml:1`

### script-injection (severity: high)

Multiple `run:` blocks directly interpolate `${{ }}` expressions inside shell commands (sub-rule a). This allows expression values to be interpreted as shell code before the shell executes the script. Affected instances include: `${{ inputs.id }}` used in a file path in a shell command, `${{ inputs.type }}` used in a shell conditional, `${{ env.REGISTRY_FQDN }}` used in an echo command, `${{ steps.docker_build.outputs.digest }}`, `${{ steps.docker_build.outputs.imageid }}`, `${{ matrix.driver }}`, `${{ matrix.load }}`, `${{ matrix.push }}`, `${{ env.DOCKER_IMAGE }}`, `${{ steps.meta.outputs.version }}` all interpolated directly into `run:` shell scripts.

Locations:

- `.github/workflows/.e2e-run.yml:56`
- `.github/workflows/.e2e-run.yml:62`
- `.github/workflows/.e2e-run.yml:64`
- `.github/workflows/.e2e-run.yml:72`
- `.github/workflows/.e2e-run.yml:77`
- `.github/workflows/.e2e-run.yml:113`
- `.github/workflows/.e2e-run.yml:116`
- `.github/workflows/ci.yml:75`
- `.github/workflows/ci.yml:89`
- `.github/workflows/ci.yml:213`
- `.github/workflows/ci.yml:217`
- `.github/workflows/ci.yml:296`
- `.github/workflows/ci.yml:302`
- `.github/workflows/ci.yml:306`
- `.github/workflows/ci.yml:311`
- `.github/workflows/ci.yml:316`

### github-env-injection (severity: high)

In `.github/workflows/.e2e-run.yml`, the step 'Set up env' runs `cat ./.github/e2e/${{ inputs.id }}/env >> $GITHUB_ENV`. The `inputs.id` value is workflow-caller-controlled and is interpolated directly into the shell command to form a file path, and the entire contents of that file are written to `$GITHUB_ENV` without any sanitization (no `tr -d '\n\r'` applied). A malicious caller could supply an `inputs.id` value that causes arbitrary environment variable injection via newline characters in the file content.

Locations:

- `.github/workflows/.e2e-run.yml:56`

## Iteration Notes

### Iteration 1

**Fixes applied:** unpinned-uses, missing-permissions, script-injection, github-env-injection

**Notes:**

Fixed all four findings across 6 workflow files:

1. **unpinned-uses**: Pinned all 11 action references to full 40-char SHAs in ci.yml (45 checkout, 41 setup-buildx, 10 setup-qemu, 2 metadata-action, 1 cache), test.yml (bake-action, codecov), update-dist.yml (create-github-app-token, checkout, bake-action), validate.yml (checkout, bake-action/subaction/list-targets, bake-action), publish.yml (checkout, publish-immutable-action), and .e2e-run.yml (checkout, metadata-action, setup-qemu, setup-buildx, login-action).

2. **missing-permissions**: Added `permissions: contents: read` top-level block to ci.yml, e2e.yml, test.yml, validate.yml, and .e2e-run.yml. Added `permissions: contents: write` + `pull-requests: read` to update-dist.yml (needs write to commit/push dist changes).

3. **script-injection**: Moved all ${{ }} expressions from run: blocks into env: blocks in ci.yml (digest checks, image ID checks, manifest inspects, example job inspect/check) and .e2e-run.yml (BuildKit config, Docker daemon setup, inspect/check manifest steps).

4. **github-env-injection**: In .e2e-run.yml Set up env step, moved inputs.id to INPUT_ID env var and sanitized it with `tr -d '\n\r/\\'` before using as a file path component to prevent path traversal and newline injection into GITHUB_ENV.

### Iteration 2

**Fixes applied:** script-injection, hardcoded-credentials

**Notes:**

Fixed 3 script-injection instances in .github/workflows/ci.yml: moved `${{ steps.docker_build.outcome }}` and `${{ steps.docker_build.conclusion }}` from inline shell `if` statements into `env:` blocks (as `OUTCOME` and `CONCLUSION`) in the `error`, `error-buildx`, and `call-check` jobs' Check steps. Fixed 1 hardcoded-credentials instance: replaced the literal `"MYSECRET=aaaaaaaa\nbbbbbbb\nccccccccc"` multi-line secret value in the `git-context-secret` job with `MYSECRET=${{ secrets.MYSECRET }}` to use a proper GitHub secret reference. Also removed the `EMPTYLINE` and `FOO=bar` test dummy values that were part of the same hardcoded block.

