---
type: "Reference"
title: "Verification Strategy"
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-3c8e8b4445cd30ad2a898a7e
    resource: repo://.github/workflows/lint_workflows.yaml
  - id: openwiki-source-c9b161b431b44a83e627c9b7
    resource: repo://docs/assembly-line/2026-08-24-workflow-verification/plan.md
  - id: openwiki-source-23775c3de52f3ab95a13cb8b
    resource: repo://README.md
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---


# Verification Strategy

This page documents the verification strategy for the `build_cloudflare-modules.yaml` workflow. It states what is checked when a change to the workflow runs, and what is not checked before merge.

## Verification Layers

The workflow is verified through four layers, each covering a different scope:

| Layer | When It Runs | What It Covers |
|-------|--------------|----------------|
| **Static checks (local)** | Contributor runs before opening PR | YAML parse, `shellcheck` on modified `run:` blocks, manual branch trace against all `caddy_tag` values |
| **Static checks (CI)** | Every PR touching `.github/workflows/**` | `actionlint` over the entire workflow file: unknown contexts, invalid expressions, undefined step outputs, and `shellcheck` on every `run:` block |
| **Pull request tag assertion** | Every PR where `do_build == true` | Asserts the exact tag set produced by `docker/metadata-action` matches the expected set for the `caddy_tag` value |
| **Scheduled / push run** | Daily 03:00 UTC schedule, push to `main` on filtered paths | Real multi-arch push to GHCR, `crane copy` mirror to Docker Hub, end-to-end publish path |

## Pull Request Tag Assertion

The workflow includes a step named **Assert produced tag set** that runs only on pull requests when the `decide` step outputs `do_build == 'true'`.

```yaml
- name: Assert produced tag set
  if: github.event_name == 'pull_request' && steps.decide.outputs.do_build == 'true'
  shell: bash
  env:
    META_TAGS: ${{ steps.meta.outputs.tags }}
    CADDY_TAG: ${{ steps.decide.outputs.caddy_tag }}
  run: |
    # ... set membership + count comparison ...
```

### Behavior

- Reads `steps.meta.outputs.tags` (newline-separated fully qualified image references from `docker/metadata-action`) and `steps.decide.outputs.caddy_tag`.
- Computes the expected tag set:
  - When `caddy_tag` is a three-part semver (e.g., `2.11.4`): `latest`, `caddy-<tag>`, `caddy-v<tag>`
  - When `caddy_tag` is the literal string `latest`: `latest`, `caddy-latest`
- Compares by **set membership plus count**, independent of line order or formatting.
- Fails the job with a diagnostic message naming both the expected set and the produced set if they do not match.
- Never runs on `push`, `schedule`, or `workflow_dispatch` events, so it cannot fail a publishing build.
- When `do_build == 'false'`, both the `meta` step and this assertion are skipped (required behavior, not an oversight).

### What This Catches

- Incorrect tag templates in the `meta` step (`docker/metadata-action` `tags:` input)
- Logic errors in the `decide` step's `caddy_tag` computation that would produce wrong tags
- Changes to the tag schema that would diverge from what the README documents

## Static Checks

A contributor changing the workflow is expected to run these checks locally before opening a pull request:

1. **YAML parse**: Validate the entire workflow file with a YAML parser (e.g., `yq`, `python -c "import yaml; yaml.safe_load(...)"`).
2. **Shellcheck**: Extract the body of every modified `run:` block to a temporary file and run `shellcheck` on it.
3. **Branch trace**: For each changed shell block, copy the logic to a temporary file, replace workflow expressions (`${{ ... }}`) with shell variables, and execute it against every value the producing step can emit. For the tag logic, this means both shapes of `caddy_tag`:
   - A three-part semver such as `2.11.4`
   - The literal string `latest`

In CI, the **Lint workflows** workflow (`.github/workflows/lint_workflows.yaml`) runs `actionlint` on every pull request touching `.github/workflows/**`. It covers:
- Unknown context properties
- Invalid expressions
- Undefined step outputs
- `shellcheck` on every `run:` block (with `SC2016`, `SC2129`, `SC2155` excluded as documented in the workflow)

These local checks are a fast pre-review pass, not the only gate. The CI `actionlint` run re-validates the same shell blocks automatically before merge.

## What Is Not Verified Before Merge

The following paths are **not** exercised or verified by any pull request run:

| Gap | Description |
|-----|-------------|
| **Real multi-arch push** | Pull requests run `docker/build-push-action` with `push: false` and a single platform (`linux/amd64`). The multi-arch manifest push (`linux/amd64,linux/arm64,linux/arm/v7`) to GHCR only occurs on `push`, `schedule`, or `workflow_dispatch` events after merge. |
| **Real `crane copy` to Docker Hub** | The mirror step runs only on non-PR events when Docker Hub secrets are configured and `do_build: true`. A pull request never executes `crane copy`. |
| **Real registry write** | Neither GHCR nor Docker Hub receives any write from a pull request run. The `Login to GHCR` step is skipped for external PR forks. |

These three paths can only happen after the merge, and only on a later push or scheduled run where `decide` judges a build necessary.

## `act` Evaluation and Rejection Ruling

`act` was considered for local verification of this workflow and **rejected**. The ruling is recorded here so the option is closed rather than left open.

### Reasons (drawn from this repository)

1. **The `decide` step is not hermetic**: It calls `crane` against `caddy:latest` and the GitHub API for release metadata. A local `act` run would require network access to both, so it would not be a self-contained local dry run.
2. **`act` cannot reproduce the multi-arch push**: The `docker/build-push-action` multi-arch push (`linux/amd64,linux/arm64,linux/arm/v7`) is the part no static check covers. `act` cannot reproduce this, so it would not close the verification gap.
3. **What a local `act` run would actually prove**: Only that the YAML parses and the shell branches execute. The static checks (`shellcheck` + branch trace) plus the pull request tag assertion already prove that more cheaply and in the place that gates a merge.
4. **Cost**: `act` requires a Docker-in-Docker setup in a repository that has no other local toolchain.

### What Remains the Standard

The verification standard for this workflow is:
- Static checks (local + CI `actionlint`)
- Pull request tag assertion (the executable gate on every relevant PR)
- Scheduled / push runs for the real publish path

`act` is not adopted and no Docker-in-Docker setup is introduced.

## Scheduled and Push Runs

The workflow also runs on:
- **Schedule**: Daily at 03:00 UTC. Checks for upstream Caddy base digest changes and addon version changes. Builds and publishes only when `decide` detects a change.
- **Push to `main`**: When `Dockerfile-cloudflare`, `.dockerignore`, or the workflow file itself changes. Forces a build for local input changes; otherwise still depends on `decide` detecting upstream changes.

On these events, when `do_build == 'true'`:
- Multi-arch image is built and pushed to GHCR
- `crane copy` mirrors tags to Docker Hub (if secrets configured)
- Real registry writes occur

These runs are the only verification of the real publish path.

## Reference

<!-- openwiki: broken internal link [../README.md#workflow-verification] file "../README.md" does not exist. Fix the href or restore the target, then delete this comment. -->
This strategy is also summarized in the **Workflow Verification** section of the [README.md](../README.md#workflow-verification), which serves as the contributor-facing reference. The [Security Considerations](../operations/security.md#verification-gaps) page documents the same gaps from a supply-chain perspective.
