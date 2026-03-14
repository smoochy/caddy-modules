# caddy-cloudflare-modules

[![Build](https://github.com/smoochy/caddy-modules/actions/workflows/build_cloudflare-modules.yaml/badge.svg)](https://github.com/smoochy/caddy-modules/actions/workflows/build_cloudflare-modules.yaml)
[![Docker Pulls](https://img.shields.io/docker/pulls/smoochy84/caddy-cloudflare-modules?logo=docker)](https://hub.docker.com/r/smoochy84/caddy-cloudflare-modules)
[![Docker Stars](https://img.shields.io/docker/stars/smoochy84/caddy-cloudflare-modules?logo=docker)](https://hub.docker.com/r/smoochy84/caddy-cloudflare-modules)

[![Buy me uptime](https://img.shields.io/badge/Buy%20me%20uptime%20%F0%9F%96%A5%EF%B8%8F-smoochy84-E9C46A?logo=buymeacoffee&logoColor=000000)](https://www.buymeacoffee.com/smoochy84)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-smoochy-7CC6FE?logo=ko-fi&logoColor=000000)](https://ko-fi.com/smoochy)

Custom Caddy image with curated Cloudflare and reverse-proxy modules, rebuilt
automatically when upstream components change.

## Quick pull

```bash
docker pull smoochy84/caddy-cloudflare-modules:latest
```

## What you get

- Cloudflare DNS support for ACME DNS-01 challenges
- Real client IP handling when Caddy sits behind Cloudflare
- Combined trusted proxy IP ranges for cleaner proxy configuration
- Docker label-driven config via `caddy-docker-proxy`
- Multi-arch images for `linux/amd64`, `linux/arm64`, and `linux/arm/v7`

## Tags

- `latest`
- `caddy-<x.y.z>`

## Why this image

- Avoid maintaining your own `xcaddy` build pipeline
- Stay current with upstream Caddy and module changes
- Use reproducible tags for pinned deployments
- Keep image metadata traceable back to upstream versions

## Full documentation

See the GitHub repository for usage examples, addon details, and release
behavior:

<https://github.com/smoochy/caddy-modules>

## Support

If this image saves you time or helps your setup, you can support ongoing
maintenance for a project I maintain in my spare time via Ko-fi or Buy Me a
Coffee.
