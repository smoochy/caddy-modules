---
type: Operations
title: Dockerfile Structure
description: Documentation of the Dockerfile-cloudflare multi-stage build, xcaddy compilation, and addon declaration pattern.
tags: [operations, docker, dockerfile, xcaddy, caddy, multi-stage-build]
openwiki:
  roles: [operations, delivery]
  change_kinds: [configuration, integration]
  source_paths: [Dockerfile-cloudflare]
  symbols: [xcaddy, caddy:builder, caddy:latest, --with]
  test_paths: []
  invariants: [Two-stage build: builder compiles, final only copies binary, Addons declared via --with lines, Commented --with lines are ignored]
  validation_commands: [docker build -f Dockerfile-cloudflare .]
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

**Base image**: `caddy:builder` - Caddy's official builder image with Go toolchain and `xcaddy` pre-installed.

**Command**: `xcaddy build` with multiple `--with` flags.

### xcaddy Build Process

`xcaddy` is Caddy's custom builder that:
1. Creates a temporary Go module
2. Adds each `--with` module as a `replace` directive
3. Runs `go build` to compile Caddy with the modules statically linked
4. Outputs binary at `/usr/bin/caddy`

### Addon Declaration Pattern

Each `--with` line follows Go module path syntax:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2
```

**Key points:**
- **Order doesn't matter** for compilation
- **Version not pinned** - `xcaddy` fetches latest compatible version at build time
- **Go version suffix** (`/v2`) is part of module path for major versions > v1
- **Commented lines ignored** - lines starting with `#` are skipped by both build and workflow parser

## Stage 2: Final (`caddy:latest`)

**Base image**: `caddy:latest` - Official Caddy runtime image (minimal, non-root user, proper entrypoint).

**Operation**: `COPY --from=builder /usr/bin/caddy /usr/bin/caddy`

**Result**: Final image contains only the custom Caddy binary + Caddy's runtime environment. No Go toolchain, no build artifacts.

## Labels

Base labels in Dockerfile (supplemented by workflow at build time):
- `org.opencontainers.image.title`
- `org.opencontainers.image.description`
- `org.opencontainers.image.url`
- `org.opencontainers.image.documentation`
- `org.opencontainers.image.licenses`
- `org.opencontainers.image.source`

Workflow adds at build time:
- Base image digest, tag, version
- Dynamic addon labels (name + version per module)

## .dockerignore

```text
.git
.github
*.md
LICENSE
.gitignore
```

Excludes documentation and Git metadata from build context (speeds up builds, reduces context size).

## Adding a New Module

**Edit `Dockerfile-cloudflare`**, add a new `--with` line:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2 \
    --with github.com/your-org/your-caddy-module
```

**Workflow automatically:**
1. Discovers new `--with` line via `grep`
2. Fetches version from GitHub API
3. Adds OCI labels
4. Includes in job summary/changelog
5. Tracks for future change detection

**No workflow changes needed.**

## Temporarily Disabling a Module

Comment out the line:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    # --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2
```

Both the build and the workflow parser skip commented lines.

## Multi-Arch Support

The Dockerfile itself is architecture-agnostic. Multi-arch is handled by **GitHub Actions Buildx** in the workflow:
- `--platform linux/amd64,linux/arm64,linux/arm/v7`
- Buildx uses QEMU emulation for non-native arches
- Produces multi-arch manifest with platform-specific layers

## Build Validation

```bash
# Local build (single arch, no push)
docker build -f Dockerfile-cloudflare .

# Verify binary works
docker run --rm <image-id> version

# List modules in binary
docker run --rm <image-id> list-modules
```

## Extending for New Variants

When creating `Dockerfile-<variant>`:
1. Copy this file
2. Modify `--with` lines for desired modules
3. Keep the two-stage structure
4. Keep base labels (workflow will add dynamic ones)
5. Update workflow `file:` reference to new Dockerfile