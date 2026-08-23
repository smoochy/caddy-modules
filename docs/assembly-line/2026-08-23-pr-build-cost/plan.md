# Plan: cut the duplicated pull-request build cost

## Problem

`.github/workflows/build_cloudflare-modules.yaml` builds the image for `linux/amd64,linux/arm64,linux/arm/v7`. The two non-amd64 platforms run under QEMU emulation, which is what makes a run take about 30 minutes.

The workflow triggers on both `pull_request` and `push` to `main`, with the same `paths` filter. A pull request that touches `Dockerfile-cloudflare`, `.dockerignore` or the workflow file therefore runs a full 30-minute three-platform build with `push: false`, and the merge commit then runs the identical 30-minute build again on `main`. GitHub Actions cache does not bridge the two runs, because a cache entry written on a pull-request ref is not readable from `main`.

## Outcome

A pull request validates the image build in a few minutes instead of about 30. The published multi-platform image is unchanged: `push`, `schedule` and `workflow_dispatch` runs still build all three platforms.

## Non-goals

- Removing the `pull_request` trigger. Validation on a pull request stays.
- Changing the `do_build` decision logic. A merged Dockerfile change is meant to force a full build on `main`.
- Gating `docker/setup-qemu-action`. It is the binfmt setup only, seconds not minutes.
- Changing the concurrency group. It separates pull-request and `main` runs correctly already.

## Work packages

### Task 1 - Package 1 - pull-request builds only `linux/amd64`

**Outcome:** the `Build and push` step builds a single platform on `pull_request` and the full three-platform set on every other event.

**Point from the brief, verbatim:** "PR-Build auf `linux/amd64` beschränken (Dockerfile-Syntax + Build-Fähigkeit verifiziert, ~2-3m statt 30m), volle Matrix nur auf `push`."

**Declared file scope:** `.github/workflows/build_cloudflare-modules.yaml` (line 584).

**Change:**

```yaml
platforms: ${{ github.event_name == 'pull_request' && 'linux/amd64' || 'linux/amd64,linux/arm64,linux/arm/v7' }}
```

`docker/build-push-action` accepts a comma-separated string, so no newline list is needed. `docker/metadata-action` tags are plain string templates over `caddy_tag` and do not depend on the platform set, so they need no change.

**Acceptance claim:** on a `pull_request` event the `Build and push` step receives `platforms: linux/amd64`; on `push`, `schedule` and `workflow_dispatch` it receives `platforms: linux/amd64,linux/arm64,linux/arm/v7`. The workflow file still parses as valid YAML and `actionlint` reports no new error.

### Task 2 - Package 2 - pull-request runs read the build cache but never write it

**Outcome:** a pull-request run restores from the GitHub Actions cache as before, and stops exporting a cache entry that no later run can use.

**Point from the brief, verbatim:** "Cache-Sharing erzwingen: `cache-from: type=gha,scope=main` zusätzlich, damit der PR-Lauf vom main-Cache liest - hilft aber nur PR-Richtung, nicht umgekehrt."

**Correction folded in before the cut:** adding `scope=main` to `cache-from` is a no-op. `scope` in the buildx gha backend is an internal namespace for several cache lines in one repository, not a branch selector. Plain `type=gha` already walks the GitHub Actions cache restore chain, current ref then pull-request base ref then default branch, and both events currently use the same default scope `buildkit`. So a pull-request run already reads the `main` cache.

The real waste in this area is the opposite direction: `cache-to: type=gha,mode=max` on a pull-request run spends time exporting and uploading layers into a cache entry keyed to the pull-request ref, which no `main` run can ever read, and that entry competes for the repository's 10 GB cache quota and can evict older `main` entries.

**Declared file scope:** `.github/workflows/build_cloudflare-modules.yaml` (lines 596-597).

**Change:**

```yaml
          cache-from: type=gha
          cache-to: ${{ github.event_name != 'pull_request' && 'type=gha,mode=max' || '' }}
```

An empty `cache-to` is read by `build-push-action` as no cache export flags at all.

**Acceptance claim:** `cache-from` is unchanged at `type=gha`; on a `pull_request` event `cache-to` resolves to the empty string, and on every other event it resolves to `type=gha,mode=max`. The workflow file still parses as valid YAML and `actionlint` reports no new error.

## Waves

Both packages edit the same file, so their declared file scopes are not disjoint and they cannot share a wave.

- **Wave 1:** Package 1. Sequential.
- **Wave 2:** Package 2. Sequential.

## Triggers

- **Shared-artifact trigger:** no hit. The repository has no lockfile and no package manifest; `.gitignore` holds only `.agents/`; the only shared file in play is the workflow itself, which the wave cut already serialises.
- **Security trigger:** no hit. The declared file scope of both packages matches none of the security vocabulary, so the pre-approval `security-reviewer` pass is skipped and both packages take the generic implementer and the correctness second inspector.

## Verification

`actionlint` on the changed workflow, plus a read of the rendered expressions. A real end-to-end proof needs a pull request against `main` that touches one of the filtered paths; this plan's own pull request is exactly that, so the pull-request run itself is the acceptance evidence.
