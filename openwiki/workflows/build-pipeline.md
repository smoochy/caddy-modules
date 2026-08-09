---
type: Workflow
title: Build Pipeline Workflow
description: Detailed documentation of the GitHub Actions build pipeline for caddy-modules, covering change detection, upstream version fetching, build decisions, and publishing.
tags: [workflow, github-actions, ci-cd, build-pipeline, change-detection]
openwiki:
  roles: [workflow, delivery]
  change_kinds: [lifecycle, configuration, operations]
  source_paths: [.github/workflows/build_cloudflare-modules.yaml]
  symbols: [decide, local_changes, addon_labels, meta, build-push]
  test_paths: []
  invariants: [Build runs only when upstream changes or local inputs change, All addon versions fetched from GitHub API at build time, OCI labels enable change detection across runs]
  validation_commands: [gh workflow run build_cloudflare-modules.yaml --ref main]
---

# Build Pipeline Workflow

## Workflow File
`.github/workflows/build_cloudflare-modules.yaml` (649 lines)

## Trigger Conditions

| Trigger | Condition | Behavior |
|---------|-----------|----------|
| **Push to main** | Files changed: `Dockerfile-cloudflare`, `.dockerignore`, workflow file | Builds and publishes |
| **Pull Request** | Same files as push | Validation build (no push) |
| **Schedule** | Daily 03:00 UTC (`0 3 * * *`) | Checks upstream, builds if changed |
| **Manual** | `workflow_dispatch` with optional `force=true` | Builds regardless of changes |

**Path filtering** prevents rebuilds for documentation-only changes (README.md, etc.).

## Job: `build`

Runs on `ubuntu-latest` with concurrency group `${{ github.workflow }}-${{ github.ref }}` (cancels in-progress runs on same ref).

### Step 1: Classify Local Changes (`local_changes`)

Determines what changed in the push/PR:

```bash
# Compares base vs head SHA for PR, or before vs current SHA for push
changed_files="$(git diff --name-only "$compare_from" "$compare_to")"

image_inputs_changed=false
workflow_changed=false

# Dockerfile* or .dockerignore changed
printf '%s\n' "$changed_files" | grep -Eq '^(Dockerfile[^/]*|\.dockerignore)$'

# Workflow file itself changed
printf '%s\n' "$changed_files" | grep -Fxq ".github/workflows/build_cloudflare-modules.yaml"
```

Outputs:
- `image_inputs_changed` - Dockerfile or .dockerignore modified
- `workflow_changed` - Workflow file modified

### Step 2: Setup (QEMU, Buildx, GHCR Login)

Standard Docker Buildx setup with QEMU for multi-arch. GHCR login uses `GITHUB_TOKEN` (built-in, no secrets needed).

### Step 3: Decide Whether to Build (`decide`)

**This is the core logic step** - a 400+ line bash script that:

1. **Fetches current upstream state**:
   - `crane digest caddy:latest` → base image digest
   - `crane config caddy:latest` → base image labels (extracts Caddy version)
   - For each addon from Dockerfile: GitHub API `/repos/{owner}/{repo}/releases` → latest stable version + release notes

2. **Reads published image metadata** (if exists):
   - `crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest`
   - Extracts: `org.opencontainers.image.base.digest`, `org.opencontainers.image.base.version`, `org.opencontainers.image.addon.N.version`

3. **Compares and decides**:
   ```
   FORCED = true if:
     - workflow_dispatch with force=true
     - PR with image_inputs_changed OR workflow_changed
     - Push with image_inputs_changed

   CHANGED = true if:
     - No current image exists (first build)
     - Caddy base digest changed
     - Any addon version changed (compared via OCI labels)

   do_build = FORCED OR CHANGED
   ```

4. **Generates build reason list** for job summary:
   - "Repository push changed local image inputs"
   - "caddy:latest base changed: `sha256:abc1234` -> `sha256:def5678`"
<!-- openwiki: broken internal link [...] file "..." does not exist. Fix the href or restore the target, then delete this comment. -->
   - "Cloudflare module updated: `v1.2.3` -> `v1.2.4` ([release notes](...))"
   - "No upstream changes detected (caddy/cloudflare unchanged)"

5. **Outputs** (key ones):
   - `do_build` - "true"/"false"
   - `caddy_tag` - e.g., "2.8.4" or "latest"
   - `caddy_digest` - e.g., "sha256:abc123..."
   - `cloudflare_version` - Cloudflare DNS module version
   - `addon_N_name`, `addon_N_version`, `addon_N_url`, `addon_N_notes` - for each discovered addon
   - `addon_count` - number of addons
   - `caddy_release_notes` - escaped release notes from Caddy release

### Step 4: Metadata (`meta`)

Uses `docker/metadata-action@v6` to generate tags:
- `type=raw,value=latest`
- `type=raw,value=caddy-${{ steps.decide.outputs.caddy_tag }}`

### Step 5: Prepare Addon Labels (`addon_labels`)

Dynamically generates OCI labels for **all discovered addons** (not hardcoded):

