---
type: Reference
title: caddy-modules Wiki Quickstart
description: "Entry point for the caddy-modules OpenWiki documentation. Covers custom Caddy Docker images with Cloudflare modules, automated build pipeline, and publishing workflow."
tags: [reference, quickstart, caddy, docker, cloudflare, github-actions]
openwiki:
  roles: [repository, architecture]
  change_kinds: [lifecycle, public-api, configuration, integration, operations]
  source_paths: [README.md, Dockerfile-cloudflare, .github/workflows/build_cloudflare-modules.yaml]
  symbols: [xcaddy, caddy:builder, caddy:latest, crane, buildx]
  test_paths: []
  invariants: [Two-stage Dockerfile produces minimal final image, Build workflow dynamically discovers addons from Dockerfile, OCI labels enable cross-run change detection, Bi-weekly OpenWiki updates via scheduled workflow]
  validation_commands: [docker build -f Dockerfile-cloudflare ., gh workflow run build_cloudflare-modules.yaml --ref main -f force=true]
verified:
  - by: openwiki/0.6.1
    at: 2026-10-01T11:27:53.038Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-42ef8192a0f869c58499e372
    resource: repo://Dockerfile-cloudflare
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.6.1", at: "2026-10-01T11:27:53.038Z" }
---

# caddy-modules Wiki Quickstart

## Overview

This repository provides **custom Caddy Docker images** with curated modules, published to GHCR (canonical) and Docker Hub (mirror), with **automated rebuilds** when upstream components change.

**Primary image**: `ghcr.io/smoochy/caddy-cloudflare-modules` (mirror: `smoochy84/caddy-cloudflare-modules`)

**Modules included**:
- Cloudflare DNS (DNS-01 ACME challenges)
- Cloudflare IP (real client IP behind proxy)
- Combine IP Ranges (trusted proxy aggregation)
- Docker Proxy (auto-config via Docker labels)

## Documentation Map

| Area | Page | Purpose |
|------|------|---------|
<!-- openwiki: broken internal link [/openwiki/architecture/overview.md] link "/openwiki/architecture/overview.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Architecture** | [Architecture Overview](/openwiki/architecture/overview.md) | System context, Dockerfile, workflow, data flow, extension points |
<!-- openwiki: broken internal link [/openwiki/workflows/build-pipeline.md] link "/openwiki/workflows/build-pipeline.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Build Pipeline** | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md) | GitHub Actions workflow detail: triggers, change detection, publishing |
<!-- openwiki: broken internal link [/openwiki/operations/image-tags.md] link "/openwiki/operations/image-tags.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Image Tags** | [Image Tags & Publishing](/openwiki/operations/image-tags.md) | Tagging strategy, OCI labels, reproducible deployments, Docker Hub mirror |
<!-- openwiki: broken internal link [/openwiki/operations/dockerfile.md] link "/openwiki/operations/dockerfile.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Dockerfile** | [Dockerfile Structure](/openwiki/operations/dockerfile.md) | Multi-stage build, xcaddy, addon declaration, module management |
<!-- openwiki: broken internal link [/openwiki/integrations/cloudflare-modules.md] link "/openwiki/integrations/cloudflare-modules.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Modules** | [Cloudflare Modules Integration](/openwiki/integrations/cloudflare-modules.md) | Four modules: config, usage, troubleshooting |
<!-- openwiki: broken internal link [/openwiki/operations/dockerhub-sync.md] link "/openwiki/operations/dockerhub-sync.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **Docker Hub Sync** | [Docker Hub Description Sync](/openwiki/operations/dockerhub-sync.md) | README.dockerhub.md → Docker Hub description workflow |
<!-- openwiki: broken internal link [/openwiki/operations/openwiki-update.md] link "/openwiki/operations/openwiki-update.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| **OpenWiki Update** | [OpenWiki Update Workflow](/openwiki/operations/openwiki-update.md) | Bi-weekly documentation regeneration workflow |

## Quick Tasks

### Trigger a Build
```bash
# Force rebuild (manual)
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true

# Check latest run
gh run list --workflow=build_cloudflare-modules.yaml --limit=1
```

### Verify Published Image
```bash
# Check labels (traceability)
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels'

# Verify multi-arch
crane manifest ghcr.io/smoochy/caddy-cloudflare-modules:latest

# List compiled modules
docker run --rm ghcr.io/smoochy/caddy-cloudflare-modules:latest list-modules
```

### Add a New Module
1. Edit `Dockerfile-cloudflare`, add `--with github.com/owner/repo` line
2. Push to main → workflow auto-discovers, fetches version, adds labels, tracks changes
3. No workflow changes needed

### Add a New Image Variant
1. Copy `Dockerfile-cloudflare` → `Dockerfile-<name>`, adjust `--with` lines
2. Copy `.github/workflows/build_cloudflare-modules.yaml` → `.github/workflows/build_<name>.yaml`
3. Update image name and `file:` reference in new workflow
4. Add to README.md Available Images table

### Update Documentation
```bash
# Manual OpenWiki update
gh workflow run openwiki-update.yaml --ref main
```

## Key Invariants

| Invariant | Where Enforced |
|-----------|----------------|
| Minimal final image (only custom binary) | Dockerfile two-stage build |
| Dynamic addon discovery | Workflow `grep -- '--with' Dockerfile` |
| Change detection via OCI labels | Workflow `decide` step compares base digest + addon versions |
| Reproducible version tags | `caddy-<x.y.z>` matches Caddy base version |
| Bi-weekly doc updates | OpenWiki workflow parity gate (even ISO weeks) |

## Change Navigation

| Change Intent | Start Here | Key Files |
|---------------|------------|-----------|
<!-- openwiki: broken internal link [/openwiki/operations/dockerfile.md#adding-a-new-module] link "/openwiki/operations/dockerfile.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Add Caddy module | [Dockerfile Structure](/openwiki/operations/dockerfile.md#adding-a-new-module) | `Dockerfile-cloudflare` |
<!-- openwiki: broken internal link [/openwiki/architecture/overview.md#adding-a-new-image-variant] link "/openwiki/architecture/overview.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Add image variant | [Architecture Overview](/openwiki/architecture/overview.md#adding-a-new-image-variant) | `Dockerfile-<name>`, `.github/workflows/build_<name>.yaml` |
<!-- openwiki: broken internal link [/openwiki/workflows/build-pipeline.md#trigger-conditions] link "/openwiki/workflows/build-pipeline.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Modify build triggers | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md#trigger-conditions) | `.github/workflows/build_cloudflare-modules.yaml` |
<!-- openwiki: broken internal link [/openwiki/operations/image-tags.md#tagging-strategy] link "/openwiki/operations/image-tags.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Change tagging scheme | [Image Tags & Publishing](/openwiki/operations/image-tags.md#tagging-strategy) | Workflow `meta` step |
<!-- openwiki: broken internal link [/openwiki/workflows/build-pipeline.md#validation-commands] link "/openwiki/workflows/build-pipeline.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Debug failed build | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md#validation-commands) | Job summary, `gh run view --log-failed` |
<!-- openwiki: broken internal link [/openwiki/operations/dockerhub-sync.md] link "/openwiki/operations/dockerhub-sync.md" is root-absolute, which no real consumer resolves against the repository root (not a coding agent reading the page, not GitHub's Markdown renderer, not a local viewer); use a path relative to this file instead. Fix the href or restore the target, then delete this comment. -->
| Update Docker Hub desc | [Docker Hub Description Sync](/openwiki/operations/dockerhub-sync.md) | `README.dockerhub.md`, sync workflow |

## Backlog

*No backlog items - initial documentation complete*
