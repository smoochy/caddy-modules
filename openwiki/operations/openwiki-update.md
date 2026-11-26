---
type: Operations
title: OpenWiki Update Workflow
description: Bi-weekly scheduled GitHub Actions workflow that regenerates the OpenWiki documentation for this repository using the OpenWiki CLI with model fallback.
tags: [operations, openwiki, documentation, github-actions, scheduled]
openwiki:
  roles: [operations, delivery]
  change_kinds: [lifecycle, operations]
  source_paths: [.github/workflows/openwiki-update.yaml]
  symbols: [openwiki, openrouter, parity-gate]
  test_paths: []
  invariants: [Runs on even ISO weeks only (bi-weekly), Manual dispatch always runs, Falls back through candidate models on failure]
  validation_commands: [gh workflow run openwiki-update.yaml --ref main]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-e082dc1d961398caa2e27f47
    resource: repo://.github/workflows/openwiki-update.yaml
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
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
        echo "run=true" >> "$GITHUB_OUTPUT"
        exit 0
      fi
      week=$(( 10#$(date -u +%V) % 2 ))
      if [ "$week" -eq 0 ]; then
        echo "run=true" >> "$GITHUB_OUTPUT"
      else
        echo "run=false" >> "$GITHUB_OUTPUT"
        echo "Odd ISO week; skipping this run."
      fi
```

- **Manual dispatch**: Always runs (`run=true`)
- **Scheduled**: Runs only on even ISO weeks (`week % 2 == 0`)
- The `10#` prefix forces base-10 interpretation to avoid octal parsing issues with week numbers 08 and 09
- Outputs `run` boolean for downstream job

### Job 2: `update` (Documentation Generation)

```yaml
needs: gate
if: needs.gate.outputs.run == 'true'
runs-on: ubuntu-latest
steps:
  - Checkout (with persist-credentials: true)
  - Setup Node.js 26
  - Install OpenWiki globally: `npm install --global openwiki`
  - Pick candidate models from openrouter-model-list
  - Run OpenWiki with model fallback
  - Create pull request with changes
```

#### Model Selection & Fallback

1. Fetches `models-openwiki.json` from `smoochy/openrouter-model-list` (score-sorted)
2. Filters to `sanity_ok` models (smoke-tested via OpenRouter)
3. Iterates through candidates until one succeeds:

```bash
for model in $OPENWIKI_MODEL_IDS; do
  echo "::group::OpenWiki with $model"
  if OPENWIKI_MODEL_ID="$model" openwiki code --update --print; then
    echo "::endgroup::"
    exit 0
  fi
  echo "::endgroup::"
  echo "Model $model failed; trying next candidate."
done
echo "All candidate models failed." >&2
exit 1
```

- `sanity_ok` only proves a smoke call succeeded, not that the model survives a full run, so multiple candidates are needed
- Free models regularly return empty responses under load; the loop falls through instead of failing the whole run

#### OpenWiki Invocation

```bash
openwiki code --update --print
```

The `--update` flag also handles the first run for code docs when environment variables are set. The `--print` flag outputs generated content to stdout.

**Environment variables:**

| Variable | Source | Purpose |
|----------|--------|---------|
| `OPENWIKI_MODEL_ID` | Set per iteration in fallback loop | Selected model ID |
| `OPENWIKI_PROVIDER=openrouter` | Hardcoded | Provider identifier |
| `OPENROUTER_API_KEY` | Repository secret | OpenRouter authentication |
| `OPENWIKI_TELEMETRY_DISABLED=1` | Hardcoded | Disable telemetry |
| `OPENWIKI_PROVIDER_RETRY_ATTEMPTS=10` | Hardcoded | Retry through 429s (free tier allows 16 req/min) |
| `LANGSMITH_ENDPOINT=https://eu.api.smith.langchain.com` | Hardcoded | EU region tracer endpoint (avoids US 403 retries) |
| `LANGSMITH_API_KEY` | Repository secret | LangSmith authentication |
| `LANGCHAIN_PROJECT=openwiki` | Hardcoded | Project name for tracing |
| `LANGCHAIN_TRACING_V2=true` | Hardcoded | Enable LangChain v2 tracing |

#### Pull Request Creation

Uses `peter-evans/create-pull-request@v8` to create a PR with the updated `openwiki/` directory. The checkout step uses `persist-credentials: true` so the PR action can push changes.

## Permissions

The workflow requires:
- `contents: write` — to push the generated documentation
- `pull-requests: write` — to create the update PR

## Manual Trigger

Run on demand via GitHub CLI:

```bash
gh workflow run openwiki-update.yaml --ref main
```

Or from the Actions tab in the GitHub UI.
