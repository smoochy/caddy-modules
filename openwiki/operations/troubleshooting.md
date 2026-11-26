---
type: "Reference"
title: "Troubleshooting"
openwiki_generated: true
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---


# Troubleshooting

This page documents common issues encountered with the Caddy modules build pipeline and how to diagnose and resolve them.

## Build Not Triggering

### Scheduled Run Doesn't Build

**Symptom**: The daily scheduled workflow (03:00 UTC) runs but reports "No upstream changes detected" and skips the build.

**Cause**: The `decide` step compares the current upstream state (Caddy base image digest, addon versions) against the OCI labels stored in the published `ghcr.io/smoochy/caddy-cloudflare-modules:latest` image. If nothing changed, `CHANGED=false` and `do_build=false`.

**Verification**:
```bash
# Check the latest run summary
gh run view --workflow=build_cloudflare-modules.yaml --log

# Inspect the published image's base digest
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels["org.opencontainers.image.base.digest"]'

# Compare with current caddy:latest digest
crane digest caddy:latest
```

**Resolution**: This is expected behavior. The pipeline intentionally avoids rebuilding when nothing changed. To force a rebuild:
```bash
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true
```

### Push to `main` Doesn't Trigger Build

**Symptom**: Pushing to `main` doesn't start a workflow run.

**Cause**: The workflow uses path filtering. Only changes to these files trigger a build:
- `Dockerfile-cloudflare`
- `.dockerignore`
- `.github/workflows/build_cloudflare-modules.yaml`

Changes to `README.md`, documentation, or other files are intentionally ignored.

**Verification**:
```bash
# Check what files changed in the push
git diff HEAD~1 --name-only
```

**Resolution**: If you need a rebuild without file changes, use `workflow_dispatch` with `force=true`.

### Pull Request Validation Build Doesn't Run

**Symptom**: Opening a PR doesn't trigger the workflow.

**Cause**: Same path filtering applies to PRs. Only changes to Dockerfile, `.dockerignore`, or the workflow file itself trigger validation builds.

## Tag Assertion Failures on PR

### "Tag set assertion failed" Error

**Symptom**: The `Assert produced tag set` step fails on a PR with:
```
Tag set assertion failed for caddy_tag=2.8.4
Expected: latest caddy-2.8.4 caddy-v2.8.4
Produced: latest caddy-2.8.4
```

**Cause**: The `meta` step generates tags based on `caddy_tag`. When `caddy_tag` is a valid semver (e.g., `2.8.4`), three tags are expected: `latest`, `caddy-<version>`, and `caddy-v<version>`. The assertion validates this. If `caddy_tag` is `latest` (non-semver), only two tags are expected: `latest` and `caddy-latest`.

The failure typically occurs when:
- The Caddy base image's `org.opencontainers.image.version` label doesn't contain a valid semver
- The fallback logic in `decide` step extracted an unexpected version string

**Debugging**:
```bash
# Check what tags metadata-action produced
gh run view <run-id> --log | grep -A 20 "Metadata"

# Inspect the Caddy base image labels directly
crane config caddy:latest | jq '.config.Labels'

# Check the decide step outputs
gh run view <run-id> --log | grep -A 5 "caddy_tag="
```

**Resolution**: The `decide` step has fallback logic to extract version from image history if the label is missing. If that also fails, it defaults to `latest`. Verify the Caddy base image has proper labels.

## Docker Hub Mirror Failures

### Mirror Step Skipped Silently

**Symptom**: Build succeeds but Docker Hub mirror section in summary shows:
```
- Skipped: set `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` repository secrets to enable Docker Hub publishing.
```

**Cause**: Both `DOCKERHUB_USERNAME` and `DOCKERHUB_TOKEN` secrets must be configured in the repository settings. If either is missing, the `registries` step outputs `dockerhub_enabled=false`, and the mirror steps are skipped.

**Verification**:
```bash
# Check repository secrets (requires admin access)
gh secret list | grep DOCKERHUB
```

**Resolution**: Add both secrets in GitHub repository settings → Secrets and variables → Actions:
- `DOCKERHUB_USERNAME` (e.g., `smoochy84`)
- `DOCKERHUB_TOKEN` (Docker Hub access token, not password)

