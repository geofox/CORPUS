---
name: Prefer Traefik over nginx
description: For CORPUS edge proxy in dev and prod-like environments, use Traefik; never propose nginx as the default
type: feedback
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
For edge reverse proxy in CORPUS, **use Traefik**, not nginx.

**Why:** Geoffrey corrected an initial nginx proposal. His reasoning (inferred + confirmed): Traefik's label-based service discovery is a much better fit for Docker Compose development — services declare their own routes, routing updates automatically when services come and go, dashboard is available for debugging, TLS via Let's Encrypt is trivial when we want it.

**How to apply:**
- Traefik is the default edge proxy in docker-compose.yml.
- For the prod frontend, serve built SPA via `static-web-server` (Rust, tiny image, SPA fallback) — not an nginx container.
- If nginx is *technically* needed later (e.g., for something Traefik doesn't handle), surface it as a tradeoff and propose alternatives first.
