# caddy-modules

[![README Style](https://img.shields.io/badge/README%20style-standard-2ea44f)](https://github.com/RichardLitt/standard-readme)
[![Build](https://github.com/smoochy/caddy-modules/actions/workflows/build_cloudflare-modules.yaml/badge.svg)](https://github.com/smoochy/caddy-modules/actions)

[![Coindrop](https://img.shields.io/badge/Tip%20me%20crypto-smoochy-FFB655?logo=data%3Aimage%2Fsvg%2Bxml%3Bbase64%2CPHN2ZyB4bWxucz0iaHR0cDovL3d3dy53My5vcmcvMjAwMC9zdmciIHZpZXdCb3g9IjAgMCA1MTIgNTEyIj48ZyB0cmFuc2Zvcm09InRyYW5zbGF0ZSgwIDUxMikgc2NhbGUoLjEgLS4xKSIgZmlsbD0iIzAwMCI%2BPHBhdGggZD0iTTE5NjIgNTAwOCBjMCAtNDAgMCAtNzkgLTEgLTg1IC0xIC05IC0yNSAtMTMgLTg2IC0xMyBsLTg1IDAgMCAtODUgYzAgLTYxIDQgLTg1IDEzIC04NiA2IDAgNDUgMCA4NSAxIDgyIDEgNzUgMTIgNzQgLTEwNyBsLTEgLTYzIDg3IDAgODcgMCAwIDgzIC0xIDgyIC04NiAzIC04NSAzIDEgODIgMSA4MiA4MyAzIGM2OCAyIDgyIDAgODMgLTEzIDAgLTggMSAtNDYgMiAtODUgbDAgLTcwIDg3IDAgODYgMCAtMSA4MyAtMSA4MiAtODYgMyAtODYgMyAxIDgyIDIgODIgLTg3IDMgLTg2IDMgMCAtNzN6Ii8%2BPHBhdGggZD0iTTM5MzIgNDgyMCBjLTQgLTMgLTcgLTQyIC02IC04NiBsMCAtODIgLTc5IDIgYy00MyAxIC04MiAtMSAtODYgLTQgLTQgLTMgLTcgLTQxIC03IC04NSBsMSAtODAgNzAgLTIgYzEwNCAtMyAxMDEgLTUgMTAwIDgzIC0xIDg5IDAgOTAgMTAxIDg5IGw3MCAtMiAtMiA4MSBjLTEgNDUgLTIgODQgLTMgODcgLTEgNiAtMTUwIDYgLTE1OSAtMXoiLz48cGF0aCBkPSJNNDEwMiA0NjQ1IGMtMyAtNSAtNiAtNDQgLTcgLTg3IGwwIC03NiA4NSAwIDg2IDEgMCA4NSAtMSA4NSAtNzggMSBjLTQ0IDEgLTgyIC0zIC04NSAtOXoiLz48cGF0aCBkPSJNMjg0MCA0NTUzIGMtMzAzIC02MyAtNTUyIC0zMTAgLTYwNCAtNTk5IC0xOCAtOTkgLTEyIC05MSAtNjkgLTk4IC0xNDkgLTIwIC0xMzAgLTI1IC0xNzUgNDEgLTc0IDEwOCAtMTg0IDIwMSAtMzEyIDI2MSAtNjkgMzMgLTE5NyA2MyAtMjc3IDY1IC02NCAyIC03NiAtMSAtOTUgLTIxIC0yMyAtMjIgLTIzIC0yNiAtMjYgLTMyNiBsLTMgLTMwMyAtNDIgLTIzIGMtMjYwIC0xNDMgLTUyMiAtMzg5IC02ODIgLTY0MiBsLTMwIC00NyAtMTkwIC0xIGMtMTA0IDAgLTIwMiAtMyAtMjE3IC03IC00MCAtMTEgLTk2IC02OSAtMTA4IC0xMTEgLTggLTI1IC0xMCAtMjE0IC04IC01ODMgbDMgLTU0NSAyNyAtNDEgYzQ1IC02OSA3MyAtNzYgMzAyIC03OCAxNTUgLTEgMjAxIC00IDIwNCAtMTQgNiAtMTggMTA4IC0xNjcgMTQxIC0yMDQgMTE4IC0xMzcgMjA3IC0yMjQgMzIwIC0zMTEgNDIgLTMyIDEwMCAtNzIgMTI4IC04OCAyOSAtMTcgNTYgLTM1IDYwIC00MSA0IC02IDcgLTE1OCA4IC0zMzcgMCAtMzIyIDEgLTMyNSAyNCAtMzY2IDE0IC0yNSA0MCAtNTEgNjUgLTY1IDQwIC0yMyA0NyAtMjMgMzIxIC0yNCAxNTQgMCAyOTYgMyAzMTYgOCA0OSAxMiAxMDUgNjYgMTE4IDExNCA2IDIxIDExIDEwOSAxMSAxOTYgMCAxNDYgMSAxNTggMTggMTUzIDkgLTMgNDIgLTggNzIgLTExIDMwIC0zIDYyIC04IDcwIC0xMCAzNyAtMTAgMjA5IC0xOSAzNTUgLTE5IDIzMCAwIDMzNiAxMSA1ODkgNjAgNjYgMTIgMjM2IDYwIDMxOSA4OSA1NCAxOSAxMDEgMzUgMTA1IDM1IDQgMCA3IC0xMDcgNyAtMjM4IDAgLTIyNiAxIC0yMzkgMjIgLTI4MyAxNiAtMzEgMzYgLTUzIDY1IC02OSA0MyAtMjQgNDUgLTI0IDMzOCAtMjQgMzM1IDAgMzQ2IDIgMzk3IDc3IGwyOCA0MiAxIDU1MSAxIDU1MSA0NCA1OSBjMTQ0IDE5MCAyNDAgNDE1IDI4MyA2NjEgNCAyMiAxMyAyNCA0OSAxMCA0NSAtMTkgNTAgLTM4IDUyIC0yMjIgMSAtMTM3IDMgLTE1NCAyNCAtMTkxIDMwIC01MyA4NCAtODQgMTUzIC04OCAyOSAtMiA1OSAtMSA2NiAyIDggMyAxMiAzMCAxMiA4NyBsMCA4MiAtNDIgLTEgLTQzIC0xIC0xIDE1NSBjLTEgMTM0IC00IDE2MiAtMjEgMjAyIC0zNiA4MCAtOTYgMTI4IC0xOTIgMTUyIGwtNDMgMTIgLTQgNzQgYy0yIDQxIC02IDkxIC0xMCAxMTAgLTMgMTkgLTggNDYgLTExIDYwIC0zMiAxODEgLTEyNiAzOTQgLTI1MSA1NzAgLTU2IDc4IC0yNDAgMjc2IC0zMTcgMzQwIC0xMDUgODcgLTMwMSAyMTQgLTQwOCAyNjQgbC0zOSAxOCA4IDcxIGMxNCAxNDEgNiAyMzIgLTMxIDM0NSAtODkgMjY0IC0zMDUgNDU2IC01ODQgNTE4IC01NCAxMiAtMjI4IDEwIC0yOTEgLTN6Ii8%2BPHBhdGggZD0iTTM5MzEgNDQ3NSBjLTYgLTggLTcgLTEwNCAtMiAtMTUzIDEgLTEwIDIyIC0xMiA4NCAtMTAgbDgyIDMgMCA3OCBjMSA0MyAtMyA4MSAtOCA4NCAtMTcgMTAgLTE0OCA4IC0xNTYgLTJ6Ii8%2BPHBhdGggZD0iTTQ2OTQgNDA1OCBsMSAtODMgLTg2IC0zIC04NiAtMyAwIC04NCAwIC04NCA4NiAtMyA4NiAtMyAtMSAtODMgLTEgLTgzIDg2IDMgODYgMyAwIDgwIDAgODAgLTg2IDMgLTg2IDMgMSA4NCAwIDg0IDg1IDMgODUgMyAxIDgzIDEgODIgLTg3IDAgLTg3IDAgMiAtODJ6Ii8%2BPHBhdGggZD0iTTQ4NjQgMzg4OCBsMSAtODMgODUgMCA4NSAwIDAgODAgLTEgODAgLTg1IDMgLTg1IDMgMCAtODN6Ii8%2BPC9nPjwvc3ZnPg%3D%3D)](https://coindrop.to/smoochy) [![Tip me uptime](https://img.shields.io/badge/Tip%20me%20uptime%20%F0%9F%96%A5%EF%B8%8F-smoochy84-E9C46A?logo=buymeacoffee&logoColor=000000)](https://www.buymeacoffee.com/smoochy84) [![Ko-fi](https://img.shields.io/badge/Ko--fi-smoochy-7CC6FE?logo=ko-fi&logoColor=000000)](https://ko-fi.com/smoochy)

> Custom [Caddy](https://github.com/caddyserver/caddy) images with curated module sets, published to GHCR with a public Docker Hub mirror and rebuilt automatically when upstream components change.

This repository provides ready-to-use Caddy container images for self-hosted
and infrastructure setups that need more than the official default image. Each
image variant is built with a defined module set, tracked against upstream
changes, and published with reproducible tags plus traceable metadata.

Registries:

- GHCR (canonical): `ghcr.io/smoochy/caddy-cloudflare-modules`
- Docker Hub (public mirror): `smoochy84/caddy-cloudflare-modules`

If this project saves you time or helps your setup, you can support ongoing
maintenance via Coindrop, Ko-fi, or Buy Me a Coffee.

## Table of Contents

- [Background](#background)
- [Available Images](#available-images)
- [Cloudflare Image](#cloudflare-image)
- [When Builds Run](#when-builds-run)
- [Dockerfile](#dockerfile)
- [GitHub Actions Workflow](#github-actions-workflow)
- [Workflow Verification](#workflow-verification)
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
    - `ghcr.io/smoochy/caddy-cloudflare-modules:caddy-v<x.y.z>`
  - Docker Hub (public mirror):
    - `smoochy84/caddy-cloudflare-modules:latest`
    - `smoochy84/caddy-cloudflare-modules:caddy-<x.y.z>`
    - `smoochy84/caddy-cloudflare-modules:caddy-v<x.y.z>`
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

## Workflow Verification

This section states what is checked when a change to `build_cloudflare-modules.yaml` runs, and what is not checked before merge.

A pull request build runs the workflow with `push: false`, so the image is built but never published to GHCR or Docker Hub. This exercises the Dockerfile and the addon install steps without any write to a registry.

On a pull request, when `do_build` is `true`, the `Assert produced tag set` step reads the tag list `docker/metadata-action` produced and fails the job unless that set matches exactly what is expected for the `caddy_tag` value the `decide` step emitted. The comparison is by set membership plus a count, so it does not depend on order or formatting. This step never runs on `push`, `schedule`, or `workflow_dispatch`, so it can never fail a publishing build. When `do_build` is `false`, both the `meta` step and this assertion step are skipped.

A change to this workflow is expected to pass these static checks locally before review:

- Parse the whole workflow file with a YAML parser and confirm it succeeds.
- Extract the body of every `run:` block you changed to a temporary file and run `shellcheck` on it.
- Copy the shell logic you changed to a temporary file, replace the workflow expressions with shell variables, and execute it against every value the producing step can emit. For the tag logic that means both shapes of `caddy_tag`: a three-part semver such as `2.11.4`, and the literal string `latest`.

These local checks are a fast pre-review pass, not the only gate: the `Lint workflows` GitHub Actions workflow runs `actionlint` in CI on any pull request that touches `.github/workflows/**`, and again on every push to `main`. `actionlint` covers unknown context properties, invalid expressions, and undefined step outputs across the whole workflow file, and it also runs `shellcheck` over every `run:` block, so the shellcheck pass a contributor ran locally is checked again automatically before merge.

The workflow also runs on the schedule described in "When Builds Run", and on a push to `main` that touches `Dockerfile-cloudflare`, `.dockerignore`, or the workflow file itself. Neither run publishes on its own. The `decide` step still has to judge a build necessary: a push forces one only when it changed `Dockerfile-cloudflare` or `.dockerignore`, a scheduled run never forces one, and either kind of run still builds and publishes when it detects an upstream Caddy or addon change. Only once `decide` outputs `do_build: true` on a `push`, `schedule`, or `workflow_dispatch` event does the real multi-arch push to GHCR happen, and the `crane copy` mirror to Docker Hub runs on top of that only when Docker Hub secrets are configured.

What is not verified before a merge: a real multi-arch push, a real `crane copy` to Docker Hub, and a real write to either registry. A pull request never publishes, so those three paths can only happen after the merge, and only on a later push or scheduled run where `decide` judges a build necessary.

`act` was considered for local verification of this workflow and rejected. The `decide` step calls `crane` against `caddy:latest` and the GitHub API for release metadata, so a local `act` run is not hermetic. `act` also cannot reproduce the multi-arch `docker/build-push-action` push or the `crane copy` mirror, which is exactly the part no static check covers, so it would not close that gap. What a local `act` run would actually prove is that the YAML parses and the shell branches execute, and `shellcheck` plus the pull request tag assertion already prove that more cheaply and in the place that gates a merge. Against that, `act` costs a Docker-in-Docker setup in a repository that has no other local toolchain.

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
- `caddy-<x.y.z>`: matches the Caddy base version used at build time and is useful for reproducible deployments pinned to a specific Caddy release
- `caddy-v<x.y.z>`: the same version, matching the upstream Caddy release tag name (for example `v2.11.4`), so pinning by it is synchronous with upstream

## Install

Pull the published image from GHCR (canonical) or Docker Hub (public mirror):

```bash
docker pull ghcr.io/smoochy/caddy-cloudflare-modules:latest
```

```bash
docker pull smoochy84/caddy-cloudflare-modules:latest
```

For reproducible deployments, pin both a version tag and digest, using either version tag shape:

```text
ghcr.io/smoochy/caddy-cloudflare-modules:caddy-<x.y.z>@sha256:<digest>
ghcr.io/smoochy/caddy-cloudflare-modules:caddy-v<x.y.z>@sha256:<digest>
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
