---
type: Operations
title: OpenWiki Update Workflow
description: Bi-weekly scheduled GitHub Actions workflow that regenerates the OpenWiki documentation for this repository using the OpenWiki CLI.
tags: [operations, openwiki, documentation, github-actions, scheduled]
openwiki:
  roles: [operations, delivery]
  change_kinds: [lifecycle, operations]
  source_paths: [.github/workflows/openwiki-update.yaml]
  symbols: [openwiki, openrouter, parity-gate]
  test_paths: []
  invariants: [Runs on even ISO weeks only (bi-weekly), Manual dispatch always runs, Falls back through candidate models on failure]
  validation_commands: [gh workflow run openwiki-update.yaml --ref main]
---

# OpenWiki Update Workflow

## Workflow File
`.github/workflows/openwiki-update.yaml`

## Purpose

Automatically regenerates the repository's OpenWiki documentation (`openwiki/` directory) on a **bi-weekly schedule** or manual trigger.

## Schedule

| Schedule | Cron | Condition |
|----------|------|-----------|
| **Bi-weekly** | `0 5 * * 4` (Thursdays 05:00 UTC) | **Only even ISO weeks** |
| **Manual** | `workflow_dispatch` | Always runs |

The parity gate (see below) enforces bi-weekly execution since GitHub cron cannot express "every other week".

## Workflow Structure

### Job 1: `gate` (Parity Check)

```yaml
runs-on: ubuntu-latest
outputs:
  run: ${{ steps.parity.outputs.run }}
steps:
  - name: Check ISO week parity
    id: parity
    run: |
      if [ "${{ github.event_name }}" = "workflow_dispatch" ]; then
        echo "run=true"
        exit 0
      fi
      week=$(( 10#$(date -u +%V) % 2 ))
      if [ "$week" -eq 0 ]; then
        echo "run=true"
      else
        echo "run=false"
        echo "Odd ISO week; skipping this run."
      fi
```

- **Manual dispatch**: Always runs (`run=true`)
- **Scheduled**: Runs only on even ISO weeks (`week % 2 == 0`)
- Outputs `run` boolean for downstream job

### Job 2: `update` (Documentation Generation)

```yaml
needs: gate
if: needs.gate.outputs.run == 'true'
runs-on: ubuntu-latest
steps:
  - Checkout (with persist-credentials: true)
  - Setup Node.js 24
  - Install OpenWiki globally: `npm install --global openwiki`
  - Pick candidate models from openrouter-model-list
  - Run OpenWiki with model fallback
  - Create pull request with changes
```

#### Model Selection & Fallback

1. Fetches `models-openwiki.json` from `smoochy/openrouter-model-list`
2. Filters to `sanity_ok` models (smoke-tested)
3. Iterates through candidates until one succeeds:
   ```bash
   for model in $OPENWIKI_MODEL_IDS; do
     if OPENWIKI_MODEL_ID="$model" openwiki code --update --print; then
       exit 0
     fi
   done
   ```

#### OpenWiki Invocation

```bash
openwiki code --update --print
```

Environment variables:
- `OPENWIKI_MODEL_ID` - Selected model (set per iteration)
- `OPENWIKI_PROVIDER=openrouter`
- `OPENROUTER_API_KEY` - From repository secrets
- `OPENWIKI_PROVIDER_RETRY_ATTEMPTS=10` - Handles rate limits
- `LANGSMITH_*` - Tracing configuration (EU endpoint)

#### Pull Request Creation

Uses `peter-evans/create-pull-request@v8` to create a PR with:
- Updated `openwiki/` directory
- Commit message: "Update OpenWiki documentation"
- Branch: `openwiki-update-<timestamp>`

## Why Bi-Weekly?

- Documentation doesn't need daily updates
- Reduces API costs (OpenRouter free tier: 16 req/min)
- Avoids noisy PR spam for minor changes
- Manual dispatch available for immediate updates

## Manual Trigger

```bash
# Run immediately (bypasses parity gate)
gh workflow run openwiki-update.yaml --ref main
```

## Verification

After workflow completes:
1. Check for new PR: `gh pr list --head openwiki-update-*`
2. Review changes in PR diff
3. Merge to update wiki

## Configuration

| Setting | Value | Source |
|---------|-------|--------|
| Schedule | Thursdays 05:00 UTC, even weeks | `.github/workflows/openwiki-update.yaml` |
| Models | Dynamic from `openrouter-model-list` | Fetched at runtime |
| Retry attempts | 10 | `OPENWIKI_PROVIDER_RETRY_ATTEMPTS` |
| Node version | 24 | `actions/setup-node@v7` |
| OpenWiki version | Latest (`npm install --global openwiki`) | Installed at runtime |

## Troubleshooting

### All Models Failed
- Check workflow logs for specific errors
- OpenRouter free tier may be rate limited
- Try manual dispatch later

### No Changes in PR
- OpenWiki determined no updates needed
- Check `.last-update.json` gitHead vs current HEAD

### PR Not Created
- Verify `persist-credentials: true` on checkout
- Check `peter-evans/create-pull-request` action logs

## Related

- [OpenWiki CLI Documentation](https://github.com/smoochy/openwiki)
- [openrouter-model-list](https://github.com/smoochy/openrouter-model-list)