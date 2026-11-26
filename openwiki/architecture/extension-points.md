---
type: Architecture
title: Extension Points
description: Documents how to extend the system by adding new modules to the Cloudflare image or adding new image variants, plus the invariants that must be preserved.
tags: [architecture, extension, docker, workflow, caddy, xcaddy]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-42ef8192a0f869c58499e372
    resource: repo://Dockerfile-cloudflare
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---

# Extension Points

This page documents the two primary extension mechanisms in the caddy-modules repository: adding a new module to an existing image variant, and adding an entirely new image variant. It also lists the system invariants that any extension must preserve.

---

## Adding a New Module to the Cloudflare Image

The Cloudflare image is defined by `Dockerfile-cloudflare`. To include another Caddy module, you add a `--with` line to the `xcaddy build` command.

### Exact Steps

1. **Edit `Dockerfile-cloudflare`** and append a new `--with` line with the Go module path of the addon:

   ```dockerfile
   RUN xcaddy build \
       --with github.com/caddy-dns/cloudflare \
       --with github.com/WeidiDeng/caddy-cloudflare-ip \
       --with github.com/fvbommel/caddy-combine-ip-ranges \
       --with github.com/lucaslorentz/caddy-docker-proxy/v2 \
       --with github.com/your-org/your-caddy-addon
   ```

2. **Commit and push** to `main` (or open a pull request).

### What Happens Automatically

The workflow (`.github/workflows/build_cloudflare-modules.yaml`) parses the Dockerfile at runtime to discover active addons:

```bash
grep -- '--with' Dockerfile-cloudflare | \
  while IFS= read -r line; do
    [[ "$line" =~ ^[[:space:]]*# ]] && continue
    [[ "$line" =~ --with[[:space:]]+([^[:space:]\\]+) ]]
  done
```

Because discovery is dynamic, the new `--with` line is **automatically**:
- Fetched for its latest version via GitHub API (`/repos/{owner}/{repo}/releases`)
- Included in the job summary with a link to the release notes
- Emitted as an OCI label: `org.opencontainers.image.addon.N.name` and `org.opencontainers.image.addon.N.version`
- Tracked for future upstream change detection (the workflow compares the stored label against the newly fetched version on every run)

No workflow changes are required.

### Constraints

- The `--with` argument must be a valid Go module path (e.g., `github.com/owner/repo` or `github.com/owner/repo/v2` for v2+ modules).
- Commented lines (starting with `#`) are ignored by both the build and the workflow parser.
- Version is **not pinned** in the Dockerfile; `xcaddy` resolves the latest compatible version at build time. The workflow records the exact version used via OCI labels.

