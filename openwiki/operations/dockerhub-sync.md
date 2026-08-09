---
type: Operations
title: Docker Hub Description Sync
description: Workflow that syncs README.dockerhub.md to Docker Hub repository description when README or dockerhub workflow changes.
tags: [operations, docker-hub, sync, description, github-actions]
openwiki:
  roles: [operations, delivery]
  change_kinds: [operations, configuration]
  source_paths: [.github/workflows/sync_dockerhub_description.yaml, README.dockerhub.md]
  symbols: [peter-evans/dockerhub-description]
  test_paths: []
  invariants: [Runs on push to main when README.md or README.dockerhub.md changes, Conditional on DOCKERHUB_USERNAME and DOCKERHUB_TOKEN secrets]
  validation_commands: [gh workflow run sync_dockerhub_description.yaml --ref main]
---

# Docker Hub Description Sync

## Workflow File
`.github/workflows/sync_dockerhub_description.yaml`

## Purpose

Automatically updates the **Docker Hub repository description** (the long description shown on the Docker Hub page) from `README.dockerhub.md` when relevant files change.

## Trigger Conditions

| Trigger | Condition |
|---------|-----------|
| **Push to main** | Files changed: `README.md`, `README.dockerhub.md`, workflow file |
| **Manual** | `workflow_dispatch` |

## Workflow Steps

### 1. Check Docker Hub Secrets (`dockerhub` step)
```bash
if [ -n "${DOCKERHUB_USERNAME:-}" ] && [ -n "${DOCKERHUB_TOKEN:-}" ]; then
  echo "enabled=true"
else
  echo "enabled=false"
fi
```

### 2. Update Docker Hub Description (Conditional)
Uses `peter-evans/dockerhub-description@v5`:
```yaml
uses: peter-evans/dockerhub-description@v5
with:
  username: ${{ secrets.DOCKERHUB_USERNAME }}
  password: ${{ secrets.DOCKERHUB_TOKEN }}
  repository: smoochy84/caddy-cloudflare-modules
  short-description: "Custom Caddy images with curated modules and automatic upstream rebuilds"
  readme-filepath: ./README.dockerhub.md
  enable-url-completion: true
```

### 3. Write Summary
Outputs success/skip status to job summary.

## Required Secrets

| Secret | Purpose |
|--------|---------|
| `DOCKERHUB_USERNAME` | Docker Hub username (e.g., `smoochy84`) |
| `DOCKERHUB_TOKEN` | Docker Hub access token (not password) |

**Both must be configured** for the sync to run. If missing, workflow completes successfully but skips the update.

## Source File: `README.dockerhub.md`

Separate from main `README.md` - optimized for Docker Hub display:
- Shorter, focused on image usage
- No GitHub-specific badges/links
- Compatible with Docker Hub markdown rendering

## Relationship to Build Workflow

| Aspect | Build Workflow | Sync Workflow |
|--------|----------------|---------------|
| **Triggers** | Dockerfile, .dockerignore, workflow | README.md, README.dockerhub.md, workflow |
| **Publishes** | Image layers + tags | Description metadata only |
| **Secrets** | GHCR (GITHUB_TOKEN), Docker Hub (optional) | Docker Hub only |
| **Frequency** | Daily schedule + changes | On README changes only |

## Manual Sync

```bash
# Trigger manually
gh workflow run sync_dockerhub_description.yaml --ref main
```

## Verification

After sync, check Docker Hub repository page:
- https://hub.docker.com/r/smoochy84/caddy-cloudflare-modules
- Description should match `README.dockerhub.md` content