# caddy-modules

[![README Style](https://img.shields.io/badge/README%20style-standard-2ea44f)](https://github.com/RichardLitt/standard-readme)
[![Build](https://github.com/smoochy/caddy-modules/actions/workflows/build_cloudflare-modules.yaml/badge.svg)](https://github.com/smoochy/caddy-modules/actions)

[![Buy me uptime](https://img.shields.io/badge/Buy%20me%20uptime%20%F0%9F%96%A5%EF%B8%8F-smoochy84-E9C46A?logo=buymeacoffee&logoColor=000000)](https://www.buymeacoffee.com/smoochy84)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-smoochy-7CC6FE?logo=ko-fi&logoColor=000000)](https://ko-fi.com/smoochy)

> Custom [Caddy](https://github.com/caddyserver/caddy) images with curated module sets, published to GHCR with a public Docker Hub mirror and rebuilt automatically when upstream components change.

This repository provides ready-to-use Caddy container images for self-hosted
and infrastructure setups that need more than the official default image. Each
image variant is built with a defined module set, tracked against upstream
changes, and published with reproducible tags plus traceable metadata.

Registries:

- GHCR (canonical): `ghcr.io/smoochy/caddy-cloudflare-modules`
- Docker Hub (public mirror): `smoochy84/caddy-cloudflare-modules`

If this project saves you time or helps your setup, you can support ongoing
maintenance for a project I maintain in my spare time via Ko-fi or Buy Me a
Coffee.

## Table of Contents

- [Background](#background)
- [Available Images](#available-images)
- [Cloudflare Image](#cloudflare-image)
- [When Builds Run](#when-builds-run)
- [Dockerfile](#dockerfile)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Job Summary](#job-summary)
- [Image Metadata](#image-metadata)
- [Image Tags](#image-tags)
- [Install](#install)
- [Usage](#usage)
- [Transparency](#transparency)
- [Adding More Addons](#adding-more-addons)
- [Adding a New Image Variant](#adding-a-new-image-variant)
- [Security](#security)
- [Maintainers](#maintainers)
- [Contributing](#contributing)
- [License](#license)

## Background

The official Caddy image is intentionally minimal. This repository provides
maintained custom image variants for setups that rely on additional modules and
want a reproducible way to stay current with upstream changes.

## Available Images

| Image                                                                                                          | Dockerfile              | Workflow                        | Description                   |
| -------------------------------------------------------------------------------------------------------------- | ----------------------- | ------------------------------- | ----------------------------- |
| [`caddy-cloudflare-modules`](https://github.com/smoochy/caddy-modules/pkgs/container/caddy-cloudflare-modules) | `Dockerfile-cloudflare` | `build_cloudflare-modules.yaml` | Cloudflare DNS and IP modules |

Additional variants can be added at any time. See
[Adding a New Image Variant](#adding-a-new-image-variant).

## Cloudflare Image

### caddy-cloudflare-modules

| Addon                                                                                     | Purpose                                                  |
| ----------------------------------------------------------------------------------------- | -------------------------------------------------------- |
| [`caddy-dns/cloudflare`](https://github.com/caddy-dns/cloudflare)                         | DNS-01 ACME challenge provider for Cloudflare            |
| [`WeidiDeng/caddy-cloudflare-ip`](https://github.com/WeidiDeng/caddy-cloudflare-ip)       | Provides the real client IP when behind Cloudflare proxy |
| [`fvbommel/caddy-combine-ip-ranges`](https://github.com/fvbommel/caddy-combine-ip-ranges) | Combines multiple IP range sources for trusted proxies   |
| [`lucaslorentz/caddy-docker-proxy`](https://github.com/lucaslorentz/caddy-docker-proxy)   | Automatic Caddy configuration via Docker labels          |

- Builds the custom Docker image `caddy-cloudflare-modules`
- Publishes the image to:
  - GHCR (canonical):
    - `ghcr.io/smoochy/caddy-cloudflare-modules:latest`
    - `ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<x.y.z>`
  - Docker Hub (public mirror):
    - `smoochy84/caddy-cloudflare-modules:latest`
    - `smoochy84/caddy-cloudflare-modules:caddy-<x.y.z>`
- Tracks upstream updates and rebuilds only when needed

## When Builds Run

The workflow is triggered in three ways:

1. Push to `main`, but only when one of these files changes:
   - `Dockerfile*`
   - `.dockerignore`
   - `.github/workflows/build_*.yaml`

   This prevents rebuilds for documentation-only changes such as `README.md`.

2. Scheduled run:
   - Runs a daily check at 03:00 UTC for upstream changes such as the Caddy
     base digest and addon versions
   - Builds only if something changed

3. Manual run with `workflow_dispatch`:
   - Optional `force=true` input to rebuild even if nothing changed

## Dockerfile

The `Dockerfile`:

- Uses a two-stage build
- Uses `caddy:builder` and `xcaddy` to compile Caddy with addons
- Uses `caddy:latest` as the final base image
- Supports multi-arch builds for:
  - `linux/amd64`
  - `linux/arm64`
- Declares addons via `--with` arguments

Commented-out `--with` lines are ignored by both the build and the workflow.

## GitHub Actions Workflow

`build_*.yaml` does the following:

1. Logs into GHCR and sets up Buildx plus QEMU.
2. Parses the Dockerfile to discover active addons from `--with` lines.
3. Fetches upstream versions using authenticated GitHub API calls.
4. Reads metadata from the currently published image when it exists.
5. Compares the Caddy base digest and addon versions.
6. Builds once, pushes to GHCR, and mirrors the published tags to Docker Hub
   when Docker Hub secrets are configured.
7. Publishes only when:
   - a push-triggered run happens
   - an upstream change is detected
   - a manual run is forced

## Job Summary

Every workflow run writes a summary that includes:

- The reason the build ran
- Which upstream component changed
- The Caddy base digest and version
- All addon versions with direct links
- Changelogs from upstream release notes where available
- The published image tags
- Whether the Docker Hub mirror was updated or skipped

## Image Metadata

Each published image includes OCI labels used for traceability and change
detection, for example:

- `org.opencontainers.image.base.tag`
- `org.opencontainers.image.base.digest`
- `org.opencontainers.image.base.version`
- `org.opencontainers.image.cloudflare.version`
- `org.opencontainers.image.addon.N.name`
- `org.opencontainers.image.addon.N.version`

## Image Tags

This image is published with:

- `latest`: always points to the newest build
- `caddy-<x.y.z>`: matches the Caddy base version used at build time and is
  useful for reproducible deployments pinned to a specific Caddy release

## Install

Pull the published image from GHCR (canonical) or Docker Hub (public mirror):

```bash
docker pull ghcr.io/smoochy/caddy-cloudflare-modules:latest
```

```bash
docker pull smoochy84/caddy-cloudflare-modules:latest
```

For reproducible deployments, pin both a version tag and digest:

```text
ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<x.y.z>@sha256:<digest>
```

The digest of every published image is visible in the
[GitHub Actions Job Summary](https://github.com/smoochy/caddy-modules/actions)
and on the
[GHCR package page](https://github.com/smoochy/caddy-modules/pkgs/container/caddy-cloudflare-modules).
The same version tags are mirrored to Docker Hub at
[smoochy84/caddy-cloudflare-modules](https://hub.docker.com/r/smoochy84/caddy-cloudflare-modules).

## Usage

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

## Transparency

The code, documentation, and related project materials in this repository were
created and refined with AI assistance. All generated output was reviewed and
adapted before publication.

## Adding More Addons

To include another Caddy module, add a `--with` line to `Dockerfile-cloudflare`:

```dockerfile
RUN xcaddy build \
    --with github.com/caddy-dns/cloudflare \
    --with github.com/WeidiDeng/caddy-cloudflare-ip \
    --with github.com/fvbommel/caddy-combine-ip-ranges \
    --with github.com/lucaslorentz/caddy-docker-proxy/v2 \
    --with github.com/your-org/your-caddy-addon
```

The workflow automatically picks up new `--with` lines and:

- Fetches the latest version
- Includes it in the Job Summary and OCI labels
- Tracks it for future upstream change detection

## Adding a New Image Variant

Each image variant is self-contained and consists of exactly two files:

| File                                  | Purpose                                     |
| ------------------------------------- | ------------------------------------------- |
| `Dockerfile-<name>`                   | Defines the modules compiled into the image |
| `.github/workflows/build_<name>.yaml` | Builds, tags, and publishes the image       |

To add a new variant:

1. Copy `Dockerfile-cloudflare` to `Dockerfile-<name>` and adjust the `--with` lines.
2. Copy `.github/workflows/build_cloudflare-modules.yaml` to `.github/workflows/build_<name>.yaml`.
3. Update the hardcoded image name in the new workflow.
4. Update the `file:` reference from `Dockerfile-cloudflare` to `Dockerfile-<name>`.
5. Add the new image to the table above.

## Security

- No secrets are baked into the image
- GitHub Actions uses the built-in `GITHUB_TOKEN` for GHCR authentication and
  GitHub API calls
- Optional Docker Hub publishing uses `DOCKERHUB_USERNAME` and
  `DOCKERHUB_TOKEN` repository secrets
- Only release metadata is queried from upstream projects

## Maintainers

- smoochy

## Contributing

Issues and pull requests are welcome. Keep image, workflow, and documentation
changes aligned so the published image behavior stays obvious from the README.

## License

[MIT](./LICENSE) 2026 [smoochy](https://github.com/smoochy)