> **Reference**: [README → Adding More Addons](https://github.com/smoochy/caddy-modules/blob/main/README.md#adding-more-addons)

---

## Adding a New Image Variant

Each image variant is self-contained and consists of **exactly two files**:

| File | Purpose |
|------|---------|
| `Dockerfile-<name>` | Defines the modules compiled into the image |
| `.github/workflows/build_<name>.yaml` | Builds, tags, and publishes the image |

### Exact Steps

1. **Copy the Dockerfile**:
   ```bash
   cp Dockerfile-cloudflare Dockerfile-<name>
   ```
   Edit `Dockerfile-<name>` and adjust the `--with` lines for the module set you want.

2. **Copy the workflow**:
   ```bash
   cp .github/workflows/build_cloudflare-modules.yaml .github/workflows/build_<name>.yaml
   ```

3. **Update the new workflow** (required changes):
   - **Image name**: Replace all occurrences of `caddy-cloudflare-modules` with your new image name (e.g., `caddy-<name>`). This appears in:
     - `env:` variables (e.g., `IMAGE_NAME`)
     - GHCR repository path (`ghcr.io/smoochy/caddy-<name>`)
     - Docker Hub repository path (`smoochy84/caddy-<name>`)
     - Tag generation (`caddy-<tag>`, `caddy-v<tag>`)
   - **Dockerfile reference**: Change the `file:` input in the `docker/build-push-action` step from `Dockerfile-cloudflare` to `Dockerfile-<name>`.
   - **Path filters** in `on.push.paths` and `on.pull_request.paths`: Update to reference the new Dockerfile (e.g., `Dockerfile-<name>`).
   - **Workflow file path** in the `local_changes` step's `WORKFLOW_PATH` env var and the grep comparison.

4. **Add the new image to the README table** under "Available Images" so users can discover it.

5. **Commit and push** both files. The new workflow will trigger on push (since the workflow file itself changed) and publish the new image variant.

### What the New Variant Inherits

The copied workflow brings all existing behavior:
- Multi-arch build (`linux/amd64`, `linux/arm64`, `linux/arm/v7`)
- Dynamic addon discovery from its own Dockerfile
- Upstream change detection (Caddy base + each addon)
- OCI label emission for traceability
- Tagging strategy: `latest`, `caddy-<x.y.z>`, `caddy-v<x.y.z>`
- GHCR canonical + Docker Hub mirror (when secrets configured)
- Job summary with changelogs and version links
- Scheduled daily check at 03:00 UTC
- Manual `workflow_dispatch` with optional `force=true`

> **Reference**: [README → Adding a New Image Variant](https://github.com/smoochy/caddy-modules/blob/main/README.md#adding-a-new-image-variant)

---

## System Invariants

Any extension—whether adding a module or a new variant—must preserve the following invariants. They are enforced by the workflow architecture and should not be circumvented.

### 1. Dynamic Addon Discovery

**Invariant**: The workflow discovers addons by parsing `--with` lines in the Dockerfile at runtime. It does **not** maintain a hardcoded list.

- **Evidence**: The `decide` step in `build_cloudflare-modules.yaml` uses `grep -- '--with' <Dockerfile>` to build the addon list.
- **Implication**: Adding a `--with` line is sufficient for integration. Removing a line removes the addon from version tracking, labeling, and change detection automatically.
- **Do not**: Hardcode addon names or counts in the workflow.

### 2. OCI Label Tracking for Change Detection

**Invariant**: Every published image carries OCI labels that record the exact Caddy base digest, Caddy version, and every addon's module path + version. The next workflow run reads these labels from the currently published `:latest` image and compares them against freshly fetched upstream versions.

- **Key labels**:
  - `org.opencontainers.image.base.digest` — Caddy base image digest
  - `org.opencontainers.image.base.version` — Caddy version (e.g., `2.8.4`)
  - `org.opencontainers.image.addon.N.name` — Go module path (e.g., `github.com/caddy-dns/cloudflare`)
  - `org.opencontainers.image.addon.N.version` — Addon version (e.g., `v1.2.3`)
- **Implication**: Change detection is content-addressable. If you rebuild with the same inputs, the labels match and no publish occurs. If you pin an addon version in the Dockerfile (not currently done), the label would still reflect the version actually built.
- **Do not**: Strip or omit these labels in a custom build step.

### 3. Tagging Strategy

**Invariant**: Every image variant publishes exactly three tag shapes per build:

| Tag | Example | Purpose |
|-----|---------|---------|
| `latest` | `ghcr.io/smoochy/caddy-<name>:latest` | Rolling pointer to newest build |
| `caddy-<x.y.z>` | `ghcr.io/smoochy/caddy-<name>:caddy-2.8.4` | Reproducible pin to Caddy version |
| `caddy-v<x.y.z>` | `ghcr.io/smoochy/caddy-<name>:caddy-v2.8.4` | Matches upstream Caddy release tag (`v2.8.4`) |

- **Evidence**: `docker/metadata-action` configuration in the workflow generates `type=raw,value=latest` and `type=raw,value=caddy-${{ steps.decide.outputs.caddy_tag }}`. The `caddy-v` tag is derived from the same `caddy_tag` output.
- **Implication**: Users can pin by either `caddy-2.8.4` (semver) or `caddy-v2.8.4` (upstream tag shape). Both are immutable once published.
- **Do not**: Add additional tag shapes (e.g., date-based, commit SHA) without updating the metadata action and the tag assertion in PR validation.

### 4. Build Trigger Discipline

**Invariant**: Builds publish only when one of these conditions is true:
- Push to `main` that changes the variant's Dockerfile or `.dockerignore`
- Scheduled run detects an upstream change (Caddy base digest or any addon version)
- Manual `workflow_dispatch` with `force=true`

A push that changes **only the workflow file** does **not** force a publish (it runs validation only). This prevents workflow-only refactors from churning image tags.

- **Evidence**: The `decide` step computes `FORCED` and `CHANGED` booleans; `do_build = FORCED OR CHANGED`. `FORCED` is true for PR validation and for push with Dockerfile change. `CHANGED` is true when upstream differs from published labels.

### 5. Multi-Architecture Build

**Invariant**: All variants build for `linux/amd64`, `linux/arm64`, and `linux/arm/v7` via Docker Buildx.

- **Evidence**: `platforms: linux/amd64,linux/arm64,linux/arm/v7` in `docker/build-push-action`.
- **Implication**: New variants inherit this automatically. Do not restrict platforms unless the addon has architecture-specific constraints (document if so).

### 6. No Secrets in Images

**Invariant**: No secrets (API tokens, credentials) are baked into the image. Configuration is injected at runtime via environment variables (e.g., `CLOUDFLARE_API_TOKEN`).

- **Evidence**: The Dockerfile copies only the compiled binary. The workflow uses `GITHUB_TOKEN` for GHCR and GitHub API; Docker Hub secrets are only used at push time via `crane copy`.

---

## Related Pages

- [Architecture Overview](../architecture/overview.md) — High-level system context and data flow
- [Cloudflare Modules Integration](../integrations/cloudflare-modules.md) — Details of the four curated modules
- [Dockerfile Structure](../operations/dockerfile.md) — Multi-stage build and xcaddy mechanics
- [Build Pipeline Workflow](../workflows/build-pipeline.md) — Full workflow step-by-step, change detection logic, and validation commands
