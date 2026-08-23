# Report: 2026-08-23-upstream-v-tag

## What the run changed

The container image `ghcr.io/smoochy/caddy-cloudflare-modules` was published with two tags: `latest` and `caddy-<x.y.z>`, for example `caddy-2.11.4`. Upstream Caddy names the same release `v2.11.4`. From now on every build publishes a third tag, `caddy-v<x.y.z>`, and mirrors it to Docker Hub. A deployment can pin an image version that reads exactly like the upstream release tag. The two existing tags are unchanged.

The third tag is published only when the build detects a real version number. If it cannot, it publishes `latest` alone. The tag `caddy-vlatest` can never be produced.

## Build result per package

| Package | Outcome | Result |
| --- | --- | --- |
| 1 - the workflow publishes and mirrors `caddy-v<x.y.z>` | The build emits, pushes and mirrors the new tag, and names it in the build summary. | Done. One commit, `39ba695`. Both inspectors clean on the first round. No fix round. |
| 2 - the documentation states both tag shapes | `README.md` and `README.dockerhub.md` name the new tag everywhere they list published tags, and the pinning example covers both shapes. | Done. One commit, `90758fa`. Both inspectors clean on the first round. No fix round. |

No package was moot, frozen, paused or dropped. Every point of the brief is assigned to a package; the reread table is in the run record.

## Waves

| Wave | Package | Ran | Reason |
| --- | --- | --- | --- |
| 1 | Package 1 | Sequential | No shared-artifact trigger fired. The parallel-wave flag is off by default and was not set, and in this version it would not run real concurrency anyway. |
| 2 | Package 2 | Sequential, after wave 1 | A content dependency, not a fallback: the documentation states the tag shape the workflow emits, so writing it first risked documenting a shape the inspection then changed. |

No wave fell back to sequential because of a shared artifact. No builder wrote outside its declared file scope.

## Inspector verdicts

| Package | Inspector A, fresh `verifier` | Inspector B |
| --- | --- | --- |
| 1 | CONFIRMED, all six parts of the acceptance claim, each reproduced independently. Stated caveat: the evidence is static. | `executor`, read-only correctness and simplification posture: no findings at any severity. It also established that the three touched sites are the only tag-enumeration sites in the workflow. |
| 2 | CONFIRMED, all five parts plus the style constraints. It cross-checked the documented tag set against the workflow's real tag-emitting sites. | `executor`, read-only correctness and simplification posture: no findings at any severity. It searched both README files exhaustively for further tag enumerations and found none left stale. |

Pre-run, the package cut went to a `plan-verifier`, which returned REVISE on one blocker: one of the two `README.md` sections that enumerate published tags sat outside the documentation package's scope, which would have left the file self-contradictory. The cut was revised and a fresh `plan-verifier` returned READY. An advisor pass, run concurrently, removed an unnecessary step output from the workflow package. The security trigger did not fire, so the pre-approval security pass was skipped, and that skip is recorded.

The final whole-branch review found no Critical issue. It found one Important issue, that the run record committed after wave 1 still described wave 2 as pending, which this run's closing commit fixes. Its two Minor findings are carried below.

## Ledger decisions

The run made no decide-and-log entry. Both packages passed both inspectors on the first round, so the fix loop never opened, the breaker never came into play, and no conflict, ambiguity or cap event required a ruling inside the run window.

## Cap and stop reason

None. The run closed normally, well inside its token budget, with no pause and no stop condition.

## Ingest per store

- **mengram**: success. `checkpoint` was called after wave 1 and again unconditionally at close.
- **ADR store**: reported at the end of this run; see the run record for the result.

Both are reported separately from each other and separately from the build result, because ingest is best effort and never fails a run.

## Proposed follow-ups

The line proposes these and files nothing. They need a go-ahead.

### Cluster A - post-merge publication steps

One revision decides both, and both are operator actions rather than code changes.

- The new tag does not exist for the currently published Caddy version until the next real upstream change, because the workflow only pushes when its rebuild gate says a rebuild is needed. Running the build workflow through `workflow_dispatch` with `force=true` once, after the merge, publishes `caddy-v<x.y.z>` for the current version immediately.
- The generated `openwiki/` index still says the build publishes two tags. Its own refresh workflow runs on even ISO weeks and will correct it, but running that workflow manually after the merge closes a drift window of about two weeks.

### Cluster B - the workflow has no executable verification path

A single finding, so a plain issue rather than a map. Every check in this run was static: a YAML parse, `shellcheck` on extracted shell blocks, and a manual trace of both possible version values. Nothing exercised a live Actions run, a real registry copy or a real metadata action. Both inspectors of package 1 raised the caveat independently, and the final review judged it non-blocking because the change reuses a guard the existing tag already proves in production. The open question is whether this repository wants a local dry-run path for its build workflow at all.
