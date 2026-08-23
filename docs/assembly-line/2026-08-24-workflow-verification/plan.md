# Plan: give the build workflow an executable verification path

## Goal

`.github/workflows/build_cloudflare-modules.yaml` cannot be executed locally, so every change to it has been verified statically: a YAML parse, `shellcheck` on the extracted `run:` block bodies, and a manual trace of each possible value through its consumers. Nothing has exercised a live Actions run, a real `crane copy` or a real `docker/metadata-action` invocation. This run decides what verification the repository wants, adds the one executable check that is cheap and covers the class of change the workflow actually receives, and writes the resulting standard down so that future changes do not re-litigate it.

## Source brief

GitHub issue adapter, <https://github.com/smoochy/caddy-modules/issues/20>, "The build workflow has no executable verification path". Body verbatim:

> Follow-up from assembly-line run `2026-08-23-upstream-v-tag` (PR #18).
>
> `build_cloudflare-modules.yaml` cannot be executed locally, so every verification in that run was static: a YAML parse, `shellcheck` on the extracted `run:` block bodies, and a manual trace of both possible `caddy_tag` values through each consumer. Nothing exercised a live Actions run, a real `crane copy` or a real `docker/metadata-action` invocation. Both inspectors of the workflow package raised this caveat independently, and the final review judged it non-blocking for that PR because the change reuses a guard the existing `caddy-<x.y.z>` tag already proves in production.
>
> The open question is whether this repository wants a local dry-run path for its build workflow at all. Candidates, none evaluated yet:
>
> - `act` for a local Actions run, at the cost of a Docker-in-Docker setup that will not reproduce the multi-arch push.
> - A pull-request-scoped job that runs the `decide` and `meta` steps with `push: false` and asserts the resulting tag list, which would cover exactly the class of change this PR made.
> - Accepting static verification as the standard for this workflow and writing that down, so future changes do not re-litigate it.

## Context the builder needs

- The build workflow is `.github/workflows/build_cloudflare-modules.yaml`.
- It already triggers on `pull_request` against `main`, filtered to the paths `Dockerfile-cloudflare`, `.dockerignore` and the workflow file itself.
- It already builds without publishing on a pull request: `docker/build-push-action@v7` receives `push: ${{ github.event_name != 'pull_request' }}` and a single-platform `platforms` value. So the "runs with `push: false`" half of the brief's second candidate exists today. What is missing is any assertion on the result.
- The `decide` step emits `caddy_tag`, which has exactly two possible shapes: a three-part semver such as `2.11.4`, or the literal string `latest` when neither the image label nor the image history yields a semver.
- The `meta` step is `docker/metadata-action@v6`. It runs only under `if: steps.decide.outputs.do_build == 'true'` and lists three raw tags: `latest`, `caddy-<caddy_tag>`, and `caddy-v<caddy_tag>` enabled only when `caddy_tag != 'latest'`.
- `do_build` is not always `true` on a pull request. The trigger's path filter evaluates the diff of the pushed commits, while the `Classify local changes` step independently diffs `PR_BASE_SHA` against `PR_HEAD_SHA`. A pull request whose later commit reverts an earlier change to a filtered path re-triggers the workflow while `image_inputs_changed` and `workflow_changed` are both false, so with no upstream change `do_build` is `false` and the `meta` step is skipped. Any new step that reads `steps.meta.outputs.tags` must handle that state.
- `steps.meta.outputs.tags` is a newline-separated list of fully qualified references, each of the shape `ghcr.io/smoochy/caddy-cloudflare-modules:<tag>`.
- The `decide` step needs the network. It calls `crane` against `caddy:latest` and the GitHub API for release metadata, so it cannot run offline, which is what rules the workflow out of a genuinely local dry run.

## Package 1 - the pull request run asserts the tag list it produced

**Outcome**: a pull request that changes the tag logic fails in CI when the produced tag set is wrong, instead of relying on a human tracing the branches by hand.

**Declared file scope**: `.github/workflows/build_cloudflare-modules.yaml`

**Point from the brief**: "A pull-request-scoped job that runs the `decide` and `meta` steps with `push: false` and asserts the resulting tag list, which would cover exactly the class of change this PR made."

**Acceptance claim**:

1. A new assertion step is added after the `meta` step and before the `Build and push` step, guarded by `if: github.event_name == 'pull_request' && steps.decide.outputs.do_build == 'true'`.
2. When that guard holds, the step reads `steps.meta.outputs.tags` and `steps.decide.outputs.caddy_tag` and fails the job unless the produced tag set is exactly the expected set for that `caddy_tag`: `latest`, `caddy-<tag>` and `caddy-v<tag>` when `caddy_tag` is a three-part semver, and `latest` and `caddy-latest` when `caddy_tag` is the literal string `latest`.
3. The comparison is by set membership plus a count, in either order. It is never an exact string comparison of the whole rendered list and never depends on line order, so a change in how `docker/metadata-action` formats its output cannot fail the job on its own.
4. When `steps.decide.outputs.do_build` is `false` on a `pull_request` event, the `meta` step is skipped and `steps.meta.outputs.tags` is empty. The assertion step is skipped by the same guard and does not fail the job. This is required behaviour, not an oversight.
5. On `push`, `schedule` and `workflow_dispatch` events the assertion step never runs, so no published build can be failed by it.
6. The step adds no permission, no secret, no new third-party action, and no change to push, tag, label or mirror behaviour on any event.
7. The failure output names the expected set and the produced set, so a failing run is diagnosable from the log alone.
8. The workflow file remains valid YAML and the new shell block is `set -euo pipefail` clean.

## Package 3 - the `act` candidate is evaluated

**Outcome**: the repository has a recorded ruling on `act`, so the option is closed rather than left open.

**Declared file scope**: none. This package produces a ruling, not a diff. The ruling is carried into the section Package 2 writes.

**Point from the brief**: "`act` for a local Actions run, at the cost of a Docker-in-Docker setup that will not reproduce the multi-arch push."

**Acceptance claim**:

1. The run states whether `act` is adopted or rejected, with reasons drawn from this repository rather than from general argument.
2. The reasons address at minimum: that the `decide` step needs network access to `crane` and the GitHub API, so a local run is not hermetic; that `act` cannot reproduce the multi-arch `docker/build-push-action` push, which is the part no static check covers; and what a local `act` run would therefore actually prove.
3. The ruling appears in the run record and in the section Package 2 writes.

## Package 2 - the repository states its verification standard for this workflow

**Outcome**: a contributor changing this workflow reads what is checked where, and what is not checked before merge, without re-opening the question.

**Declared file scope**: `README.md` (a new section), `CLAUDE.md` (one pointer line, appended outside the `OPENWIKI:START` and `OPENWIKI:END` markers)

**Point from the brief**: "Accepting static verification as the standard for this workflow and writing that down, so future changes do not re-litigate it."

**Acceptance claim**:

1. The new `README.md` section states what each verification layer covers: the pull request tag assertion Package 1 adds, the static checks a change to this workflow is expected to run locally, and the live scheduled run.
2. It states explicitly what is not verified before a merge: a real multi-arch push, a real `crane copy` to Docker Hub, and a real write to either registry.
3. It records the `act` ruling from Package 3 with its reason.
4. It names the static checks concretely enough to repeat them, rather than saying "static verification" and stopping.
5. It does not contradict the tag list `README.md` already documents in its "Image Tags" and "Available Images" sections, and it does not restate that list.
6. `CLAUDE.md` gains one pointer line to the new section, appended outside the OpenWiki marker block, so the generated content stays untouched.
7. No section of `README.md` other than the new one changes.
8. Both edits are Simplified Technical English, with no hard-wrapped prose and no em dashes or en dashes.

## Waves

- **Wave 1**: Package 1 and Package 3.
- **Wave 2**: Package 2. Runs after wave 1.

Both waves are sequential. Package 3 touches no file, so it conflicts with nothing; wave 1 is sequential because this version of the line runs every wave sequentially, not because a shared artifact forced a fallback. Wave 2 is a content dependency rather than a file conflict: Package 2's section describes the assertion Package 1 builds and carries the ruling Package 3 makes, so writing it first risks documenting a shape the inspection then changes.

## Triggers

- **Shared artifacts**: no hit. No lockfile, no manifest, no snapshot directory, no formatter or linter config is in any declared scope. The repo probe read `.gitignore`, which holds the single entry `.agents/`, and the CI configs, which run no repo-wide formatter. The generated `openwiki/` tree exists but no package declares it and no package may hand-edit it.
- **Security**: no hit. No declared file scope matches the trigger vocabulary by path, and no package adds, reads or changes a secret, a permission block, a login step or a credential. The pre-approval `security-reviewer` pass is skipped and that skip is recorded.

## Non-goals

- No change to what the build publishes, and no change to push, tag, label or mirror behaviour on any event.
- No new third-party action, and no new permission or secret.
- No adoption of `act`, and no Docker-in-Docker setup.
- No hand-edit of the generated `openwiki/` tree.
- No cleanup of the unrelated stale `claude/actionlint-ci` worktree and its uncommitted `lint_workflows.yaml`. That is outside this brief and is proposed as a follow-up instead.
