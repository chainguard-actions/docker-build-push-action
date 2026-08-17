<!-- markdownlint-disable -->

# Hardening Report: docker--build-push-action/v7.2.0

> This file was generated automatically by the hardening agent.

**Policy SHA:** `d636be7e43ef829af6e853da6b3c7566db9f72fe`

**Test Policy SHA:** `843adf9e4b8f85d0c08b27b9d0b09dd094b54702`

**Harden Agent Version:** `2`

Action **docker--build-push-action/v7.2.0** was hardened automatically. 3 finding(s) were identified and resolved across 3 iteration(s).

## Findings Fixed

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in `.github/workflows/.e2e-run.yml`, violating rule (a). Offending lines include:
- `echo -e "[registry.\"${{ env.REGISTRY_FQDN }}\"]\nhttp = true\ninsecure = true" > /tmp/buildkitd.toml` — the env context value is injected directly into the shell command before the shell parses it.
- `DOCKERD_CONFIG=$(jq '.+{"insecure-registries":["http://${{ env.REGISTRY_FQDN }}"]}' /etc/docker/daemon.json)` — same issue.
- `docker pull ${SLUG}:${{ steps.meta.outputs.version }}` — step output injected directly.
- `docker image inspect ${SLUG}:${{ steps.meta.outputs.version }}` — step output injected directly.
- `docker buildx imagetools inspect ${SLUG}:${{ steps.meta.outputs.version }} --format '{{json .}}'` — step output injected directly.
All of these allow an attacker-controlled or workflow-controlled value to be interpreted by the shell before quoting can protect it.

Locations:

- `.github/workflows/.e2e-run.yml:70`
- `.github/workflows/.e2e-run.yml:79`
- `.github/workflows/.e2e-run.yml:113`
- `.github/workflows/.e2e-run.yml:114`
- `.github/workflows/.e2e-run.yml:118`

### script-injection (severity: high)

Multiple `${{ }}` expressions are interpolated directly inside `run:` shell command strings in `.github/workflows/ci.yml`, violating rule (a). Offending patterns include:
- `docker image inspect ${{ env.DOCKER_IMAGE }}:${{ steps.meta.outputs.version }}` — env and step-output contexts injected directly into shell.
- `docker buildx imagetools inspect ${{ env.DOCKER_IMAGE }}:${{ steps.meta.outputs.version }} --format '{{json .}}'` — same.
- `if [ -z "${{ steps.docker_build.outputs.digest }}" ]` — step output injected directly into shell conditional.
- `if [[ "${{ matrix.driver }}" = "docker-container" ]] && [[ "${{ matrix.load }}" = "false" ]] && [[ "${{ matrix.push }}" = "false" ]]` — matrix context injected directly.
- `docker buildx imagetools inspect ${{ env.DOCKER_IMAGE }}@${{ steps.docker_build.outputs.digest }}` — env and step output injected directly.
- `docker image inspect ${{ steps.docker_build.outputs.imageid }}` — step output injected directly.
- `if [ "${{ steps.docker_build.outcome }}" != "failure" ] || [ "${{ steps.docker_build.conclusion }}" != "success" ]` — step outputs injected directly.
Any of these values flowing through YAML template substitution before the shell sees them can carry shell metacharacters.

Locations:

- `.github/workflows/ci.yml:258`
- `.github/workflows/ci.yml:263`
- `.github/workflows/ci.yml:270`
- `.github/workflows/ci.yml:316`
- `.github/workflows/ci.yml:730`
- `.github/workflows/ci.yml:735`
- `.github/workflows/ci.yml:742`
- `.github/workflows/ci.yml:1120`
- `.github/workflows/ci.yml:1330`

### github-env-injection (severity: high)