### Docker Hub Rate Limit Errors

**Symptom**: Mirror step fails with:
```
error copying image: denied: requested access to the resource is denied
```
or
```
toomanyrequests: You have reached your pull rate limit
```

**Cause**: Docker Hub enforces rate limits. Anonymous pulls: 100/6hr per IP. Authenticated: 200/6hr. The `crane copy` pulls from GHCR and pushes to Docker Hub, counting against the Docker Hub account's limits.

**Mitigation**:
- Ensure `DOCKERHUB_TOKEN` is a valid access token (not password)
- The workflow runs at most once per day (scheduled) plus on changes, so rate limits are rarely hit
- If hit, wait for rate limit reset or use `workflow_dispatch` later

### Mirror Partial Failure (One Tag Fails)

**Symptom**: Some tags mirror successfully but others fail.

**Cause**: The mirror step iterates over tags and runs `crane copy` for each. A transient network error or Docker Hub API issue can affect individual tags.

**Verification**: Check the `dockerhub_mirror` step logs for which specific tag failed.

**Resolution**: Re-run the workflow manually:
```bash
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true
```

## Module Version Detection Issues

### Addon Shows "unknown" Version

**Symptom**: In the build summary, an addon shows version `unknown`:
```
- github.com/some/owner/repo: `unknown` ([repo](https://github.com/some/owner/repo))
```

**Cause**: The `fetch_addon_version` function in the `decide` step tries multiple strategies:
1. **Stable releases**: Queries `/repos/{owner}/{repo}/releases` for non-prerelease releases
2. **Tags**: Falls back to `/repos/{owner}/{repo}/tags` (first tag)
3. **Commit SHA**: Falls back to latest commit on default branch (short SHA, 7 chars)
4. **Unknown**: If repo doesn't exist or API fails

"unknown" means all strategies failed — typically the repo doesn't exist, is private, or GitHub API returned an error.

**Debugging**:
```bash
# Test GitHub API access for the addon repo
curl -H "Authorization: Bearer <GITHUB_TOKEN>" \
  "https://api.github.com/repos/owner/repo/releases?per_page=10"

# Check if repo exists and is public
curl -H "Authorization: Bearer <GITHUB_TOKEN>" \
  "https://api.github.com/repos/owner/repo"
```

**Resolution**:
- Verify the addon path in `Dockerfile-cloudflare` is correct (e.g., `github.com/owner/repo` not `github.com/owner/repo@v1.2.3`)
- Ensure the repo is public (GitHub API requires auth for private repos; the workflow uses `GITHUB_TOKEN` which only has access to this repository)
- If the addon has no releases/tags, the commit SHA fallback should work — check the default branch name

### Addon Version Shows Commit SHA Instead of Release Tag

**Symptom**: Addon version appears as a short commit hash (e.g., `a1b2c3d`) instead of a semantic version.

**Cause**: The repository has no releases and no tags. The fallback logic fetches the latest commit SHA from the default branch and uses the first 7 characters.

**Behavior**: This is expected and handled correctly:
- The version label stores the short SHA
- The release URL points to the commit page (`https://github.com/owner/repo/commit/a1b2c3d`)
- Change detection still works: if the commit SHA changes, a rebuild triggers

**No action needed** — this is the designed fallback behavior.

### Caddy Base Version Shows "latest" Instead of Semver

**Symptom**: `caddy_tag` output is `latest` instead of a version like `2.8.4`.

**Cause**: The `decide` step extracts the version from the Caddy base image's `org.opencontainers.image.version` label. If that label is missing, malformed, or doesn't match semver pattern, it falls back to parsing the image history. If that also fails, it defaults to `latest`.

**Impact**:
- Tags published: `latest` and `caddy-latest` (no `caddy-v...` alias)
- Change detection still works via base image digest comparison

**Debugging**:
```bash
# Check Caddy base image labels
crane config caddy:latest | jq '.config.Labels'

# Check image history for version info
crane config caddy:latest | jq '.history[0].created_by'
```

**Resolution**: Usually temporary — upstream Caddy image may have a labeling issue. The digest-based change detection still functions correctly.

## Change Detection Not Working

### Build Doesn't Trigger Despite Upstream Changes

