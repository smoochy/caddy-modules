---
type: Integration
title: Cloudflare Modules Integration
description: "Documentation of the four curated Caddy modules in the Cloudflare image variant: DNS provider, IP detection, IP range combining, and Docker proxy."
tags: [integration, caddy, cloudflare, dns, modules]
openwiki:
  roles: [integration, domain]
  change_kinds: [integration, configuration]
  source_paths: [Dockerfile-cloudflare, .github/workflows/build_cloudflare-modules.yaml]
  symbols: [caddy-dns/cloudflare, caddy-cloudflare-ip, caddy-combine-ip-ranges, caddy-docker-proxy]
  test_paths: []
  invariants: [All four modules compiled into single Caddy binary, Modules provide DNS-01 challenge, real client IP, trusted proxy ranges, and Docker label config]
  validation_commands: [docker run --rm ghcr.io/smoochy/caddy-cloudflare-modules:latest list-modules | grep -E "cloudflare|docker-proxy"]
verified:
  - by: openwiki/0.9.0
    at: 2026-11-26T14:51:28.771Z
sources:
  - id: openwiki-source-3cea697c8dad20efcc356a29
    resource: repo://.github/workflows/build_cloudflare-modules.yaml
  - id: openwiki-source-42ef8192a0f869c58499e372
    resource: repo://Dockerfile-cloudflare
generated: { by: "openwiki/0.9.0", at: "2026-11-26T14:51:28.771Z" }
---

# Cloudflare Modules Integration

The `caddy-cloudflare-modules` image includes **four Caddy modules** compiled at build time via `xcaddy`:

| Module | Repository | Purpose |
|--------|------------|---------|
| **Cloudflare DNS** | `github.com/caddy-dns/cloudflare` | DNS-01 ACME challenge provider for Cloudflare |
| **Cloudflare IP** | `github.com/WeidiDeng/caddy-cloudflare-ip` | Extracts real client IP behind Cloudflare proxy |
| **Combine IP Ranges** | `github.com/fvbommel/caddy-combine-ip-ranges` | Combines multiple IP range sources for trusted proxies |
| **Docker Proxy** | `github.com/lucaslorentz/caddy-docker-proxy/v2` | Automatic Caddy config via Docker labels |

> **Note on Go version suffixes**: The Docker Proxy module uses the `/v2` suffix in its import path (`github.com/lucaslorentz/caddy-docker-proxy/v2`). This is a Go module convention for major versions v2+. The workflow automatically strips this suffix when querying GitHub for releases (e.g., it queries `github.com/lucaslorentz/caddy-docker-proxy`).

All four modules are **compiled into a single Caddy binary** at build time—there are no runtime plugin loads. The workflow (`build_cloudflare-modules.yaml`) parses the `--with` directives from `Dockerfile-cloudflare` and dynamically discovers module versions from GitHub at build time.

## Module Details

### 1. Cloudflare DNS (`caddy-dns/cloudflare`)

**Purpose**: Automate TLS certificate issuance/renewal via Cloudflare DNS-01 challenge.

**Configuration** (Caddyfile):
```caddyfile
{
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
}

example.com {
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
}
```

**Required**: `CLOUDFLARE_API_TOKEN` environment variable with Zone:DNS:Edit permissions.

**Documentation**: https://github.com/caddy-dns/cloudflare

---

### 2. Cloudflare IP (`WeidiDeng/caddy-cloudflare-ip`)

**Purpose**: Replaces `X-Forwarded-For` with real client IP when behind Cloudflare proxy.

**How it works**:
- Reads `CF-Connecting-IP` header (set by Cloudflare)
- Validates request comes from Cloudflare IP ranges
- Replaces `X-Forwarded-For` with real IP for downstream apps

**Configuration** (Caddyfile):
```caddyfile
{
  # Optional: customize header names
  cloudflare_ip {
    header CF-Connecting-IP
    forwarded_header X-Forwarded-For
  }
}
```

**Documentation**: https://github.com/WeidiDeng/caddy-cloudflare-ip

---

### 3. Combine IP Ranges (`fvbommel/caddy-combine-ip-ranges`)

**Purpose**: Aggregates multiple IP range sources into a single trusted proxy configuration.

**Use case**: When behind Cloudflare **and** other proxies (load balancers, Kubernetes ingress), combine all trusted IP ranges.

**Configuration** (Caddyfile):
```caddyfile
{
  combine_ip_ranges {
    # Cloudflare IPv4 + IPv6 (auto-fetched)
    cloudflare
    
    # Additional custom ranges
    10.0.0.0/8
    172.16.0.0/12
  }
}
```

