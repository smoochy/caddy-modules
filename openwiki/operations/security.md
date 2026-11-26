---
type: operations
title: Security Considerations
description: Documents supply chain trust, secrets management, registry permissions, and verification gaps for the caddy-modules build pipeline.
tags: [security, supply-chain, secrets, registry, verification]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-e082dc1d961398caa2e27f47
    resource: repo://.github/workflows/openwiki-update.yaml
  - id: openwiki-source-42ef8192a0f869c58499e372
    resource: repo://Dockerfile-cloudflare
  - id: openwiki-source-c9b161b431b44a83e627c9b7
    resource: repo://docs/assembly-line/2026-08-24-workflow-verification/plan.md
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---

# Security Considerations

This page documents the security posture of the `caddy-modules` build pipeline: which upstream components are trusted, how secrets are handled, what registry permissions are required, and what is **not** verified before a merge.

## Supply Chain Trust

### Base Images

The Dockerfile uses two official Caddy base images from Docker Hub:

| Stage | Image | Purpose |
|-------|-------|---------|
| Builder | `caddy:builder` | Provides `xcaddy` and the Go toolchain to compile Caddy with additional modules |
| Runtime | `caddy:latest` | Minimal runtime base; the custom-built `caddy` binary is copied into it |

Both images are published by the Caddy project (`caddyserver/caddy` on Docker Hub). The workflow resolves `caddy:latest` to a concrete digest at build time and records that digest in the output image labels (`org.opencontainers.image.base.digest`). A rebuild is triggered automatically when the digest changes.

**Trust assumption**: The official Caddy images are assumed to be unmodified upstream artifacts. No additional verification (cosign, SBOM, reproducibility build) is performed in this pipeline.

### Addon Sources

All Caddy modules are fetched from public GitHub repositories at build time via `xcaddy`. The workflow parses `--with` lines in the Dockerfile and queries the GitHub API for each addon's latest stable release (or tag, or commit SHA as a fallback).

| Addon | Repository | Purpose |
|-------|------------|---------|
| Cloudflare DNS | `github.com/caddy-dns/cloudflare` | DNS-01 ACME challenge provider for Cloudflare |
| Cloudflare IP | `github.com/WeidiDeng/caddy-cloudflare-ip` | Extracts real client IP when behind Cloudflare proxy |
| Combine IP Ranges | `github.com/fvbommel/caddy-combine-ip-ranges` | Merges multiple IP range sources for trusted proxies |
| Docker Proxy | `github.com/lucaslorentz/caddy-docker-proxy/v2` | Automatic Caddy configuration via Docker labels |

**Version pinning behavior**: The workflow does **not** pin addons to a fixed version in the Dockerfile. Instead, it resolves the latest stable release at build time and records the resolved version in the image labels (`org.opencontainers.image.addon.N.version`). A rebuild occurs automatically when the workflow detects a version change via the scheduled run or a manual dispatch.

