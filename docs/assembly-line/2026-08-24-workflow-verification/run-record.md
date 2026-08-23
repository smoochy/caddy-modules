# Run record: 2026-08-24-workflow-verification

## Run header

- **Source adapter**: GitHub issue. <https://github.com/smoochy/caddy-modules/issues/20>, "The build workflow has no executable verification path". Fetched with `gh issue view --json title,body,labels`. No `wayfinder:map` label, so the plain issue adapter applied and title plus body became the brief text.
- **Brief hash**: the issue body as of 2026-08-24, quoted verbatim in `plan.md`.
- **Token budget**: roughly 1.5M, the default. Not overridden.
- **Concurrency cap**: 4, owned by the run.
- **Run branch**: `claude/workflow-verification`, in the linked worktree `.worktrees/caddy-modules/workflow-verification`.
- **Target repo**: `smoochy/caddy-modules`.

## The package cut

| # | Outcome | Declared file scope | Acceptance claim |
| --- | --- | --- | --- |
| 1 | A pull request that changes the tag logic fails in CI when the produced tag set is wrong. | `.github/workflows/build_cloudflare-modules.yaml` | Eight parts, listed in full in `plan.md`. In summary: a new assertion step after `meta` and before `Build and push`, guarded on `pull_request` and on `do_build == 'true'`, comparing the produced tag set against the expected set for that `caddy_tag` by membership plus count, skipped in every other state, adding no permission, secret or action, printing both sets on failure, and leaving the file valid YAML with a `set -euo pipefail` clean block. |
| 3 | The repository has a recorded ruling on `act`. | none, the package produces a ruling rather than a diff | The run states whether `act` is adopted or rejected, with reasons drawn from this repository, and the ruling appears in this record and in the section Package 2 writes. |
| 2 | A contributor changing this workflow reads what is checked where, and what is not checked before merge. | `README.md`, `CLAUDE.md` | Eight parts, listed in full in `plan.md`. |

The numbering follows the brief's own order of candidates, so Package 3 is the `act` candidate and runs in wave 1 while Package 2 is the documentation candidate and runs in wave 2.

## Pre-run gates

The cut went to a fresh `plan-verifier`, which returned **REVISE** on one blocker: Package 1's acceptance claim did not state the assertion step's behaviour when `steps.decide.outputs.do_build` is `false` on a `pull_request` event, although the `meta` step it reads is itself gated on that output. The reviewer showed a real path into that state: the trigger's path filter evaluates the diff of the pushed commits, while the `Classify local changes` step diffs `PR_BASE_SHA` against `PR_HEAD_SHA`, so a pull request whose later commit reverts an earlier change to a filtered path re-triggers the workflow with both change flags false. The controller had asserted the opposite when drafting the cut, and the reviewer refuted it.

An advisor pass ran concurrently with the `plan-verifier` pass. It made two changes to the cut. It required the same `do_build` guard from the other direction, and it required the comparison to be set membership rather than exact list equality, so the assertion does not couple to how `docker/metadata-action` renders its output. It also moved Package 2's target from a new `docs/workflow-verification.md` to a section of the existing `README.md`, on the grounds that a repository with one build workflow and no engineering-standards document does not justify creating that layer.

The cut was revised materially on all three points and submitted once to a fresh `plan-verifier`, which returned **READY**. That closed the second readiness epoch. The revised cut was then presented and the operator approved it, which opened the run window.

**Security pre-pass**: skipped, and the skip is recorded here. The security trigger did not fire. No declared file scope matches the trigger vocabulary by path, and no package adds, reads or changes a secret, a permission block, a login step or a credential.

## Waves

| Wave | Packages | Ran | Reason |
| --- | --- | --- | --- |
| 1 | Package 1, Package 3 | Sequential | No shared-artifact trigger fired. The repo probe read `.gitignore`, which holds the single entry `.agents/`, and the CI configs, which run no repo-wide formatter; there is no lockfile, no manifest and no snapshot directory in any scope. The parallel-wave flag is off by default and was not set, and in this version it would not run real concurrency anyway. Package 3 declares no file scope, so it could not conflict with Package 1 in any case. |
| 2 | Package 2 | Sequential, after wave 1 | A content dependency, not a shared-artifact fallback: Package 2's section describes the assertion Package 1 builds and carries the ruling Package 3 makes, so writing it first risked documenting a shape the inspection then changed. |

## Per package

### Package 1

Built by SDD's generic implementer on a standard model, since the security trigger did not fire and no `security-executor` was required. One commit, `1832287`. A 46 line pure insertion in one file.

