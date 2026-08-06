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
| **Architecture** | [Architecture Overview](/openwiki/architecture/overview.md) | System context, Dockerfile, workflow, data flow, extension points |
| **Build Pipeline** | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md) | GitHub Actions workflow detail: triggers, change detection, publishing |
| **Image Tags** | [Image Tags & Publishing](/openwiki/operations/image-tags.md) | Tagging strategy, OCI labels, reproducible deployments, Docker Hub mirror |
| **Dockerfile** | [Dockerfile Structure](/openwiki/operations/dockerfile.md) | Multi-stage build, xcaddy, addon declaration, module management |
| **Modules** | [Cloudflare Modules Integration](/openwiki/integrations/cloudflare-modules.md) | Four modules: config, usage, troubleshooting |
| **Docker Hub Sync** | [Docker Hub Description Sync](/openwiki/operations/dockerhub-sync.md) | README.dockerhub.md → Docker Hub description workflow |
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
| Add Caddy module | [Dockerfile Structure](/openwiki/operations/dockerfile.md#adding-a-new-module) | `Dockerfile-cloudflare` |
| Add image variant | [Architecture Overview](/openwiki/architecture/overview.md#adding-a-new-image-variant) | `Dockerfile-<name>`, `.github/workflows/build_<name>.yaml` |
| Modify build triggers | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md#trigger-conditions) | `.github/workflows/build_cloudflare-modules.yaml` |
| Change tagging scheme | [Image Tags & Publishing](/openwiki/operations/image-tags.md#tagging-strategy) | Workflow `meta` step |
| Debug failed build | [Build Pipeline Workflow](/openwiki/workflows/build-pipeline.md#validation-commands) | Job summary, `gh run view --log-failed` |
| Update Docker Hub desc | [Docker Hub Description Sync](/openwiki/operations/dockerhub-sync.md) | `README.dockerhub.md`, sync workflow |

## Backlog

*No backlog items - initial documentation complete*