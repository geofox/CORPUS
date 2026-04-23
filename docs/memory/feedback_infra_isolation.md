---
name: Isolate CORPUS from existing production infra
description: When CORPUS needs infrastructure already present on the host (Traefik, Tailscale, etc.), spin up a dedicated instance rather than reuse; prod must remain untouched
type: feedback
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
When CORPUS needs a piece of infrastructure that already exists on the host (reverse proxy, VPN, shared daemon), **always prefer spinning up a dedicated CORPUS-owned instance** rather than reusing the production one.

**Why:** Geoffrey has production services running on this machine (srv-traefik, srv-tailscale-1, others). After we initially proposed reusing the existing Traefik for CORPUS, he course-corrected: "wrong decision to reuse traefik as i've some service in prod." The same reflex applies to other shared infra (Tailscale daemon on host, existing Tailscale container, any shared config). A CORPUS-scoped misconfig could break prod; a dedicated CORPUS-scoped instance can be stopped/rebuilt freely.

**How to apply:**
- CORPUS Compose always ships its own Traefik, its own Tailscale sidecar (identity `corpus-dev`), etc. — never a `network_mode: "container:<prod-container-name>"` or any other shared coupling.
- Isolation is worth small extra config (e.g., separate pre-auth key, separate ACME config) to keep prod untouched.
- When we need to decide between "reuse X" and "ship our own X", default to "ship our own X" and justify reuse if ever chosen.
- Applies beyond Traefik/Tailscale: shared Postgres, shared Redis, shared Docker networks — all get CORPUS-owned equivalents unless there's a strong reason to share.
