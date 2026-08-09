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
---

# Cloudflare Modules Integration

The `caddy-cloudflare-modules` image includes **four Caddy modules** compiled at build time via `xcaddy`:

| Module | Repository | Purpose |
|--------|------------|---------|
| **Cloudflare DNS** | `github.com/caddy-dns/cloudflare` | DNS-01 ACME challenge provider for Cloudflare |
| **Cloudflare IP** | `github.com/WeidiDeng/caddy-cloudflare-ip` | Extracts real client IP behind Cloudflare proxy |
| **Combine IP Ranges** | `github.com/fvbommel/caddy-combine-ip-ranges` | Combines multiple IP range sources for trusted proxies |
| **Docker Proxy** | `github.com/lucaslorentz/caddy-docker-proxy/v2` | Automatic Caddy config via Docker labels |

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

**How it works**: Reads labels on Docker containers, generates Caddy config dynamically.

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
  
  # Docker label-based config (optional)
  # docker_proxy is enabled by module presence
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

## Module Version Tracking

The build workflow **fetches latest versions at build time** from GitHub API:

| Module | Version Source |
|--------|----------------|
| Cloudflare DNS | `github.com/caddy-dns/cloudflare` releases |
| Cloudflare IP | `github.com/WeidiDeng/caddy-cloudflare-ip` releases |
| Combine IP Ranges | `github.com/fvbommel/caddy-combine-ip-ranges` releases |
| Docker Proxy | `github.com/lucaslorentz/caddy-docker-proxy/v2` releases |

Versions recorded in:
- Job summary (GitHub Actions)
- OCI labels: `org.opencontainers.image.addon.N.version`
- Image tag: `caddy-<caddy_version>` (Caddy version only)

## Adding/Removing Modules

**Add**: Edit `Dockerfile-cloudflare`, add `--with` line (workflow auto-discovers)

**Remove**: Comment out or delete `--with` line

**No workflow changes needed** - dynamic discovery handles it.

## Troubleshooting

### Module Not Loading
```bash
# Verify module compiled into binary
docker run --rm ghcr.io/smoochy/caddy-cloudflare-modules:latest list-modules | grep cloudflare
```

### DNS Challenge Fails
- Verify `CLOUDFLARE_API_TOKEN` has correct permissions (Zone:DNS:Edit)
- Check domain is in Cloudflare account
- Verify token not expired

### Real IP Not Working
- Ensure requests actually come through Cloudflare (orange-clouded)
- Check `CF-Connecting-IP` header present
- Verify `cloudflare_ip` directive in Caddyfile

### Docker Proxy Not Discovering
- Container must have `caddy` label
- Container must be on same Docker network
- Check `caddy-docker-proxy` logs: `docker logs caddy`