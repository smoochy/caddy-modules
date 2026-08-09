---
type: Architecture
title: caddy-modules Architecture Overview
description: High-level architecture of the caddy-modules repository covering the multi-stage Dockerfile build, xcaddy compilation, and GitHub Actions workflow orchestration.
tags: [architecture, docker, caddy, xcaddy, github-actions]
openwiki:
  roles: [architecture]
  change_kinds: [lifecycle, configuration, integration]
  source_paths: [Dockerfile-cloudflare, .github/workflows/build_cloudflare-modules.yaml]
  symbols: [xcaddy, caddy:builder, caddy:latest]
  test_paths: []
  invariants: [Two-stage build produces minimal final image, All modules compiled at build time via xcaddy]
  validation_commands: [docker build -f Dockerfile-cloudflare .]
---

# Architecture Overview

## System Context

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Upstream       │     │  Build Pipeline  │     │  Registries     │
│  (Caddy, Addons)│────▶│  (GitHub Actions)│────▶│  (GHCR + Docker)│
└─────────────────┘     └──────────────────┘     └─────────────────┘
                               │
                               ▼
                        ┌──────────────────┐
                        │  OCI Image Labels│
                        │  (Traceability)  │
                        └──────────────────┘
```

## Core Components

### 1. Multi-Stage Dockerfile (`Dockerfile-cloudflare`)

**Build Stage** (`caddy:builder`):
- Uses `xcaddy` to compile Caddy with declared addons
- Each `--with` line adds a Go module at compile time
- Produces custom `/usr/bin/caddy` binary

**Final Stage** (`caddy:latest`):
- Copies only the compiled binary from builder
- Inherits Caddy's runtime configuration (entrypoint, user, etc.)
- Adds OCI labels for traceability

```dockerfile
# Build stage
FROM caddy:builder AS builder
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2

# Final stage
FROM caddy:latest
COPY --from=builder /usr/bin/caddy /usr/bin/caddy
```

### 2. GitHub Actions Workflow (`.github/workflows/build_cloudflare-modules.yaml`)

Single `build` job with these phases:

| Phase | Purpose | Key Steps |
|-------|---------|-----------|
| **Classify Changes** | Detect local file changes | `local_changes` step |
| **Setup** | QEMU, Buildx, GHCR login | `setup-qemu-action`, `setup-buildx-action`, `docker/login-action` |
| **Decide Build** | Fetch upstream versions, compare with published image | `decide` step (bash script) |
| **Metadata** | Generate tags and labels | `docker/metadata-action`, `addon_labels` step |
| **Build & Push** | Multi-arch build via Buildx | `docker/build-push-action` |
| **Mirror** | Copy tags to Docker Hub (optional) | `crane copy` |
| **Summary** | Write job summary | `cat /tmp/summary.md` |

### 3. Addon Discovery (Dynamic)

The workflow **parses the Dockerfile** at runtime to discover active addons:

```bash
grep -- '--with' Dockerfile-cloudflare | \
  while IFS= read -r line; do
    # Skip commented lines
    [[ "$line" =~ ^[[:space:]]*# ]] && continue
    # Extract addon: --with github.com/owner/repo
    [[ "$line" =~ --with[[:space:]]+([^[:space:]\\]+) ]]
  done
```

This means **adding a `--with` line automatically integrates it** into version fetching, labeling, and change detection.

## Data Flow

```mermaid
sequenceDiagram
    participant GH as GitHub Actions
    participant CR as crane (registry CLI)
    participant GHAPI as GitHub API
    participant DX as Docker Buildx
    participant GHCR as ghcr.io
    participant DH as Docker Hub

    GH->>CR: crane digest caddy:latest
    CR-->>GH: Base image digest
    GH->>CR: crane config caddy:latest
    CR-->>GH: Base image labels (version)
    GH->>GHAPI: GET /repos/{addon}/releases
    GHAPI-->>GH: Addon versions + release notes
    GH->>CR: crane config published image
    CR-->>GH: Current OCI labels (base digest, addon versions)
    GH->>GH: Compare digests/versions
    alt Build needed
        GH->>DX: docker buildx build --push
        DX->>GHCR: Push multi-arch image
        GH->>CR: crane copy to Docker Hub
        CR->>DH: Mirror tags
    end
    GH->>GH: Write job summary
```

## Adding a New Image Variant

Each variant is self-contained with exactly two files:

| File | Purpose |
|------|---------|
| `Dockerfile-<name>` | Declares modules via `--with` lines |
| `.github/workflows/build_<name>.yaml` | Builds, tags, publishes the image |

**Steps to add a variant:**
1. Copy `Dockerfile-cloudflare` → `Dockerfile-<name>`, adjust `--with` lines
2. Copy `.github/workflows/build_cloudflare-modules.yaml` → `.github/workflows/build_<name>.yaml`
3. Update hardcoded image name in workflow (`ghcr.io/smoochy/caddy-<name>`)
4. Update `file:` reference from `Dockerfile-cloudflare` to `Dockerfile-<name>`
5. Add to Available Images table in README.md

## Extension Points

| Extension | Location | Mechanism |
|-----------|----------|-----------|
| New Caddy module | `Dockerfile-<variant>` | Add `--with github.com/owner/repo` line |
| New image variant | Repository root | Two-file pattern above |
| Build trigger | Workflow `on:` section | Push paths, schedule, workflow_dispatch |
| Registry target | Workflow `registries` step | Add secrets, extend mirror logic |
| OCI label | Workflow `addon_labels` / `build-push` | Dynamic label generation from discovered addons |

## Invariants

- **Minimal final image**: Only the custom Caddy binary is copied; no build tools in final layer
- **Reproducible builds**: Same Dockerfile + same base digest + same addon versions = identical binary
- **Dynamic addon tracking**: Workflow discovers addons from Dockerfile, no hardcoded list
- **Multi-arch by default**: `linux/amd64`, `linux/arm64`, `linux/arm/v7`