```bash
for i in $(seq 0 $((addon_count - 1))); do
  name=$(grep "^addon_${i}_name=" "$GITHUB_OUTPUT" | cut -d= -f2-)
  version=$(grep "^addon_${i}_version=" "$GITHUB_OUTPUT" | cut -d= -f2-)
  labels+="org.opencontainers.image.addon.${i}.name=${name}\n"
  labels+="org.opencontainers.image.addon.${i}.version=${version}\n"
done
```

Labels produced:
- `org.opencontainers.image.addon.0.name=github.com/caddy-dns/cloudflare`
- `org.opencontainers.image.addon.0.version=v1.2.3`
- `org.opencontainers.image.addon.1.name=github.com/WeidiDeng/caddy-cloudflare-ip`
- ...etc.

Also adds base image labels:
- `org.opencontainers.image.base.name=caddy:latest`
- `org.opencontainers.image.base.tag=<caddy_tag>`
- `org.opencontainers.image.base.digest=<caddy_digest>`
- `org.opencontainers.image.base.version=<caddy_tag>`
- `org.opencontainers.image.cloudflare.version=<cloudflare_version>` (legacy, for backward compat)

### Step 6: Build and Push (`build-push`)

Uses `docker/build-push-action@v7`:
- Context: `.`
- File: `./Dockerfile-cloudflare`
- Platforms: `linux/amd64,linux/arm64,linux/arm/v7`
- Push: `true` (except for PR)
- Tags: from `meta` step
- Labels: from `meta` + `addon_labels` + base labels
- Cache: GitHub Actions cache (`type=gha`)

### Step 7: Docker Hub Mirror (Conditional)

Only runs if:
- `do_build == true`
- Not a PR
- `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets configured

Uses `crane copy` to mirror both tags:
```bash
crane copy ghcr.io/smoochy/caddy-cloudflare-modules:latest \
           smoochy84/caddy-cloudflare-modules:latest
crane copy ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<tag> \
           smoochy84/caddy-cloudflare-modules:caddy-<tag>
```

### Step 8: Write Build Summary

Always runs (`if: always()`). Outputs structured markdown to `$GITHUB_STEP_SUMMARY`:

```
### Publish reason:
- Repository push changed local image inputs
- caddy:latest base changed: `sha256:abc1234` -> `sha256:def5678`

### Selected versions (this run):
- caddy digest: `sha256:def5678`
<!-- openwiki: broken internal link [...] file "..." does not exist. Fix the href or restore the target, then delete this comment. -->
- caddy: `2.8.4` ([GitHub tag](...))

### Addons:
<!-- openwiki: broken internal link [...] file "..." does not exist. Fix the href or restore the target, then delete this comment. -->
- github.com/caddy-dns/cloudflare: `v1.2.3` ([release](...))
<!-- openwiki: broken internal link [...] file "..." does not exist. Fix the href or restore the target, then delete this comment. -->
- github.com/WeidiDeng/caddy-cloudflare-ip: `v0.1.0` ([repo](...))

### Changelogs
<details><summary>caddy <code>2.8.4</code></summary>
> Release notes...
</details>
<details><summary>github.com/caddy-dns/cloudflare <code>v1.2.3</code></summary>
> Release notes...
</details>

### Image Tags
- `ghcr.io/smoochy/caddy-cloudflare-modules:latest`
- `ghcr.io/smoochy/caddy-cloudflare-modules:caddy-2.8.4`

### Docker Hub Mirror
- `smoochy84/caddy-cloudflare-modules:latest`
- `smoochy84/caddy-cloudflare-modules:caddy-2.8.4`
```

## Change Detection Deep Dive

### What Triggers a Build

| Source | Detection Method | Label Key |
|--------|------------------|-----------|
| Caddy base image | `crane digest` comparison | `org.opencontainers.image.base.digest` |
| Caddy version | Label comparison | `org.opencontainers.image.base.version` |
| Cloudflare DNS module | GitHub API release version | `org.opencontainers.image.addon.0.version` |
| Cloudflare IP module | GitHub API release version | `org.opencontainers.image.addon.1.version` |
| Combine IP ranges | GitHub API release version | `org.opencontainers.image.addon.2.version` |
| Docker proxy | GitHub API release version | `org.opencontainers.image.addon.3.version` |

### Forced Builds

| Scenario | Force Condition |
|----------|-----------------|
| Manual run with `force=true` | Always builds |
| PR with Dockerfile change | Builds (validation) |
| PR with workflow change | Builds (validation) |
| Push with Dockerfile change | Builds + publishes |
| Push with workflow change only | **Does not publish** unless upstream changed |

### First Build

If no published image exists (`crane manifest` fails), `CHANGED=true` → builds.

## Adding a New Addon

1. Add `--with github.com/owner/repo` to `Dockerfile-cloudflare`
2. Workflow automatically:
   - Discovers it via `grep -- '--with'`
   - Fetches version from GitHub API
   - Adds OCI labels
   - Includes in job summary and changelog
   - Tracks for future change detection

No workflow changes needed.

## Validation Commands

```bash
# Trigger manual build (with force)
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true

# View latest run summary
gh run list --workflow=build_cloudflare-modules.yaml --limit=1
gh run view --log-failed

# Check published image labels
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels'

# Verify multi-arch manifest
crane manifest ghcr.io/smoochy/caddy-cloudflare-modules:latest
```