| Inspector | Role | Verdict |
| --- | --- | --- |
| A | fresh `verifier` | **CONFIRMED**, all eight parts. It re-executed the extracted assertion logic rather than reading it, covering the four cases the brief required plus five it added itself: a trailing empty line in the tag list, which is the realistic shape of a newline separated action output; a duplicate entry whose count coincidentally matches the expected count while an expected tag is missing; an empty `caddy_tag`; a `caddy_tag` carrying a shell metacharacter, which passed safely because the value only ever reaches the script through the `env:` block and inside quotes; and the consistency of bare tag against fully qualified reference on the two sides of the comparison. It re-parsed the whole workflow with a real YAML parser and re-ran `shellcheck` clean. |
| B | `executor`, read-only correctness and simplification posture | No Critical and no Important finding. It independently re-ran `shellcheck` and six assertion cases, and it established four things the claim does not cover: that `CADDY_TAG` and `META_TAGS` never reach the script body as interpolated expressions, so the untrusted `caddy_tag` cannot arrive as code; that the `[ -z "$line" ] && continue` idiom is exempt from `set -e` under the `&&` list rule and does not exit the loop early; that the here-string feeding the read loop keeps the array alive, where a pipe would have built it in a subshell and discarded it; and that the new step's guard is mutually exclusive with the Docker Hub mirror step and does not mislead the `if: always()` build summary step. One Minor finding, deferred, listed below. |

Both inspectors ran concurrently. No fix round opened.

### Package 3

No implementer was dispatched. The package declares no file scope and produces a ruling rather than a diff, so the controller adjudicated it directly; that decision is a ledger ruling and is repeated under Ledger decisions below. The package is reported as moot in the sense the cut defines: it was a real point of the brief, it was worked, and its outcome is a recorded rejection rather than a change.

**The ruling**: `act` is rejected. Three reasons drawn from this repository. The `decide` step calls `crane` against `caddy:latest` and the GitHub API for release metadata, so an `act` run is not hermetic. `act` cannot reproduce the multi-arch `docker/build-push-action` push or the `crane copy` mirror, which is precisely the part no static check covers, so it does not close the gap the issue names. What a local `act` run would actually prove is that the YAML parses and the shell branches execute, and `shellcheck` plus the Package 1 assertion already prove that more cheaply and in the place that gates a merge. Against that, the option costs a Docker-in-Docker setup in a repository with no other local toolchain.

### Package 2

Recorded after wave 2.

## Ledger decisions

Every decide-and-log entry made inside the run window, in the order it was made.

1. **No unit test is expected for any package.** The repository has no test framework, no test directory and no test runner, and the only artifacts under change are one workflow and two documents. The runnable check that replaces a test for Package 1 was named in the dispatch and required in the report: a YAML parse of the whole workflow, `shellcheck` on the new `run:` block, and a local execution of the assertion logic against both `caddy_tag` shapes plus wrong tag sets, proving it passes and fails where it should. *Cost if wrong*: the assertion ships with a logic error that only a live pull request run would surface. That is no worse than the exposure the repository has today, and strictly better, because the assertion also runs live on the very pull request that introduces it.
2. **Package 3 gets no implementer dispatch.** It declares no file scope and produces a ruling, so dispatching a subagent to write no code would add a review surface with nothing on it. *Cost if wrong*: the `act` ruling carries one person's reasoning rather than an independent builder's, and a reader who disagrees reopens the question against the reasons recorded above.
3. **`act` is rejected.** Reasons above. *Cost if wrong*: a contributor who wanted a full local rehearsal does not get one and must open a draft pull request instead, which this repository's pull request path already builds without publishing.

## Findings outside the packages

| Finding | Proposed cluster |
| --- | --- |
| Package 1, Minor, deferred: the tag-set check is a hand-rolled count plus nested membership scan, where a sort and compare would be materially shorter and equally strong, because the expected list never holds a duplicate. Inspector B states the current code is correct and proposes the change as a simplification only. | Not clustered. Carried to the final whole-branch review for triage rather than proposed as a follow-up, because it lives inside this run's own diff. |
| A stale linked worktree `.worktrees/caddy-modules/actionlint-ci` sits on branch `claude/actionlint-ci` and holds an untracked `.github/workflows/lint_workflows.yaml` that would run `rhysd/actionlint` on workflow changes. It was never committed and never pushed. It is outside this brief, which asks about executable verification of the build workflow rather than linting, but it is adjacent to the standard Package 2 writes down. | Cluster A, recorded in the report. |

## Ingest status per store

Recorded at run close.