**Symptom**: You know an addon released a new version, but scheduled run doesn't build.

**Cause**: The `decide` step compares the **currently published image's OCI labels** against **freshly fetched upstream versions**. If the published image is stale (e.g., manual delete of GHCR image) or labels are missing, comparison may fail.

**Verification**:
```bash
# Check published image labels
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels'

# Check current upstream versions
# For each addon in Dockerfile:
curl -H "Authorization: Bearer <GITHUB_TOKEN>" \
  "https://api.github.com/repos/caddy-dns/cloudflare/releases?per_page=1"
```

**Resolution**: Force a build to refresh the baseline:
```bash
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true
```

### False Positive: Build Triggers Without Real Changes

**Symptom**: Scheduled run builds but no actual upstream changes occurred.

**Cause**: Possible reasons:
- Caddy base image was re-published with same version but different digest (rebuild)
- Addon repo's default branch received a commit (SHA changed)
- GitHub API returned different ordering for tags/releases

**Verification**: Check the build summary's "Publish reason" section — it shows exactly what changed (digest comparison, version comparison).

**Resolution**: Usually harmless. The rebuild ensures the image matches current upstream state.

## Debugging Commands Reference

### GitHub Actions Run Inspection
```bash
# List recent workflow runs
gh run list --workflow=build_cloudflare-modules.yaml --limit=5

# View full logs for a run
gh run view <run-id> --log

# View only failed steps
gh run view <run-id> --log-failed

# Re-run a failed workflow
gh run rerun <run-id>

# Trigger manual build with force
gh workflow run build_cloudflare-modules.yaml --ref main -f force=true
```

### Image Inspection with Crane
```bash
# View image config and labels (GHCR)
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels'

# View image config (Docker Hub mirror)
crane config smoochy84/caddy-cloudflare-modules:latest | jq '.config.Labels'

# View multi-arch manifest
crane manifest ghcr.io/smoochy/caddy-cloudflare-modules:latest

# Get image digest
crane digest ghcr.io/smoochy/caddy-cloudflare-modules:latest

# Get base image digest
crane digest caddy:latest

# Copy image between registries (manual mirror)
crane copy ghcr.io/smoochy/caddy-cloudflare-modules:latest \
         smoochy84/caddy-cloudflare-modules:latest
```

### GitHub API Debugging
```bash
# Test token access
curl -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  https://api.github.com/user

# Check addon repo releases
curl -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  "https://api.github.com/repos/caddy-dns/cloudflare/releases?per_page=5"

# Check addon repo tags
curl -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  "https://api.github.com/repos/caddy-dns/cloudflare/tags"

# Check Caddy releases
curl -H "Authorization: Bearer ${GITHUB_TOKEN}" \
  "https://api.github.com/repos/caddyserver/caddy/releases/tags/v2.8.4"
```

### Local Dockerfile Validation
```bash
# Test xcaddy build locally (requires Go toolchain)
xcaddy build \
  --with github.com/caddy-dns/cloudflare \
  --with github.com/WeidiDeng/caddy-cloudflare-ip \
  --with github.com/fvbommel/caddy-combine-ip-ranges \
  --with github.com/lucaslorentz/caddy-docker-proxy/v2

# Build Docker image locally
docker build -f Dockerfile-cloudflare -t caddy-test .
```

## Common Error Patterns Quick Reference

| Error Message | Likely Cause | Quick Fix |
|---------------|--------------|-----------|
| `Tag set assertion failed` | Caddy version not semver | Check base image labels; force rebuild |
| `dockerhub_enabled=false` | Missing secrets | Add `DOCKERHUB_USERNAME` + `DOCKERHUB_TOKEN` |
| `crane copy: denied` | Docker Hub auth/rate limit | Verify token; wait for rate limit reset |
| `addon version: unknown` | Repo private/no releases | Verify addon path; ensure public repo |
| `No upstream changes detected` | Nothing actually changed | Expected; use `force=true` to rebuild |
| `manifest unknown` | Image doesn't exist | First build or image deleted; force rebuild |

## Escalation

If troubleshooting doesn't resolve the issue:
1. Check the workflow run logs for the exact error
2. Run the debugging commands above to isolate the failure point
3. Open an issue in the repository with the run ID and relevant log excerpts