**Documentation**: https://github.com/fvbommel/caddy-combine-ip-ranges

---

### 4. Docker Proxy (`lucaslorentz/caddy-docker-proxy/v2`)

**Purpose**: Automatic Caddy configuration from Docker container labels.

**How it works**: Reads labels on Docker containers, generates Caddy config dynamically. The module is enabled by its presence in the binary; no global config block is required.

**Example labels** on a service container:
```yaml
labels:
  caddy: "example.com"
  caddy.reverse_proxy: "{{upstreams 80}}"
  caddy.tls: "internal"  # or use cloudflare DNS challenge
```

**Documentation**: https://github.com/lucaslorentz/caddy-docker-proxy

---

## Combined Usage Example

```caddyfile
{
  # ACME DNS challenge via Cloudflare
  acme_dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  
  # Real client IP behind Cloudflare
  cloudflare_ip
  
  # Combine Cloudflare + custom trusted ranges
  combine_ip_ranges {
    cloudflare
    10.0.0.0/8
  }
  
  # Docker label-based config (enabled by module presence)
  # No global docker_proxy block needed
}

# Site using Cloudflare DNS for TLS
example.com {
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
  
  # Reverse proxy to Docker service (via docker-proxy labels)
  reverse_proxy service-name:80
}

# API with real client IP logging
api.example.com {
  tls {
    dns cloudflare {env.CLOUDFLARE_API_TOKEN}
  }
  
  log {
    format json
    # Real client IP available via cloudflare_ip module
  }
  
  reverse_proxy api-service:8080
}
```

## Environment Variables

| Variable | Required | Purpose |
|----------|----------|---------|
| `CLOUDFLARE_API_TOKEN` | Yes (for DNS) | Cloudflare API token with Zone:DNS:Edit |
| `CLOUDFLARE_ACCOUNT_ID` | No | For account-level operations |

## Build & Version Tracking

The build workflow **fetches latest versions at build time** from GitHub API:

| Module | Version Source |
|--------|----------------|
| Cloudflare DNS | `github.com/caddy-dns/cloudflare` releases |
| Cloudflare IP | `github.com/WeidiDeng/caddy-cloudflare-ip` releases |
| Combine IP Ranges | `github.com/fvbommel/caddy-combine-ip-ranges` releases |
| Docker Proxy | `github.com/lucaslorentz/caddy-docker-proxy` releases (strips `/v2` suffix) |

**Version recording**:
- Job summary (GitHub Actions) includes a table of all addons with versions and release links
- OCI labels: `org.opencontainers.image.addon.N.name` and `org.opencontainers.image.addon.N.version` for each module
- Image tags: `caddy-<caddy_version>` (e.g., `caddy-2.8.4`), plus `latest` and `caddy-v2.8.4`

**Upstream change detection**: The workflow runs daily (scheduled) and on pushes to `Dockerfile-cloudflare`. It compares:
- Base image digest (`caddy:latest`)
- Each module's latest release tag/commit against the version recorded in the published image's OCI labels
- If any upstream change is detected, a new build is triggered automatically

**Adding/removing modules**:
- **Add**: Edit `Dockerfile-cloudflare`, add a `--with` line with the module import path
- **Remove**: Comment out or delete the `--with` line
- **No workflow changes needed** — the workflow dynamically parses `--with` directives and discovers versions

## Image Metadata (OCI Labels)

The published image includes the following labels:

| Label | Description |
|-------|-------------|
| `org.opencontainers.image.base.name` | Base image name (`caddy:latest`) |
| `org.opencontainers.image.base.tag` | Base image tag (e.g., `2.8.4`) |
| `org.opencontainers.image.base.digest` | Base image SHA256 digest |
| `org.opencontainers.image.base.version` | Base image version |
| `org.opencontainers.image.cloudflare.version` | Cloudflare DNS module version (legacy, kept for compat) |
| `org.opencontainers.image.addon.N.name` | Module import path (e.g., `github.com/caddy-dns/cloudflare`) |
| `org.opencontainers.image.addon.N.version` | Module version (tag/commit) |

## Troubleshooting

### Module Not Loading
```bash
# Verify module compiled into binary
docker run --rm ghcr.io/smoochy/caddy-cloudflare-modules:latest list-modules | grep cloudflare
```

### Check Published Image Labels
```bash
# View all OCI labels including addon versions
crane config ghcr.io/smoochy/caddy-cloudflare-modules:latest | jq '.config.Labels'
```

### Force Rebuild
Trigger the workflow manually with the `force: true` input, or push a change to `Dockerfile-cloudflare`.