**Trust assumption**: Each addon repository is trusted to publish legitimate releases. The workflow authenticates to the GitHub API using `GITHUB_TOKEN` (see [Secrets](#secrets-management)), which provides higher rate limits but does not verify signature or provenance of the addon source code.

## Secrets Management

### `GITHUB_TOKEN` (Built-in)

- **Scope**: Automatically provided by GitHub Actions for each workflow run.
- **Used for**:
  - Authenticating `docker/login-action` to push images to **GHCR** (`ghcr.io/smoochy/caddy-cloudflare-modules`).
  - Authenticating GitHub API calls in the `decide` step to fetch release metadata for Caddy and all addons.
- **Permissions**: The workflow declares `permissions: { contents: read, packages: write }`, granting read access to the repository and write access to the package registry.

### `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` (Optional Repository Secrets)

- **Scope**: Configured as repository secrets in the GitHub UI. Both must be present for Docker Hub publishing to be enabled.
- **Used for**: Authenticating `docker/login-action` to mirror the built image to **Docker Hub** (`smoochy84/caddy-cloudflare-modules`).
- **Behavior**: If either secret is missing, the mirror step is skipped silently and the job summary records "Skipped: set `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` repository secrets to enable Docker Hub publishing."
- **Risk**: These secrets grant push access to the Docker Hub repository. Rotate them if the maintainer team changes.

### `OPENROUTER_API_KEY` (OpenWiki Workflow Only)

- **Scope**: Repository secret used **only** by the separate `.github/workflows/openwiki-update.yaml` workflow.
- **Used for**: Authenticating to OpenRouter to run the OpenWiki documentation generator.
- **Isolation**: This secret is **not** available to the build workflow (`build_cloudflare-modules.yaml`) and does not interact with the container image build or publish steps.

## Registry Permissions

| Registry | Image | Permission | Mechanism |
|----------|-------|------------|-----------|
| GHCR | `ghcr.io/smoochy/caddy-cloudflare-modules` | Write (push) | `GITHUB_TOKEN` with `packages: write` |
| Docker Hub | `smoochy84/caddy-cloudflare-modules` | Write (push) | `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets (optional) |

The workflow pushes multi-arch manifests (`linux/amd64`, `linux/arm64`, `linux/arm/v7`) to GHCR on every successful build triggered by `push`, `schedule`, or `workflow_dispatch` (when `do_build: true`). The Docker Hub mirror uses `crane copy` to replicate tags without re-building.

## Verification Gaps

The following are **not** exercised or verified before a pull request is merged:

| Gap | Description |
|-----|-------------|
| Real multi-arch push | Pull requests run `docker/build-push-action` with `push: false` and a single platform (`linux/amd64`). The multi-arch manifest push to GHCR only occurs on `push`, `schedule`, or `workflow_dispatch` events after merge. |
| Real `crane copy` to Docker Hub | The mirror step runs only on non-PR events when Docker Hub secrets are configured and `do_build: true`. A pull request never executes `crane copy`. |
| Real registry write | Neither GHCR nor Docker Hub receives any write from a pull request run. The `Login to GHCR` step is skipped for external PR forks (`if: github.event_name != 'pull_request' || github.event.pull_request.head.repo.full_name == github.repository`). |
| Addon provenance | The workflow fetches version metadata from the GitHub API but does not verify signatures, SBOMs, or reproducible builds for any addon. |
| Base image provenance | The `caddy:builder` and `caddy:latest` digests are recorded but not verified against an independent source (e.g., cosign keyless verification). |

### Local Verification Standard

Contributors changing the build workflow are expected to run these static checks locally before opening a pull request:

1. **YAML parse**: Validate the entire workflow file with a YAML parser.
2. **Shellcheck**: Extract every modified `run:` block to a temporary file and run `shellcheck` on it.
3. **Branch trace**: For each changed shell block, replace workflow expressions (`${{ ... }}`) with shell variables and execute the script against every value the producing step can emit (for tag logic: `caddy_tag` as a three-part semver like `2.11.4`, and the literal string `latest`).

In CI, the `Lint workflows` workflow runs `actionlint` on every pull request touching `.github/workflows/**`, which covers unknown context properties, invalid expressions, undefined step outputs, and runs `shellcheck` on every `run:` block.

The pull request build also includes a **tag assertion step** (guarded by `if: github.event_name == 'pull_request' && steps.decide.outputs.do_build == 'true'`) that reads the tag list produced by `docker/metadata-action` and fails the job unless the set matches exactly what is expected for the `caddy_tag` value emitted by `decide`. This assertion runs only on pull requests and never on publishing events.

### `act` Evaluation

The repository evaluated `act` for local Actions execution and **rejected** it because:

- The `decide` step requires network access to `crane` (to resolve `caddy:latest` digest) and the GitHub API (for release metadata), so a local `act` run is not hermetic.
- `act` cannot reproduce the multi-arch `docker/build-push-action` push or the `crane copy` mirror, which are exactly the paths no static check covers.
- A local `act` run would only prove that the YAML parses and shell branches execute—already covered more cheaply by `shellcheck` plus the pull request tag assertion.

## Related Pages

- [Build Pipeline](../workflows/build-pipeline.md) — Workflow structure, triggers, and publish logic.
- [Image Tags](./image-tags.md) — Tag schema, digest pinning, and reproducibility.
- [Architecture Overview](../architecture/overview.md) — High-level component diagram and data flow.
