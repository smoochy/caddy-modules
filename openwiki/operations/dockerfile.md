---
type: Operations
title: Dockerfile Structure
description: Documentation of the Dockerfile-cloudflare multi-stage build, xcaddy compilation, addon declaration pattern, and label strategy.
tags: [operations, docker, dockerfile, xcaddy, caddy, multi-stage-build]
openwiki:
  roles: [operations, delivery]
  change_kinds: [configuration, integration]
  source_paths: [Dockerfile-cloudflare]
  symbols: [xcaddy, caddy:builder, caddy:latest, --with]
  test_paths: []
  invariants: [Two-stage build: builder compiles, final only copies binary, Addons declared via --with lines, Commented --with lines are ignored by both build and workflow parser]
  validation_commands: [docker build -f Dockerfile-cloudflare .]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-715dace563ef484b6e8bd1e2
    resource: repo://.dockerignore
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-42ef8192a0f869c58499e372
    resource: repo://Dockerfile-cloudflare
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---

# Dockerfile Structure

## File: `Dockerfile-cloudflare`

```dockerfile
# Build stage
FROM caddy:builder AS builder

# Build Caddy with the Cloudflare DNS module
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2

# Final stage
FROM caddy:latest

# Copy the custom-built Caddy binary
COPY --from=builder /usr/bin/caddy /usr/bin/caddy

LABEL org.opencontainers.image.title="Caddy with additional modules"
LABEL org.opencontainers.image.description="Caddy web server image with several additional modules baked in"
LABEL org.opencontainers.image.url=https://caddyserver.com
LABEL org.opencontainers.image.documentation=https://caddyserver.com/docs
LABEL org.opencontainers.image.licenses=MIT
LABEL org.opencontainers.image.source="https://github.com/smoochy/caddy-modules"
```

## Stage 1: Builder (`caddy:builder`)

**Base image**: `caddy:builder` — Caddy's official builder image that includes the Go toolchain and `xcaddy` pre-installed.

**Command**: `xcaddy build` with multiple `--with` flags specifying Go module paths.

### xcaddy Build Process

`xcaddy` is Caddy's custom build tool that:

1. Creates a temporary Go module in a working directory
2. Adds each `--with` module as a `replace` directive in `go.mod`, pinning it to the module path provided
3. Runs `go build` to compile Caddy with the modules statically linked into a single binary
4. Outputs the binary at `/usr/bin/caddy` inside the builder image

**Key behavior**:
- The `--with` flag accepts a Go module import path (e.g., `github.com/caddy-dns/cloudflare`)
- No version is pinned in the Dockerfile — `xcaddy` fetches the latest compatible version at build time based on the Caddy version in the builder image
- Go major version suffixes (`/v2`, `/v3`) are part of the module path per Go modules convention
- The build runs inside the `caddy:builder` image which matches the Caddy version that will be used at runtime

### Addon Declaration Pattern

Each `--with` line follows Go module path syntax:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2
```

**Rules**:
- **Order doesn't matter** for compilation — all modules are linked into the same binary
- **Version not pinned** — `xcaddy` resolves the latest compatible version at build time against the Caddy version in the builder image
- **Go version suffix** (`/v2`) is required for modules at v2+ and is part of the import path
- **Commented lines ignored** — lines starting with `#` (after optional whitespace) are skipped by both the Docker build (shell comment) and the workflow parser (explicit filter). The workflow uses `grep -- '--with' Dockerfile-cloudflare` and then filters out lines matching `^[[:space:]]*#` before extracting module paths.

## Stage 2: Final (`caddy:latest`)

**Base image**: `caddy:latest` — Official Caddy runtime image (minimal, runs as non-root user, proper entrypoint, no Go toolchain).

**Operation**: `COPY --from=builder /usr/bin/caddy /usr/bin/caddy`

**Result**: Final image contains only the custom Caddy binary plus Caddy's runtime environment. No Go toolchain, no build artifacts, no source code. The image size stays close to the base `caddy:latest` size.

## Labels

### Base Labels (in Dockerfile)

These static labels are baked into the image at build time by the Dockerfile:

| Label | Value |
|-------|-------|
| `org.opencontainers.image.title` | `Caddy with additional modules` |
| `org.opencontainers.image.description` | `Caddy web server image with several additional modules baked in` |
| `org.opencontainers.image.url` | `https://caddyserver.com` |
| `org.opencontainers.image.documentation` | `https://caddyserver.com/docs` |
| `org.opencontainers.image.licenses` | `MIT` |
| `org.opencontainers.image.source` | `https://github.com/smoochy/caddy-modules` |

### Workflow-Added Labels (at build time)

The GitHub Actions workflow (`.github/workflows/build_cloudflare-modules.yaml`) injects additional labels during `docker build-push-action` using the `labels:` input. These include:

**Base image metadata**:
- `org.opencontainers.image.base.name` = `caddy:latest`
- `org.opencontainers.image.base.tag` = Caddy version tag (e.g., `2.8.4`)
- `org.opencontainers.image.base.digest` = Full digest of the `caddy:latest` image (e.g., `sha256:abc123...`)
- `org.opencontainers.image.base.version` = Caddy version tag

**Per-addon metadata** (one set per `--with` module, indexed by position):
- `org.opencontainers.image.addon.{i}.name` = Module import path (e.g., `github.com/caddy-dns/cloudflare`)
- `org.opencontainers.image.addon.{i}.version` = Resolved version from GitHub (release tag, tag, or commit SHA)

**Legacy/specific**:
- `org.opencontainers.image.cloudflare.version` = Version of the first addon (Cloudflare DNS module) for backward compatibility

**Description override**:
- `org.opencontainers.image.description` = `Caddy with Cloudflare modules (auto rebuild on upstream updates)`

The workflow parses the `--with` lines from `Dockerfile-cloudflare` (excluding commented lines), queries GitHub for each module's latest release/tag/commit, and emits these labels dynamically.

## .dockerignore

```text
.git
.github
*.md
LICENSE
.gitignore
```

**Purpose**: Excludes repository metadata, documentation, and license files from the Docker build context. This keeps the context small and prevents unnecessary cache invalidation. The workflow also triggers on changes to `.dockerignore` (see `paths:` filter in the workflow), so modifying it will force a rebuild.

## Adding a New Module

To include another Caddy module:

1. **Edit `Dockerfile-cloudflare`** and append a new `--with` line with the Go module path:
   ```dockerfile
   RUN xcaddy build \
       --with github.com/caddy-dns/cloudflare \
       --with github.com/WeidiDeng/caddy-cloudflare-ip \
       --with github.com/fvbommel/caddy-combine-ip-ranges \
       --with github.com/lucaslorentz/caddy-docker-proxy/v2 \
       --with github.com/owner/new-module
   ```
2. **Commit and push** to `main` — the workflow triggers on changes to `Dockerfile-cloudflare` and will:
   - Parse the new `--with` line
   - Fetch the module's latest version from GitHub
   - Build the image with the new module linked in
   - Publish with updated addon labels

To temporarily disable a module, comment out its `--with` line (prefix with `#`). The build and workflow parser will both ignore it.
