# Run record: 2026-08-23-pr-build-cost

## Run header

- **Brief:** free text, `option 1 & 3`, resolved against the two named options from the preceding analysis of why the build workflow runs twice for about 30 minutes each.
- **Source adapter:** free text.
- **Token budget:** default, about 1.5M. **Concurrency cap:** 4.
- **Run branch:** `claude/pr-build-cost`, worktree `D:\Hold\VS Code\.worktrees\caddy-modules\pr-build-cost`.
- **Target repository:** `smoochy/caddy-modules`, branched from `main` at `5c9015e`.
- **Plan:** `docs/assembly-line/2026-08-23-pr-build-cost/plan.md`.

## The package cut

**Package 1 - pull-request builds only `linux/amd64`.** Declared file scope `.github/workflows/build_cloudflare-modules.yaml`, line 584. Acceptance claim: on a `pull_request` event the `Build and push` step receives `platforms: linux/amd64`; on `push`, `schedule` and `workflow_dispatch` it receives `platforms: linux/amd64,linux/arm64,linux/arm/v7`; the file still parses as valid YAML and `actionlint` reports no new error.

**Package 2 - pull-request runs read the build cache but never write it.** Declared file scope `.github/workflows/build_cloudflare-modules.yaml`, lines 596-597. Acceptance claim: `cache-from` is unchanged at `type=gha`; on a `pull_request` event `cache-to` resolves to the empty string, and on every other event it resolves to `type=gha,mode=max`; the file still parses as valid YAML and `actionlint` reports no new error.

## Gate verdicts on the cut

- **`plan-verifier`:** READY, first submission, no REVISE epoch.
- **`security-reviewer` pre-pass:** not run. The security trigger matched nothing in either package's declared file scope, so the run recorded no security hit and skipped the pass.
- **Human approval:** given on the presented cut, both packages, before any builder started.

## Waves

- **Wave 1 - Package 1.** Sequential. Not a shared-artifact fallback: the two packages simply declare the same file, so their scopes are not disjoint and the wave cut serialises them. The shared-artifact catalogue matched nothing, and the repo probe found no lockfile, no package manifest, and a `.gitignore` holding only `.agents/`.
- **Wave 2 - Package 2.** Sequential, same reason.

## Per package

**Package 1.** Implementer: generic, cheapest tier, returned DONE_WITH_CONCERNS (the concern was that `actionlint` is absent from this machine). Commit `d428dd0`.

- Inspector A, fresh `verifier` on the exact acceptance claim: **CONFIRMED**. It walked all four event names through the `&&`/`||` idiom, confirmed both branches yield a non-empty platform string so the falsy-middle-operand trap does not apply, and confirmed `yq` parses the file and round-trips the expression unchanged. It reported the `actionlint` sub-claim as unverifiable in this environment rather than failed.
- Inspector B, `executor` in a read-only correctness and simplification posture: spec **✅**, quality **approved**, no Critical and no Important findings. It checked that no step summary text becomes untrue, since the pull-request summary line makes no platform claim, and that the four non-goals were respected.

**Package 2.** Recorded below after wave 2 closes.

## Decide-and-log entries

Every entry is also in the ledger at `.superpowers/sdd/plan/progress.md`.

1. **Ruling on brief point 3.** The brief asked for `cache-from: type=gha,scope=main`. A correction was folded into the cut before approval instead of building it as written: `scope` in the buildx gha backend is an internal namespace for several cache lines in one repository, not a branch selector, and plain `type=gha` already walks the Actions cache restore chain, current ref then pull-request base ref then default branch, so the literal request would have changed nothing. Package 2 removes the pull-request `cache-to` export instead, which is the real waste in that area. **Cost if wrong:** the run ships a change the operator did not literally ask for; the literal version is one line and can still be added at no risk.
2. **Ruling on dispatch granularity.** Packages 1 and 2 stayed two dispatches rather than being batched into one, although both are single-line edits of the same shape in one file and the batching rule would otherwise apply. The approved cut gives each package its own acceptance claim and its own inspector pair, and batching would merge those two review surfaces. **Cost if wrong:** one extra implementer dispatch and one extra inspector pair, no correctness risk.
3. **Ruling on the missing `actionlint`.** `actionlint` is not installed on this machine, and its absence was accepted as an unverifiable sub-claim rather than a failed one. `yq` parses the file, both inspectors read the rendered expression, and the pull request this run opens is itself a build on the changed paths, so the workflow is exercised for real before merge. **Cost if wrong:** a lint-level defect only `actionlint` catches reaches the pull request, where the run fails visibly and cheaply.

## Ingest status per store

Recorded at run close.

## Findings outside the packages

- `Set up QEMU` and `Set up Buildx` still run unconditionally, although a pull-request build is now native `linux/amd64` on an `ubuntu-latest` runner. Raised by Package 1's inspector B and out of scope by the plan's own non-goals. Proposed under the follow-up cluster **pull-request CI cost, remaining items**.
- `actionlint` is not available in this environment, so no workflow change in this repository can be lint-checked locally. Raised by Package 1's implementer and confirmed by both of its inspectors. Proposed under the follow-up cluster **workflow verification has no local check**.
