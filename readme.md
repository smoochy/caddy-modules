# caddy-modules

[![standard-readme compliant](https://img.shields.io/badge/readme%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/RichardLitt/standard-readme)
[![Build](https://github.com/smoochy/caddy-modules/actions/workflows/build_cloudflare-modules.yaml/badge.svg)](https://github.com/smoochy/caddy-modules/actions)

> A collection of custom **[Caddy](https://github.com/caddyserver/caddy)** images with curated module sets, automatically rebuilt when upstream components change.

This repository builds and publishes custom Docker images based on `caddy:latest`,
each compiled with a specific set of modules via
[`xcaddy`](https://github.com/caddyserver/xcaddy).

Each image variant has its own `Dockerfile-*` and
`.github/workflows/build_*.yaml`. The CI rebuilds images **only when necessary**
and writes a Job Summary explaining _why_ a build happened (including links to
upstream release notes).

---

## Table of Contents

- [caddy-modules](#caddy-modules)
  - [Table of Contents](#table-of-contents)
  - [Background](#background)
  - [Available images](#available-images)
  - [Cloudflare image](#cloudflare-image)
  - [When builds run](#when-builds-run)
  - [Dockerfile](#dockerfile)
  - [GitHub Actions workflow](#github-actions-workflow)
  - [Job Summary](#job-summary)
  - [Image metadata](#image-metadata)
  - [Image tags](#image-tags)
  - [Adding more addons](#adding-more-addons)
  - [Adding a new image variant](#adding-a-new-image-variant)
  - [Usage](#usage)
  - [Security](#security)
  - [Maintainers](#maintainers)
  - [License](#license)

---

## Background

The official Caddy Docker image does not include third-party modules. This
repository provides a framework for building and maintaining multiple custom
Caddy images, each compiled with a specific set of modules, and keeps them
up to date automatically.

---

## Available images

| Image                                                                                                          | Dockerfile              | Workflow                        | Description                 |
| -------------------------------------------------------------------------------------------------------------- | ----------------------- | ------------------------------- | --------------------------- |
| [`caddy-cloudflare-modules`](https://github.com/smoochy/caddy-modules/pkgs/container/caddy-cloudflare-modules) | `Dockerfile-cloudflare` | `build_cloudflare-modules.yaml` | Cloudflare DNS + IP modules |

> Additional variants can be added at any time — see [Adding a new image variant](#adding-a-new-image-variant).

### caddy-cloudflare-modules

| Addon                                                                                     | Purpose                                                  |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| [`caddy-dns/cloudflare`](https://github.com/caddy-dns/cloudflare)                         | DNS-01 ACME challenge provider for Cloudflare            |
| [`WeidiDeng/caddy-cloudflare-ip`](https://github.com/WeidiDeng/caddy-cloudflare-ip)       | Provides the real client IP when behind Cloudflare proxy |
| [`fvbommel/caddy-combine-ip-ranges`](https://github.com/fvbommel/caddy-combine-ip-ranges) | Combines multiple IP range sources for trusted proxies   |
| [`lucaslorentz/caddy-docker-proxy`](https://github.com/lucaslorentz/caddy-docker-proxy)   | Automatic Caddy configuration via Docker labels          |

---

## Cloudflare image

- Builds a custom Docker image `caddy-cloudflare-modules`:
  - Base: `caddy:latest`
  - Compiled with:
    - `github.com/caddy-dns/cloudflare` — DNS-01 ACME challenge provider
    - `github.com/WeidiDeng/caddy-cloudflare-ip` — real client IP behind Cloudflare
    - `github.com/fvbommel/caddy-combine-ip-ranges` — combines trusted IP range sources
    - `github.com/lucaslorentz/caddy-docker-proxy` — automatic config via Docker labels
- Publishes the image to **GitHub Container Registry (GHCR)**:
  - `ghcr.io/smoochy/caddy-cloudflare-modules:latest`
  - `ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<x.y.z>`
- Tracks upstream updates and rebuilds only when needed.

---

## When builds run

The workflow is triggered in three ways:

1. **Push to `main`**, but _only_ when one of these files changes:
   - `Dockerfile*`
   - `.dockerignore`
   - `.github/workflows/build_*.yaml`

   This prevents rebuilds for documentation-only changes (e.g. `README.md`).

2. **Scheduled run** (cron)
   - Runs a daily check at 03:00 UTC for upstream changes (Caddy base digest,
     addon versions).
   - Builds only if something changed.

3. **Manual run** (`workflow_dispatch`)
   - Optional `force=true` input to rebuild even if nothing changed.

---

## Dockerfile

The `Dockerfile`:

- Uses a two-stage build:
  - **Build stage**: `caddy:builder` + `xcaddy` to compile Caddy with addons
  - **Final stage**: `caddy:latest`, copies in the compiled binary
- Supports multi-arch builds for:
  - `linux/amd64`
  - `linux/arm64`
- Addons are declared via `--with` arguments. Commented-out lines are ignored
  by both the build and the workflow.

---

## GitHub Actions workflow

`build_*.yaml` does the following:

1. Logs into GHCR and sets up Buildx + QEMU
2. Parses the Dockerfile to discover all active addons (`--with` lines)
3. Fetches upstream versions using **authenticated GitHub API calls**:
   - Release tag → git tag → latest commit SHA (7-char short SHA as fallback)
4. Reads the metadata from the currently published image (if it exists)
5. Compares the Caddy base digest and all addon versions
6. Builds and pushes only when:
   - a push-triggered run happens (filtered by `paths`), or
   - an upstream change is detected, or
   - a manual run is forced

---

## Job Summary

Every workflow run writes a summary that includes:

- The reason the build ran (push, schedule, manual, forced)
- Which upstream component changed (base, addon versions)
- The Caddy base digest and version
- All addon versions with direct links:
  - Release link (if a release tag exists)
  - `[repo]` + `[commit]` links (if only a commit SHA is available)
- Changelogs from upstream release notes where available
- The published image tags

---

## Image metadata

Each published image includes OCI labels used for traceability and change
detection, for example:

- `org.opencontainers.image.base.tag`
- `org.opencontainers.image.base.digest`
- `org.opencontainers.image.base.version`
- `org.opencontainers.image.cloudflare.version`
- `org.opencontainers.image.addon.N.name`
- `org.opencontainers.image.addon.N.version`

---

## Image tags

This image is published with:

- `latest`
  - Always points to the newest build.
- `caddy-<x.y.z>`
  - Matches the Caddy base version used at build time (e.g. `caddy-2.9.1`).
  - Useful for reproducible deployments pinned to a specific Caddy release.

---

---

## Adding more addons

To include any additional Caddy module, add a `--with` line to the `Dockerfile-cloudflare`:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy \
    --with github.com/your-org/your-caddy-addon
```

The workflow automatically picks up new `--with` lines and:

- Fetches the latest version (release, tag, or commit SHA)
- Includes it in the Job Summary and OCI labels
- Tracks it for future upstream change detection

Commented-out `--with` lines are fully ignored.

---

## Adding a new image variant

Each image variant is self-contained and consists of exactly two files:

| File                                  | Purpose                                     |
| ------------------------------------- | ------------------------------------------- |
| `Dockerfile-<name>`                   | Defines the modules compiled into the image |
| `.github/workflows/build_<name>.yaml` | Builds, tags, and publishes the image       |

To add a new variant:

1. Copy `Dockerfile-cloudflare` → `Dockerfile-<name>` and adjust the `--with` lines.
2. Copy `.github/workflows/build_cloudflare-modules.yaml` → `.github/workflows/build_<name>.yaml`.
3. In the new workflow file, update the hardcoded image name (three occurrences of `ghcr.io/smoochy/caddy-cloudflare-modules`) to `ghcr.io/smoochy/caddy-<name>`.
4. Update the `file:` reference from `Dockerfile-cloudflare` to `Dockerfile-<name>`.
5. Add a row to the [Available images](#available-images) table above.

---

## Usage

```bash
docker pull ghcr.io/smoochy/caddy-cloudflare-modules:latest
```

> [!TIP]
> For reproducible and tamper-proof deployments it is recommended to pin the
> image using both a version tag **and** its digest:
>
> ```text
> ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<x.y.z>@sha256:<digest>
> ```
>
> This guarantees that the exact same image is pulled every time, even if the
> tag is later overwritten. The digest of every published image is visible in
> the [GitHub Actions Job Summary](https://github.com/smoochy/caddy-modules/actions)
> and in the
> [GHCR package page](https://github.com/smoochy/caddy-modules/pkgs/container/caddy-cloudflare-modules).

Example `docker-compose.yml`:

```yaml
services:
  caddy:
    image: ghcr.io/smoochy/caddy-cloudflare-modules:latest
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - caddy_data:/data
      - caddy_config:/config
    environment:
      - CLOUDFLARE_API_TOKEN=${CLOUDFLARE_API_TOKEN}

volumes:
  caddy_data:
  caddy_config:
```

Example `Caddyfile`:

```caddyfile
{
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

example.com {
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }

  respond "Hello from Caddy with Cloudflare addons!"
}
```

---

## Security

- No secrets are baked into the image.
- GitHub Actions uses the built-in `GITHUB_TOKEN` for GHCR authentication and
  GitHub API calls (avoids unauthenticated rate limits).
- Only release metadata is queried from upstream projects.

---

## Maintainers

- smoochy

---

## License

MIT
