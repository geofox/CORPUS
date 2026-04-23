---
name: Project CORPUS overview
description: CORPUS is BIPT's complaint management platform; foundations spec at v3 (critic + missing-components reviews folded in)
type: project
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
**CORPUS** = Complaint Office Routing & Processing Unified System. BIPT's platform for handling complaints across EU regulations (DSA, AI Act, Data Act, future) plus BIPT's pre-existing telecom-consumer and postal mandates.

**As of 2026-04-23:**
- Repo at `github.com:geofox/CORPUS`, branch `main`, pushed through commit `71a460c`.
- Only content: design spec at `docs/superpowers/specs/2026-04-17-corpus-foundations-design.md` (**1443 lines**, 128 numbered decisions, 35 open questions).
- Spec has been through 3 amendment rounds:
  1. Initial design (brainstorm → spec)
  2. First three-critic review (architecture / security / pragmatism) — 19 decisions added (§7 #46-64)
  3. Second three-agent review (operational practice / templating / process catalog) — 18 decisions added (§7 #65-82)
- Awaiting implementation planning (writing-plans skill) — not yet invoked.

**Architecture — platform-first (Option B):**
- Python 3.12 + FastAPI + SQLAlchemy 2 + Postgres 16 + React/Vite + shadcn/ui + Tailwind.
- Custom YAML workflow DSL with CEL expressions; primitives: start/task/parallel/sub_process/end states, event+scheduled triggers, SLAs-as-observers, `ai_prompts` block.
- Docker Compose with own Traefik for local dev; dedicated Tailscale sidecar (`corpus-dev`) for remote access (never reuses prod infra). Azure Container Apps in Belgium Central for later deploy.
- Append-only event log with `schema_version`; per-case namespaced Postgres advisory lock; frozen process definition snapshots on ProcessInstance.
- Multiple concurrent ProcessInstances per case (main + spawned + triggered).

**v0 domain — rich:**
- Case (with priority, admissibility, closure_reason, escalation, ai_disabled)
- Party (with trusted-flagger, ai_opt_out, is_anonymized)
- User (with OOO + delegate)
- CaseLink / Inquiry / CaseRecusal (operational practice entities)
- Event (append-only, schema_version, via_ai_agent_id, request_ip/user_agent)
- Attachment (SHA-256, soft-delete, is_internal, category with is_shareable_default)
- ProcessInstance (with definition snapshot)
- Regulator-grade templating: TemplateSet + TemplateVersion (approval state machine + multi-language staleness tracking) + TemplateFragment + LetterheadProfile
- Decision + ConfidentialityClaim + FormalDocument (shapes only in v0; workflows in §6.13)
- AIAgent + AIInteraction (dummy-only in v0; real providers in §6.4)

**v0 AI Companion (A3):** humans invoke AI on demand, AI returns text, humans act; no autonomous AI action; LLMClient only ships `dummy`; real providers (OpenAI/Anthropic/Azure OpenAI) gated behind DPO sign-off in §6.4. Complainant opt-out at Party level propagates to Case.ai_disabled; blocks all AI invocations; emits ai.invocation_blocked for audit.

**Deferred / future sub-projects (18 entries in §6), ordered:**
§6.1 Timers & SLAs (next after Foundations — EU-mandated deadlines) → §6.2 Email I/O (includes PDF letterhead pipeline) → §6.3 Public Intake Form → §6.4 AI as workflow actor → §6.5 External Forwarding & Share Bundles → §6.6 Real Auth (Entra ID) → §6.7 Dashboards & Council Reporting → §6.8 Workflow Designer UI → §6.9 Azure Deployment → §6.10 Anonymisation Script → §6.11 Virus Scanning → §6.12 RBAC refinement → §6.13 Formal Enforcement (Decisions/appeals/publication) → §6.14 In-app Collaboration (notifications/@mentions) → §6.15 Operational Search (FTS) → §6.16 Bulk Ops & Data Migration → §6.17 Webhooks → §6.18 Additional items.

**Key platform principles to remember across sessions:**
- Human is always the actor for state-changing events; AI is the tool (`via_ai_agent_id`).
- Complainant consent-first for AI processing (opt-out honoured case-wide).
- Evidence integrity: SHA-256 re-verify on download; system-actor events carry `build_sha`; append-only everything.
- Every CORPUS-owned infra instance is isolated from prod (Traefik, Tailscale sidecar, etc.).
- Local-first development on Docker Compose; Azure is its own later sub-project.
- Specs always carry deferred sub-projects + decisions log (with why) + open questions (with defaults).

**How to apply:** Treat the spec as authoritative. When adding features or opening sub-projects, read §6 deferred entries first (many hooks already in place). When revisiting an architectural choice, read §7 (128 entries) for the why. When unclear on a domain term, check §9 glossary.
