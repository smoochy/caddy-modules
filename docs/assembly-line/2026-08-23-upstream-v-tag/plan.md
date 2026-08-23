# Plan: publish an upstream-shaped `caddy-v<x.y.z>` image tag

## Goal

The container image at `ghcr.io/smoochy/caddy-cloudflare-modules` is published today as `latest` and `caddy-<x.y.z>` (for example `caddy-2.11.4`). Upstream Caddy names the same release `v2.11.4` (<https://github.com/caddyserver/caddy/releases/tag/v2.11.4>). From now on the image must additionally carry the upstream-shaped tag, for example `caddy-v2.11.4`, so a deployment can pin an image version that reads exactly like the upstream release tag. The existing `caddy-<x.y.z>` tag stays, unchanged, and `latest` stays.

## Source brief

Verbatim (German, free-text adapter):

> die releases auf https://github.com/smoochy/caddy-modules/pkgs/container/caddy-cloudflare-modules laufen immer auf "caddy-<version>", zb "caddy-2.11.4". zusätzlich soll absofort auch noch das tagging, welches auch im upstream verwendet wird dabei sein
>
> https://github.com/caddyserver/caddy/releases/tag/v2.11.4
>
> so das es zusätzlich zb "caddy-v2.11.4" als image version gibt und man das fürs image pinning verwenden kann. damit ist es synchron zum upstream

## Context the builder needs

- The build workflow is `.github/workflows/build_cloudflare-modules.yaml`.
- The `decide` step derives `caddy_tag` at line ~140 by reading the `org.opencontainers.image.version` label of `caddy:latest` and stripping a leading `v` (`CADDY_TAG="${CADDY_TAG_RAW#v}"`). When neither the label nor the image history yields a semver, `CADDY_TAG` falls back to the literal string `latest`. A naive `caddy-v${CADDY_TAG}` would then publish the meaningless tag `caddy-vlatest`.
- By the end of that step `caddy_tag` has exactly two possible shapes, a three-part semver or the literal string `latest`, because `is_semver()` already filtered the label path and the history fallback captures `x.y.z` only. So `caddy_tag != 'latest'` is the guard, and no new step output is needed to re-derive it.
- The leading `v` is stripped exactly once at line ~141, so prepending one cannot produce `vv2.11.4`.
- The tag list is produced by `docker/metadata-action@v6` in the `meta` step (`type=raw,value=latest` plus `type=raw,value=caddy-<caddy_tag>`).
- The Docker Hub mirror step copies a fixed tag list with `crane copy` (`for tag in latest "caddy-${CADDY_TAG}"`).
- The "Write build summary" step prints the published tag list for GHCR and for Docker Hub separately.

## Package 1 - the workflow publishes and mirrors `caddy-v<x.y.z>`

**Outcome**: every pushed build additionally publishes and mirrors the upstream-shaped tag.

**Declared file scope**: `.github/workflows/build_cloudflare-modules.yaml`

**Point from the brief**: "zusätzlich soll absofort auch noch das tagging, welches auch im upstream verwendet wird dabei sein ... so das es zusätzlich zb 'caddy-v2.11.4' als image version gibt".

**Acceptance claim**:

1. No new `decide` step output is added. All three sites below gate on the existing `steps.decide.outputs.caddy_tag != 'latest'`.
2. The `meta` step lists a third raw tag, `type=raw,value=caddy-v<caddy_tag>,enable=<caddy_tag != 'latest'>`, so no build can ever publish `caddy-vlatest`.
3. The Docker Hub mirror step copies the new tag in addition to `latest` and `caddy-<x.y.z>`, and skips it when `caddy_tag` is `latest`.
4. The build summary lists the new tag under both "Image Tags" and "Docker Hub Mirror", and omits it when `caddy_tag` is `latest`.
5. `latest` and `caddy-<x.y.z>` are unchanged in name, count and ordering.
6. The workflow file remains valid YAML and the changed shell stays `set -euo pipefail` clean.

## Package 2 - the documentation states both tag shapes

**Outcome**: a reader pinning an image learns that `caddy-v<x.y.z>` exists and is the upstream-synchronous form.

**Declared file scope**: `README.md`, `README.dockerhub.md`

**Point from the brief**: "man das fürs image pinning verwenden kann. damit ist es synchron zum upstream".

**Acceptance claim**:

1. `README.md` section "Image Tags" lists `caddy-v<x.y.z>` next to the existing `caddy-<x.y.z>`, and says it matches the upstream Caddy release tag name.
2. `README.md` section "Available Images", the "Publishes the image to:" bullet list at lines 70-77, names `caddy-v<x.y.z>` in both the GHCR list and the Docker Hub list, so the two places in that file that enumerate published tags agree. A grep for `caddy-v<x.y.z>` in `README.md` hits both this list and the "Image Tags" section.
3. The pinning example in `README.md` "Install" stays correct and mentions that either version tag shape can be pinned.
4. `README.dockerhub.md` section "Tags" lists `caddy-v<x.y.z>`.
5. No section other than those named in claims 1 to 4 changes in either file.

## Waves

- **Wave 1**: Package 1. Runs alone.
- **Wave 2**: Package 2. Runs alone, after wave 1.

Both waves are sequential. The file scopes of the two packages are disjoint, so no shared-artifact trigger and no file conflict forces this; the ordering is a content dependency, because the documentation states the tag shape the workflow actually emits, and writing it before the workflow is settled would risk documenting a shape that the first inspector then changes.

## Triggers

- **Shared artifacts**: no hit. No lockfile, no manifest, no generated directory, no snapshot directory, no formatter or linter config is in scope. The repo probe found `.gitignore` holding a single entry and no repo-wide formatter in CI.
- **Security**: no hit. The declared file scopes carry no authentication, authorization, secrets, credentials, identity, privacy, crypto or input-validation vocabulary. The workflow does read `DOCKERHUB_USERNAME` / `DOCKERHUB_TOKEN` secrets, but no package changes a secret, a permission or a login step, so the pre-approval `security-reviewer` pass is skipped and that skip is recorded.

## Non-goals

- No change to the `latest` tag, to the addon set, to the rebuild-detection logic, or to the image labels.
- No retroactive re-tagging of images already published; the brief says "ab sofort" (from now on), so the new tag appears on the next build. No backfill logic is added either: the workflow only pushes when `do_build` is true, which is existing behaviour that the current `caddy-<x.y.z>` tag already lives with. The operator can publish `caddy-v<x.y.z>` for the current version immediately by running the workflow through `workflow_dispatch` with `force=true`, and the run report states that.
- No change to the Docker Hub repository name or to registry credentials.
