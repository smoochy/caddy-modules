# Report: 2026-08-24-workflow-verification

## What the run was asked to do

GitHub issue <https://github.com/smoochy/caddy-modules/issues/20> asked one question and listed three candidate answers. The question: does this repository want a local dry-run path for its build workflow at all. The three candidates: `act` for a local Actions run, a pull-request-scoped job that asserts the tag list, and accepting static verification as the standard and writing it down.

## What the run did

The run cut the brief into three numbered packages, one per candidate, and ran them in two waves.

**Package 1** added an `Assert produced tag set` step to `.github/workflows/build_cloudflare-modules.yaml`. On a pull request, and only when the `decide` step says a build is needed, the step reads the tag list `docker/metadata-action` produced and fails the job unless that set matches what the `caddy_tag` value implies. It compares by set membership plus a count, so a change in how the action renders its output cannot fail the job on its own. It never runs on `push`, `schedule` or `workflow_dispatch`, so it cannot fail a publishing build.

**Package 3** evaluated `act` and rejected it. The `decide` step calls `crane` against `caddy:latest` and the GitHub API, so a local run is not hermetic. `act` cannot reproduce the multi-arch push or the `crane copy` mirror, which is the one part no static check covers. What a local run would prove is already proven more cheaply by the static checks and by the new assertion, and in the place that gates a merge.

**Package 2** wrote the standard into `README.md` as a new "Workflow Verification" section, with one table of contents entry and one pointer line in `CLAUDE.md`. The section names each verification layer, names the static checks concretely enough to repeat them, records the `act` ruling with its reason, and states plainly what is not verified before a merge: a real multi-arch push, a real `crane copy` to Docker Hub, and a real write to either registry.

## Closing pass against the brief

Every point of the brief is assigned to a package and every package is finished.

| Point in the brief | Package | State |
| --- | --- | --- |
| "`act` for a local Actions run" | 3 | Finished. Rejected, with reasons recorded in the run record and in the README section. |
| "A pull-request-scoped job that runs the `decide` and `meta` steps with `push: false` and asserts the resulting tag list" | 1 | Finished and shipped. Reduced to the assertion alone, because the pull request build with `push: false` already existed. |
| "Accepting static verification as the standard for this workflow and writing that down" | 2 | Finished and shipped, with one amendment and one fix round. |
| The framing question, "whether this repository wants a local dry-run path for its build workflow at all" | 3 and 2 together | Answered. The answer is no local dry-run path, and the reason is now written down where a contributor reads it. |

Nothing in the brief is left open.

## What the inspections caught

Every package had two inspectors, always concurrent: a fresh `verifier` on the exact acceptance claim, and a second reviewer in a read-only correctness and simplification posture.

The inspections earned their place three times, and each time on the same class of defect: a claim that was plausible and wrong.

Before the run, the `plan-verifier` refuted the controller's own assertion that `do_build` is always `true` on a pull request. It is not: the trigger's path filter evaluates the pushed commits' diff while the `Classify local changes` step diffs base to head, so a pull request whose later commit reverts an earlier change re-triggers the workflow with both change flags false. That refutation is why the assertion step carries a `do_build` guard rather than crashing on an empty tag list.

During wave 2, inspector B found that the section documenting the verification standard was itself wrong about the workflow: it claimed the workflow runs on every push to `main`, ignoring the path filter, and it claimed a real push and mirror "actually execute" on the next push or scheduled run, when in truth a push forces a build only on an image-input change, a scheduled run never forces one, and the mirror needs Docker Hub secrets. One fix round corrected both.

At the whole-branch review, the reviewer found that the branch predated PR #23, which added actionlint to `main` while this run was in flight. The README section therefore presented a set of checks as a local responsibility without naming the CI layer that now runs them automatically, and the run record asserted as current a fact that had since resolved. The branch was rebased and both were corrected.

## What was deferred

The tag-set check is a hand-rolled count plus a nested membership scan, where a sort and compare would be shorter. The whole-branch reviewer agreed with deferring it and showed why the two forms are equally strong here: the expected list is provably free of duplicates, so "same count and every expected element present" already forces set equality. It is a style simplification with no behaviour change and no residual risk.

## Follow-up proposed

The line proposes follow-ups and files nothing.

**Cluster A is withdrawn.** It held the stale `claude/actionlint-ci` worktree and its uncommitted `lint_workflows.yaml`. That work merged into `main` as PR #23 during this run, so there is nothing left to propose. The worktree itself can be removed whenever its owner is done with it, which is housekeeping rather than a follow-up.

No other follow-up is proposed. The brief is fully answered and the deferred simplification is not worth a ticket.
