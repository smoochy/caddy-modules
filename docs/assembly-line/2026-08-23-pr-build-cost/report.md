# Report: 2026-08-23-pr-build-cost

## What the run changed

The workflow `.github/workflows/build_cloudflare-modules.yaml` built three platforms on every trigger. Two of those platforms, `linux/arm64` and `linux/arm/v7`, run under QEMU emulation on the runner, and that emulation is what made a run take about 30 minutes.

The workflow has a `pull_request` trigger and a `push` trigger on `main` with the same `paths` filter. A pull request that touched `Dockerfile-cloudflare`, `.dockerignore` or the workflow file therefore ran a full 30-minute three-platform build that pushed nothing, and the merge commit then ran the same 30-minute build again on `main`. The GitHub Actions cache does not bridge the two runs, because a cache entry written on a pull-request ref cannot be read from `main`.

After this run, a pull-request build makes one native `linux/amd64` image and exports no cache. A `push`, `schedule` or `workflow_dispatch` build is unchanged: three platforms, cache export, image published.

## Build result per package

**Package 1 - pull-request builds only `linux/amd64`. Finished.** Commit `d428dd0`. The `platforms:` input of the `Build and push` step is now `${{ github.event_name == 'pull_request' && 'linux/amd64' || 'linux/amd64,linux/arm64,linux/arm/v7' }}`.

**Package 2 - pull-request runs read the build cache but never write it. Finished.** Commit `6bee751`. `cache-from` stays `type=gha`, so a pull-request build still restores layers. `cache-to` is now `${{ github.event_name != 'pull_request' && 'type=gha,mode=max' || '' }}`, so a pull-request build no longer spends time exporting and uploading a cache entry that only its own ref could ever read.

No package was frozen, paused or dropped. No package turned out to be moot.

## Waves

**Wave 1 - Package 1. Sequential. Wave 2 - Package 2. Sequential.**

Neither wave was a shared-artifact fallback. The shared-artifact catalogue matched nothing, and the repo probe found no lockfile, no package manifest, and a `.gitignore` holding only `.agents/`. The two packages simply declare the same file, so their scopes are not disjoint and the wave cut serialises them. That is the ordering rule working as intended, not a fallback.

## Inspector verdicts

**Package 1.** Inspector A, a fresh `verifier` on the exact acceptance claim: CONFIRMED. Inspector B, an `executor` in a read-only correctness posture: spec compliant, quality approved, no Critical and no Important findings.

**Package 2.** Inspector A: CONFIRMED, with two evidentiary gaps stated as advisories: `actionlint` is absent from this machine, and no live Actions run was available to observe `docker/build-push-action` treating an empty `cache-to` as no export. Inspector B: spec compliant, quality approved, no Critical and no Important findings; it also confirmed that no other step depends on a cache export existing and that the `Write build summary` step states nothing untrue for a pull-request run.

**Final whole-branch review.** No blocker for merge. It confirmed that the two expressions do not interact, that `schedule` and `workflow_dispatch` still get all three platforms and the cache export, and that a fork pull request still behaves sanely even though the `Login to GHCR` step is skipped for it, because a pull-request build never pushes.

## Decisions taken inside the run window

**1. The literal wording of brief point 3 was corrected before the cut.** The brief asked for `cache-from: type=gha,scope=main` so a pull-request run would read the `main` cache. That change does nothing. `scope` in the buildx gha backend is an internal namespace for several cache lines in one repository, not a branch selector, and plain `type=gha` already walks the Actions cache restore chain: current ref, then pull-request base ref, then default branch. A pull-request run therefore already read the `main` cache. The real waste in that area is the opposite direction, the pull-request run writing a cache entry nothing can read, so Package 2 removes that write instead. **If this is wrong:** the run shipped something other than the literal request. The literal version is one line and can be added later at no risk.

**2. Packages 1 and 2 stayed two dispatches instead of one batch.** Both are single-line edits of the same shape in one file, which normally gets batched. The approved cut gives each package its own acceptance claim and its own inspector pair, and batching would have merged those two review surfaces. **If this is wrong:** one extra implementer dispatch and one extra inspector pair were spent, with no correctness risk.

**3. The missing `actionlint` was accepted as an unverifiable check, not a failed one.** `actionlint` is not installed on this machine. `yq` parses the file, both inspectors of both packages read the rendered expressions and walked every event name through them, and the pull request this run opens is itself a build on the changed paths, so the workflow is exercised for real before merge. **If this is wrong:** a lint-level defect only `actionlint` catches reaches the pull request, where the run fails visibly and cheaply.

**4. The stale OpenWiki page was left alone.** `openwiki/workflows/build-pipeline.md` states the platform list and the cache setting as unconditional facts, and both are now event-dependent. The page is generated: this repository's `CLAUDE.md` says the scheduled OpenWiki workflow refreshes it and that generated pages are not to be hand-edited. **If this is wrong:** the wiki page contradicts the workflow until the next scheduled OpenWiki run, and a reader who trusts the page believes a pull request still builds three platforms.

## Deferred minor findings

None of these block the merge, and the final whole-branch review triaged all three as such.

- `actionlint` cannot run on this machine, so no workflow change in this repository can be lint-checked before it is pushed.
- `Set up QEMU` and `Set up Buildx` still run unconditionally, although a pull-request build is now native `linux/amd64`. Excluded by the plan's own non-goals.
- The empty-string branch of the `cache-to` expression is an indirect way to say "no cache export", though it matches the idiom already used on the two lines above it.

## Cap or stop reason

None. The run closed normally. Nothing was frozen, nothing paused, no cap was reached.

## Brief coverage

Both points of the brief are assigned to a package and both packages are finished. Point 1 became Package 1. Point 3 became Package 2, with the correction in decision 1 above folded in before the cut was approved. Nothing in the brief is unassigned and no package is open.

## Ingest status per store

- **mengram:** written. One `checkpoint` after wave 1 and one final `checkpoint` at run close.
- **ADR store:** written. One `manage_adr` call at run close, read then merged then updated, never a blind replace.

## Proposed follow-ups

The line proposes these and files nothing. They need a separate go-ahead.

**Cluster A - remaining pull-request CI cost.** One member: `Set up QEMU` and `Set up Buildx` run on every event, although a pull-request build is now native `linux/amd64` and needs no emulation. Small, seconds rather than minutes. A go-ahead would file one plain issue.

**Cluster B - workflow changes have no local verification.** One member: `actionlint` is not installed on this machine, so every GitHub Actions change in this repository is verified only by a YAML parse and by reading it, until the pull request runs. A go-ahead would file one plain issue, most usefully asking for `actionlint` in the toolchain or as a repository workflow step.

**Cluster C - the OpenWiki build-pipeline page contradicts the workflow.** One member: `openwiki/workflows/build-pipeline.md` states the platform list and the cache setting as unconditional. It should correct itself at the next scheduled OpenWiki run, so the follow-up is to confirm that it did, and to file the drift as a bug if it did not.