In `.github/workflows/.e2e-run.yml`, the 'Set up env' step writes the contents of a file whose path is derived from `inputs.id` (an untrusted workflow input) directly to `$GITHUB_ENV` without any sanitization:
```yaml
env:
  ID: ${{ inputs.id }}
run: |
  cat ./.github/e2e/${ID}/env >> $GITHUB_ENV
```
The `${ID}` variable is set from `${{ inputs.id }}`, which is caller-controlled. The file content piped into `$GITHUB_ENV` may contain newlines and arbitrary `KEY=VALUE` pairs, allowing an attacker to inject environment variables into subsequent steps. The required sanitization step (`printf '%s' ... | tr -d '\n\r'`) is absent.

Locations:

- `.github/workflows/.e2e-run.yml:62`

## Iteration Notes

### Iteration 1

**Fixes applied:** script-injection, github-env-injection

**Notes:**

Fixed all script-injection and github-env-injection findings:

**.e2e-run.yml**:
1. github-env-injection (line 62): Added ID validation (alphanumeric/underscore/hyphen only) to prevent path traversal, and line-by-line filtering of env file content to only write valid KEY=VALUE patterns to $GITHUB_ENV.
2. script-injection (line 70): Moved ${{ env.REGISTRY_FQDN }} to env block as REGISTRY_FQDN; replaced echo -e with printf using the env var in 'Set up BuildKit config'.
3. script-injection (line 79): Moved ${{ env.REGISTRY_FQDN }} to env block; used jq --arg to safely pass the registry value in 'Set up Docker daemon'.
4. script-injection (lines 113-114): Moved ${{ steps.meta.outputs.version }} to env block as META_VERSION in 'Inspect image'.
5. script-injection (line 118): Moved ${{ steps.meta.outputs.version }} to env block as META_VERSION in 'Check manifest'.

**ci.yml**:
1. Lines 258-263: Moved ${{ env.DOCKER_IMAGE }} and ${{ steps.meta.outputs.version }} to env blocks in 'example' job's 'Inspect image' and 'Check manifest' steps.
2. Line 270: Fixed all 6 'Check digest' steps using ${{ steps.docker_build.outputs.digest }} by moving to env block as BUILD_DIGEST.
3. Line 316: Fixed all 3 'Check' steps using ${{ steps.docker_build.outcome }} and ${{ steps.docker_build.conclusion }} by moving to env block as BUILD_OUTCOME and BUILD_CONCLUSION.
4. Lines 730-742: Fixed 'digest' job's 'Check digest', 'Check manifest', 'Check image ID', and 'Inspect image' steps by moving all matrix and step output expressions to env blocks.
5. Line 1120: Fixed 'annotations' job's 'Check manifest' step by moving ${{ env.DOCKER_IMAGE }} and ${{ steps.meta.outputs.version }} to env block.
6. Line 1330: Fixed 'call-check' job's 'Check' step (covered by the replace_all fix for all 3 Check steps).

### Iteration 2

**Fixes applied:** script-injection

**Notes:**

Fixed the script injection vulnerability in the 'Install' step of `.github/workflows/.e2e-run.yml`. The step had its own `env:` block binding `ID: ${{ inputs.id }}` and used it unquoted in `sudo -E bash ./.github/e2e/${ID}/install.sh`. The 'Set up env' step's validation did not protect this separately-bound env var.

Fix applied:
1. Added the same allowlist validation (`[[ "$ID" =~ [^a-zA-Z0-9_-] ]]`) directly inside the Install step, so it validates ID before use regardless of the prior step's validation.
2. Quoted the `${ID}` expansion in the bash command: `sudo -E bash "./.github/e2e/${ID}/install.sh"` — this ensures the shell treats the path as a single word and prevents word-splitting or glob expansion on the value.

### Iteration 3

**Fixes applied:** hardcoded-credentials

**Notes:**

Replaced the hardcoded literal value `MYSECRET=aaaaaaaa\nbbbbbbb\nccccccccc` in the `git-context-secret` job's `secrets:` input block with `MYSECRET=${{ secrets.MYSECRET }}`. This references a proper GitHub Actions repository secret instead of embedding a literal alphanumeric value directly in the workflow file. The multi-line quoted format was removed since the value is now a single-line expression.

