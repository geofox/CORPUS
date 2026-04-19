# CORPUS — Foundations Design Spec

**Status:** Draft for review — ready for implementation planning
**Created:** 2026-04-17
**Last amended:** 2026-04-19 (three-critic review feedback folded in; §6 re-prioritized so Timers & SLAs is the next sub-project after Foundations)
**Scope (v0, this spec):** Foundations + Case, Party, Attachment & Event Timeline + Workflow Engine Skeleton (state machine, parallel/sub_process/triggers runtime, SLAs parsed but not fired) + Staff UI (dashboard, case detail tabs, AI Companion drawer) + AI Companion (v0 dummy-only LLM client) + remote access via Tailscale sidecar. Runs on Docker Compose locally; Azure deploy is its own later sub-project.
**Next sub-project (recommended):** §6.1 Timers & SLAs — actual timer firing, reminder automation, breach-triggered escalation. Elevated from former §6.3 position because EU-mandated SLAs are non-negotiable regulator obligations, not internal targets.
**Author:** Geoffrey Richard (with Claude)

---

## Table of contents

1. [Executive summary](#1-executive-summary)
2. [Context](#2-context)
3. [Architectural approaches considered](#3-architectural-approaches-considered)
4. [System design](#4-system-design)
   1. [Runtime shape (Compose + Traefik)](#41-runtime-shape-compose--traefik)
   2. [Domain model](#42-domain-model)
   3. [Workflow DSL](#43-workflow-dsl)
   4. [Event timeline](#44-event-timeline)
   5. [API surface](#45-api-surface)
   6. [Staff UI](#46-staff-ui)
   7. [Infra, CI/CD, local dev workflow](#47-infra-cicd-local-dev-workflow)
   8. [AI Companion](#48-ai-companion)
5. [Testing strategy](#5-testing-strategy)
6. [Deferred / future sub-projects](#6-deferred--future-sub-projects)
7. [Decisions log](#7-decisions-log)
8. [Open questions](#8-open-questions)
9. [Glossary](#9-glossary)

---

## 1. Executive summary

**CORPUS** — Complaint Office Routing & Processing Unified System — is BIPT's platform for managing complaints across multiple EU regulatory mandates (DSA, AI Act, Data Act, and future regulations).

CORPUS is a **configurable workflow platform**, not a hardcoded complaint CRM. The Complaint Team authors processes (initially as reviewed YAML in git, later via a no-code designer); the engine runs them; every state change is captured in an append-only event timeline. AI-assisted triage, multilingual correspondence, bidirectional email, external forwarding with share bundles, and dashboards for oversight are all first-class features planned across the roadmap.

**This first sub-project (the subject of this spec) establishes the Foundations:** domain model (including AI companion entities and complainant AI-opt-out), append-only event timeline with `schema_version` and evidence-integrity guarantees (SHA-256 re-verify on download, system-actor attribution, request-IP logging), attachment handling with soft-delete and share-bundle exclusion flags, the workflow engine skeleton (start/task/parallel/sub_process/end state types + triggers; `gateway`/`wait`/`spawn_and_continue`/`debounce` reserved; SLAs declared but not fired), the **AI Companion** (drawer UI, workflow-scoped prompts, opt-out enforcement — v0 ships only the `dummy` LLM client, real providers land in §6.4 behind a DPO gate), and a staff UI (tabbed case detail, Complaint-Team dashboard with 5 widgets, escalate + send-reminder actions). Runs locally on Docker Compose with an isolated Tailscale sidecar for remote dev access.

**Next sub-project after Foundations: §6.1 Timers & SLAs** — the regulatory deadline loop (external EU-set SLAs, automatic reminders, breach-triggered escalations). Elevated to first position in the roadmap because SLA compliance is a core regulator obligation.

**Explicit non-goals for v0:** public intake form, email I/O, AI-as-workflow-actor (proposals + review flow), real LLM provider clients (OpenAI/Anthropic/Azure OpenAI — behind DPO gate in §6.4), timer/SLA firing, external share-bundle dispatch, workflow designer UI, real Entra ID authentication, reporting & analytics dashboards, Azure deployment. Each is its own future sub-project (see §6) with v0 hooks already in place.

**Post three-critic review (architecture / security / pragmatism):** the spec was amended on 2026-04-19 to lock in 19 additional decisions (cheap architectural hardening, security guardrails, DSL primitive restraint, scope trims, testing trims) driven by critic feedback. See decisions §7 #46–64 for the full list and rationale.

---

## 2. Context

### 2.1 BIPT's role

BIPT is the Belgian federal regulator for postal services and telecommunications. It has been designated as the competent authority for a growing set of EU digital regulations — notably the Digital Services Act (DSA), the AI Act, and work touching the Data Act / cloud services. Complaints arrive at BIPT from the public (citizens, businesses) and must be triaged, routed to internal domain teams, sometimes forwarded to regional authorities, tracked against regulatory SLAs, and reported on.

The Complaint Team operates as the **coordinator and guardian** of the full complaint lifecycle — including cases that are day-to-day being handled by DSA, AI, or Market teams. They report to the BIPT Council.

### 2.2 Current state

**Greenfield.** No existing complaint management system is being replaced. Today's process relies on email, manual forms, and ad-hoc spreadsheets. This frees us from migration constraints but also means taxonomies, routing rules, and SLAs must be modeled from scratch.

### 2.3 Users

- **Complaint Team** — triage, routing, oversight, reminders, escalations; responsible to the Council.
- **Domain teams** — Users/DSA Team, Market Team, AI Team; act on forwarded cases within their remit.
- **External authorities** (semi-internal) — Belgian regional regulators (Flanders, Wallonia, Brussels) and other federal/EU bodies; receive forwarded complaints, return responses.
- **Target companies** — entities the complaint is about; may be contacted, may reply with evidence.
- **Complainants** — public individuals and businesses; submit complaints, receive acknowledgements and responses. In v0 they don't log in; intake is via future public form / email (deferred sub-projects).

### 2.4 Constraints

| Area | Constraint |
|---|---|
| **Team / timeline** | Small team; regulatory SLAs (DSA, AI Act) create real deadlines driving delivery. |
| **Scale** | 100s of complaints per year. Microservices are over-engineering; a well-structured monolith is right-sized. |
| **Hosting** | Self-hostable preferred; Azure acceptable for managed services. Belgium Central region. |
| **Data residency** | Belgium Central Azure region when on Azure; otherwise EU-only. |
| **LLM provider** (when AI lands) | Azure OpenAI Service (EU endpoint, no-training commitment, Microsoft DPA). |
| **Languages** | Templates must support NL, FR, EN (DE later). Each Party (complainant, company, authority) has a language set on it; templates render per-recipient. Staff UI is EN-only v0; i18next wired from day one so locales add later without refactor. |
| **Auth** | Azure AD (Entra ID) eventually. v0 uses a pluggable stub. |
| **Retention** | Unlimited for now; an anonymization hook is in the data model (`Party.is_anonymized`) so a future script is trivial. Not MVP. |
| **Open-source visibility** | Not yet decided. No OSS-hostile dependencies chosen (no GPL-contaminating libs, no commercial-only engines). |
| **Accessibility** | WCAG 2.1 AA (AnySurfer target) is a Belgian federal-body obligation. |

---

## 3. Architectural approaches considered

Three approaches were evaluated; **Approach 1 was chosen**.

### 3.1 Approach 1 (CHOSEN) — Python modular monolith + custom YAML workflow DSL + React

- Backend: Python 3.12 + FastAPI + SQLAlchemy 2 + Pydantic v2 + Alembic.
- DB: Postgres 16, single instance, logical schema grouping by module family (refined during implementation).
- Workflow engine: custom, declarative YAML DSL, in-process runtime. CEL (Google's Common Expression Language) for conditions.
- Frontend: React + Vite + TypeScript + shadcn/ui + Tailwind + TanStack Query + React Hook Form + Zod.
- Runtime: Docker Compose (dev) with Traefik as edge proxy; Azure Container Apps (future deploy sub-project).
- AI (later sub-project): Azure OpenAI Python SDK.

**Rationale:** Python's AI ecosystem pays off substantively when triage lands. A custom DSL matches regulator primitives ("forward to 3 regions with 15-day SLA, auto-remind, escalate on breach") cleanly — BPMN contorts the same semantics. A monolith is correct at 100s/year. Custom DSL primitives are ~a few hundred lines and grow incrementally. All dependencies permissively licensed.

### 3.2 Approach 2 (not chosen) — Python monolith + embed Camunda 8 / SpiffWorkflow

Off-the-shelf workflow engine. Rejected because BPMN's "everything is a task with a form" abstraction fights the regulator mental model (parallel forward + per-branch SLA + auto-reminder is natural in our DSL, verbose in BPMN), and Camunda's self-hosted infra footprint is disproportionate for 100s/year. SpiffWorkflow is LGPL — acceptable but less clean if we later choose a permissive OSS release.

### 3.3 Approach 3 (not chosen) — .NET 8 + Blazor + custom DSL

Native Azure fit. Rejected because the AI ecosystem (triage roadmap) is materially richer in Python, and the team gains nothing specific from C# end-to-end for this problem shape.

---

## 4. System design

### 4.1 Runtime shape (Compose + Traefik)

CORPUS runs as a small set of Compose services on a Linux host. Azure mapping is deferred to its own sub-project.

```
            ┌────────────────────────────────────────────┐
            │         Host (Linux dev machine)           │
            │                                            │
            │  ┌──────────────────────────────────────┐  │
            │  │  traefik  (:80, dashboard :8080)     │  │
            │  │  routes: /api → api, / → web         │  │
            │  └──────────┬──────────────┬────────────┘  │
            │             │              │               │
            │     ┌───────▼──────┐  ┌────▼─────────────┐ │
            │     │  web         │  │  api             │ │
            │     │  (Vite dev / │  │  FastAPI         │ │
            │     │  static-srv) │  │  (Uvicorn)       │ │
            │     └──────────────┘  └────────┬─────────┘ │
            │                                │           │
            │                        ┌───────▼──────┐    │
            │                        │  db          │    │
            │                        │  postgres:16 │    │
            │                        └──────────────┘    │
            │                                            │
            │   Profiles: tools (adminer), e2e           │
            │   (Playwright runner), prod (overlay)      │
            └────────────────────────────────────────────┘
```

**Services (default dev):**

| Service | Image / build | Purpose |
|---|---|---|
| `traefik` | `traefik:v3` | Edge reverse proxy with Docker provider; routes by label; dashboard at `traefik.localtest.me:8080`. |
| `db` | `postgres:16-alpine` | Single DB, named volume `corpus_pgdata`. |
| `api` | built from `backend/Dockerfile` | FastAPI application. Entrypoint: wait for db → `alembic upgrade head` → seed (if empty) → Uvicorn. Background-job runtime (APScheduler or similar) arrives with the timers sub-project. |
| `web` | built from `frontend/Dockerfile` | Vite dev server (HMR) in default profile; static-web-server in `prod` profile. |
| `adminer` | `adminer` (profile: `tools`) | Browser DB inspector. |
| `e2e` | built from `frontend/Dockerfile` (profile: `e2e`) | Playwright runner. |
| `tailscale` | `tailscale/tailscale` (profile: `remote`, `demo`) | Tailscale sidecar with its own tailnet identity (`corpus-dev`). Traefik joins its netns via `network_mode: "service:tailscale"`, making CORPUS reachable at `corpus-dev.<tailnet>.ts.net`. See §4.7.1. |

**Traefik routing via Docker labels** — each service declares its own routes; no central config to maintain. Dev hostnames via `*.localtest.me` (public DNS that resolves to 127.0.0.1 — no `/etc/hosts` editing). Remote access is opt-in via Compose profiles (§4.7.1), deliberately isolated from any existing production Traefik or Tailscale on the host.

**Module boundaries inside `api`:**
`auth` (stub), `parties`, `cases`, `events`, `attachments`, `storage` (shared infra), `workflow`, `templates`, `admin`, `dashboards`, `reminders`, `escalations`, `api` (FastAPI routers), `core` (shared: db, logging, config, errors).

**Repo layout (monorepo):**

```
/docker-compose.yml              Default dev stack
/docker-compose.prod.yml         Overlay for prod-like local
/.env.example                    Config template
/backend                         FastAPI app (+ Dockerfile, Alembic, tests, scripts)
/frontend                        React app (+ Dockerfile, tests)
/processes                       YAML process definitions (git = source of truth)
/infra                           Reserved for future Azure IaC (Bicep/Terraform)
/docs                            Specs, plans, ADRs
/scripts                         Dev helpers (seed, reset, etc.)
/justfile                        Optional `just` command shortcuts
```

### 4.2 Domain model

```
                      ┌──────┐         ┌──────┐
                      │ User │◄───────►│ Team │
                      └──┬───┘         └───┬──┘
                         │                 │
       ┌─────────┐  participant   ┌────────▼────────┐
       │  Party  │◄───────────────┤      Case       │
       └─────────┘  (role + data) └───┬────────┬────┘
                                      │        │
                               ┌──────▼──┐  ┌──▼────────┐
                               │  Event  │  │ Process   │
                               │ append- │  │ Instance  │──► Task (n)
                               │ only    │  └───────────┘
                               └─────────┘        │
                                                  │ references
                                                  ▼
                                       ProcessDefinition
                                       (YAML file, not a table)

       Attachments belong to a Case; may link to a specific Event.
```

**Entities (v0):**

- **`User`** — staff member. `id`, `email` (unique citext), `display_name`, `team_id`, `is_admin`, `external_id` (nullable, OIDC sub when Entra lands), `is_active`, timestamps.
- **`Team`** — internal BIPT team. `id`, `slug` (`complaint`, `dsa`, `ai`, `market`), `display_name`, `description`, timestamps.
- **`Party`** — any entity referenced in a case. `id`, `kind` enum `individual | organization | authority`, `display_name`, `language` enum `nl | fr | en | de`, `email`, `phone`, `address` jsonb, `metadata` jsonb, `is_anonymized` bool, `anonymized_at`, `ai_opt_out` bool default false (when true on a Party playing the `complainant` role, every case they're on gets AI disabled — the data subject has refused AI processing), `ai_opt_out_recorded_at` nullable, `ai_opt_out_source` enum `intake_form | staff_manual | email_reply | other` nullable, timestamps.
- **`Case`** — the complaint aggregate. `id`, `reference` unique (`CORPUS-2026-0042`), `title`, `description`, `category_id` FK, `status` enum `draft | open | closed`, `received_at`, `closed_at`, `created_by_user_id` (nullable), `is_escalated` bool, `escalation_level` enum `team_lead | council | leadership` (nullable), `escalated_at` (nullable), `escalated_by_user_id` (nullable), `escalation_reason` (nullable), `ai_disabled` bool default false (effective flag: true if explicitly set OR any complainant Party has `ai_opt_out=true`; computed+stored so the UI check is a single column read), `ai_disabled_reason` text nullable (`complainant_opt_out | staff_decision | policy | other`), `ai_disabled_at` nullable, `ai_disabled_by_user_id` nullable FK, timestamps.
- **`CaseCategory`** — seeded taxonomy. `id`, `slug` (`dsa`, `ai-act`, `data-act`), `display_name`, `description`, timestamps.
- **`CaseParticipant`** — party's role on a case. `id`, `case_id`, `party_id`, `role` enum `complainant | target | authority | cc | other`, `added_at`, `metadata` jsonb. Unique on `(case_id, party_id, role)`.
- **`Event`** — append-only timeline. `id`, `case_id`, `sequence_number` (per-case monotonic bigint), `event_type` (dotted namespace), `schema_version` smallint NOT NULL default 1 (bumps when a payload shape changes; renderers dispatch on `(event_type, schema_version)`), `actor_type` enum `user | system | party`, `actor_user_id` (nullable), `actor_party_id` (nullable), `system_actor_ref` jsonb nullable (for `system` events: `{component, build_sha}` identifying which service/version emitted it), `request_ip` inet nullable, `user_agent` text nullable (populated on user-triggered events — read/download audit trail), `via_ai_agent_id` (nullable FK AIAgent — set when a human action was AI-assisted), `from_interaction_id` (nullable FK AIInteraction — specific invocation that produced content the user used), `payload` jsonb, `occurred_at`, `recorded_at`. **No `UPDATE`/`DELETE` allowed.** The AI fields preserve the "human is always the actor" principle (§4.8) while making AI assistance auditable.
- **`Attachment`** — evidence. `id`, `case_id`, `event_id` (nullable), `original_filename`, `content_type` (sniffed, not trusted), `size_bytes`, `checksum_sha256`, `storage_key` (opaque UUID-based), `category_id` FK (NOT NULL, defaults to `other`), `is_internal` bool (never shareable externally when true), `uploaded_by_user_id`, `uploaded_by_party_id` (for future intake/email), `description`, `is_deleted` bool (soft delete), `deleted_at`, `deleted_by_user_id`, timestamps.
- **`AttachmentCategory`** — admin-managed. `id`, `slug`, `display_name`, `description`, `is_system` (prevents deletion), `display_order`, `is_shareable_default` bool NOT NULL default true (when false, attachments of this category are excluded from future share bundles by default even if `is_internal=false` — safety rail for categories that are usually internal like `internal_note`), timestamps. Seeded: `evidence` (shareable), `correspondence` (shareable), `contract` (shareable), `screenshot` (shareable), `internal_note` (**not shareable** by default), `other` (shareable).
- **`ProcessInstance`** — one process running on a case. `id`, `case_id`, `process_id`, `process_version`, **`process_definition_snapshot` jsonb NOT NULL** (the full compiled process definition as of instance creation — the engine executes against *this* snapshot, never the live file on disk; PRs that edit `processes/*.yaml` never break running instances), `current_state`, `status` enum `running | completed | cancelled | error`, `kind` enum `main | spawned | triggered`, `parent_instance_id` (nullable, for sub-processes), `context` jsonb, `started_at`, `ended_at`, timestamps. **Partial unique index**: `(case_id) WHERE kind = 'main' AND status = 'running'` — one main per case. **Multiple concurrent instances are allowed** (main + spawned + triggered).
- **`Task`** — work item. `id`, `process_instance_id`, `case_id` (denormalized), `task_key`, `title`, `description`, `assigned_to_user_id` (nullable), `assigned_to_team_id` (nullable), `status` enum `open | claimed | completed | cancelled`, `outcome` (drives transitions), `data` jsonb (form data), `created_at`, `claimed_at`, `completed_at`, `completed_by_user_id`.
- **`Template`** — correspondence text (text only v0; email subject field reserved). `id`, `slug`, `language` enum `nl | fr | en | de`, `subject` (nullable), `body` (Jinja2 placeholders; rendering via the `jinja2` package with an autoescaped, sandboxed environment), timestamps. Unique on `(slug, language)`.
- **`AIAgent`** — a configured LLM endpoint that the AI Companion can invoke. `id`, `slug` (`summarizer`, `translator`, `analyst`…), `display_name`, `description`, `provider` enum **`dummy` (v0)** | `openai` | `anthropic` | `azure_openai` (enum values exist, but **in v0 only `dummy` is accepted at agent creation**; the other providers + their `LLMClient` implementations land with §6.4 behind a DPO-sign-off gate), `model`, `endpoint` nullable, `api_key_secret_ref` (**name** of the env var or Key Vault secret; never the key itself), `default_prompt_prefix` nullable, `region` nullable (`eu` / `non-eu` — drives the PII warning UX), `is_active`, timestamps. See §4.8.
- **`AIInteraction`** — append-only log of each AI Companion invocation. `id`, `case_id`, `user_id` (invoker), `agent_id` FK AIAgent, `prompt_slug` nullable (if a workflow-scoped or global prompt template was used), `prompt_version` (git rev of prompt at invocation), `interaction_kind` (`summarize | explain_attachment | draft_reply | chat | workflow_prompt`), `shared_input` jsonb (what was sent: included case fields, event ids, attachment ids — summarized, never raw), `input_hash` (sha256 of the assembled prompt), `prompt_rendered` text (the final prompt sent; internal-only), `output_text` text, `tokens_in`, `tokens_out`, `cost_estimate` nullable, `latency_ms`, `status` enum `ok | provider_error | rate_limited | cancelled`, `created_at`. See §4.8.

**`ProcessDefinition` is NOT a DB table.** Process YAML files in `/processes/` are the authoritative source; loaded into an in-memory `ProcessRegistry` at app start. `ProcessInstance.process_id + process_version` pins each instance to a specific YAML.

**Indexes (at creation):**

- `events(case_id, sequence_number)` — composite; powers the timeline.
- `tasks(assigned_to_user_id) WHERE status IN ('open','claimed')` — partial; "my inbox".
- `tasks(assigned_to_team_id) WHERE status = 'open'` — partial; "team queue".
- `process_instances(case_id) WHERE kind = 'main' AND status = 'running'` — partial unique.
- `cases(status, received_at DESC)` — default case list.
- `cases(is_escalated) WHERE is_escalated = true` — dashboard escalations.
- `parties(email)` — for dedup on ingest.
- `attachments(case_id) WHERE is_deleted = false` — default case attachment list.
- `ai_interactions(case_id, created_at DESC)` — case-scoped AI history, recent-first.
- `ai_interactions(user_id, created_at DESC)` — per-user AI usage rollup for cost awareness.

**Seed data on first boot (idempotent):** Teams (`complaint`, `dsa`, `ai`, `market`), case categories (`dsa`, `ai-act`, `data-act`), attachment categories with `is_shareable_default` flags (see §4.2), one dev user per team + one admin, Belgian region authorities (Flanders, Wallonia, Brussels), bare-bones NL/FR/EN templates for a minimal DSA flow, one demo case with a small timeline, **one `AIAgent` (`summarizer`) using the `dummy` provider** so the AI Companion is usable out of the box. Real-provider keys (OpenAI/Anthropic/Azure OpenAI) come back in §6.4.

### 4.3 Workflow DSL

**Design goals:** express regulator processes the way staff describe them; YAML-in-git for auditability; deterministic; small enough to own; designer-UI-friendly for later.

**Top-level file shape:**

```yaml
id: dsa
version: "1.0.0"
title: "DSA complaint"
description: "..."
category: dsa                          # links to CaseCategory.slug
initial_state: received
context_schema:
  - name: regions_pending
    type: array<string>
  - name: summary
    type: string
triggers:                              # optional — event-spawned processes
  - event_type: "external.info_received"
    when: "event.payload.from_role == 'complainant'"
    debounce: 10m
slas:                                  # optional — deadlines as observers
  - name: final_response
    duration: 30d
    starts_at: { on_event: "process.started" }
    on_breach:
      - action: notify_team
        args: { team: complaint, template: sla_breach }
ai_prompts:                            # optional — AI Companion tools scoped to this workflow (§4.8)
  - slug: suggest_classification
    label: "Suggest a category"
    description: "Ask AI to propose a BIPT category from the complaint text"
    available_at_states: [completeness_check, classification]
    default_include:
      case: true
      attachments: non_internal
      events: recent_20
    prompt_template: |
      You are helping BIPT triage an incoming complaint.
      Complaint: {{ case.description }}
      Attachments: {{ attachments_digest }}
      Suggest the most likely category (DSA, AI Act, Data Act, …), a confidence
      (low/medium/high), and your reasoning.
states: [ ... ]
```

**State types (v0 implements marked ✅; reserved types parse but raise `NotImplementedError` at instance-creation-time):**

| Type | v0 | Role |
|---|---|---|
| `start` | ✅ | Entry point. |
| `task` | ✅ | Produces a Task assigned to a user, team, or party. Completion drives transitions. |
| `parallel` | ✅ | Fan-out to N named branches. v0 join policies: `all`, `any`, `threshold:N`; `all_or_timeout` reserved until timers ship in §6.1. |
| `sub_process` | ✅ | Spawns another process on the same case. v0 mode: `spawn_and_wait` only; `spawn_and_continue` reserved. `pass_context` supported. |
| `end` | ✅ | Terminal; optionally sets `Case.status`. |
| `gateway` | ❌ reserved | Pure routing on context. Not used in v0 scaffolds — `task` + `transitions` with `when` covers the pattern. Reserved for when a use case emerges. |
| `wait` | ❌ reserved | Suspended, awaits external event. Covered in v0 by externally-satisfied `task` states (`completed_by: external_event`). Reserved. |

Reserved for later sub-projects: `timer`, `script` (§6.1 / §6.4). `triggers.debounce` reserved (no v0 fixture needs it; fires with §6.2 email arrival cadence concerns).

**Expression language: CEL** (Google's Common Expression Language, `celpy`). Safe, deterministic, typed, widely used (Kubernetes, GCP IAM). Variables: `task`, `context`, `case`, `event`, `now()`. Whitelisted functions only.

**Actions (hooks `on_enter`, `on_complete`, `on_timeout`):**

Initial vocabulary: `log_event`, `render_template`, `add_participant`, `set_context`, `set_case_status`.

Reserved (wired in later sub-projects): `start_timer`, `cancel_timer`, `send_email`, `share_attachments`, `call_ai_triage`, `forward_to_authority`, `send_reminder`, `escalate`, `notify_team`.

**Task assignment:**

```yaml
assign_to_team: complaint              # any member may claim
assign_to_user: "@case.created_by"     # CEL expression
assign_to_party: "authority:flanders"  # participant task
```

**Externally-satisfied tasks** — first-class concept:

```yaml
- key: await_flanders
  type: task
  assign_to_party: "authority:flanders"
  completed_by: external_event
  accepting_events:
    - type: external.reply
      party_role: authority
```

A participant task is completed by an external input event landing on the case (manually in v0 via `/api/cases/{id}/external-events`; automatically in the email sub-project).

**Parallel fan-out / fan-in:**

```yaml
- key: forward_to_regions
  type: parallel
  branches:
    - key: flanders
      states:
        - key: await_flanders
          type: task
          assign_to_party: "authority:flanders"
          completed_by: external_event
    - key: wallonia
      states: [ ... ]
    - key: brussels
      states: [ ... ]
  join: all
  transitions:
    - to: internal_review
```

**Sub-processes and triggered processes** — see §2 state types and the examples in §4.3.4 below.

**Concurrency on a case:** multiple `ProcessInstance`s may run simultaneously (one `main` + any number of `spawned` and `triggered`). All react to the same ordered event stream on the case.

**SLAs are observers, not drivers.** They emit events and may run side-effects on breach (notify, reminder, escalate), but they never move a process forward — flow control stays in `transitions`. This matters for regulator defensibility ("we noted the deadline, we chose to keep waiting, here's the record").

**Runtime semantics (dispatch loop):**

1. Open transaction; acquire `pg_advisory_xact_lock(namespace=1 /* case_dispatch */, hashtextextended(case_id::text, 0))` — namespaced per concern so timers (namespace=2), triggers, etc. never collide with case-dispatch locks.
2. Allocate `sequence_number = max + 1` for the case; insert the incoming event.
3. **Trigger dispatch** — for every loaded process with a matching `triggers:` rule, evaluate `when`; spawn a new instance if satisfied.
4. **Per-active-instance evaluation** — for each running instance on the case: execute against the instance's `process_definition_snapshot` (never the live file). Evaluate current state's transitions against the new event/context; if a transition fires, emit paired `process.state_exited` / `process.state_entered`, run `on_enter` actions (which **append to an in-tx deferred-event queue**, processed in order after the triggering event; max 256 deferred events per inbound event, cycle detector fires `ProcessLoopError` if the same `(instance_id, state_key)` transition fires twice in one batch).
5. SLA evaluation (v0: parsed and validated only; firing in §6.1 timers sub-project).
6. Commit transaction; release lock.

**Bounds and guards:**
- Max deferred events per dispatch: **256** (configurable, `ENGINE_MAX_DEFERRED_EVENTS`). Exceeding raises `ProcessLoopError`, rolls back the transaction, emits `process.error` event.
- Cycle detector: tracks `(instance_id, state_key)` transitions within one dispatch batch; a repeat raises `ProcessLoopError`.
- Process definition binding: the engine uses `ProcessInstance.process_definition_snapshot` for everything — transitions, `on_enter` actions, state type lookups, sub-process specs. The live `processes/*.yaml` file is the source for *new* instances only.

**Validation at load time:** JSON Schema on file shape; CEL parse on every `when`; reachability analysis; no-dead-end check; assignment references (team slugs, party refs) resolve. A process failing validation is **not loaded**; admin UI surfaces it with the errors.

**Example process file (simplified DSA, v0 scope):**

```yaml
id: dsa
version: "0.1.0"
title: "DSA complaint (v0 scaffold)"
category: dsa
initial_state: received
slas:
  - name: final_response
    duration: 30d
    starts_at: { on_event: "process.started" }
    on_breach: [{ action: notify_team, args: { team: complaint } }]
states:
  - key: received
    type: start
    transitions: [{ to: completeness_check }]
  - key: completeness_check
    type: task
    assign_to_team: complaint
    task:
      title: "Verify the complaint is complete"
      form: [{ name: is_complete, type: boolean, required: true }]
    transitions:
      - to: classification
        when: "task.data.is_complete == true"
      - to: request_more_info
        when: "task.data.is_complete == false"
  - key: request_more_info
    type: task
    assign_to_team: complaint
    task: { title: "Ask complainant for missing information" }
    transitions: [{ to: completeness_check }]
  - key: classification
    type: task
    assign_to_team: complaint
    task:
      title: "Classify and identify targets"
      form:
        - { name: summary, type: text, required: true }
    on_complete:
      - action: set_context
        args: { key: summary, value: "@task.data.summary" }
    transitions: [{ to: do_handling }]
  - key: do_handling
    type: parallel
    branches:
      - key: region_forward
        states:
          - key: spawn_regions
            type: sub_process
            process: "forward_to_regions"
            mode: spawn_and_wait
            transitions: [{ to: internal_review }]
          - key: internal_review
            type: task
            assign_to_team: dsa
            task: { title: "Review region responses" }
            transitions: [{ to: done_forward }]
          - key: done_forward
            type: end
      - key: company_engagement
        states:
          - key: maybe_contact_companies
            type: task
            assign_to_team: dsa
            task: { title: "Decide whether to contact target companies" }
            transitions: [{ to: done_company }]
          - key: done_company
            type: end
    join: all
    transitions: [{ to: respond_to_complainant }]
  - key: respond_to_complainant
    type: task
    assign_to_team: complaint
    task: { title: "Send final response to complainant" }
    transitions: [{ to: closed }]
  - key: closed
    type: end
    on_enter:
      - action: set_case_status
        args: { value: closed }
```

Accompanied by `/processes/forward_to_regions.yaml` (parallel Flanders/Wallonia/Brussels branches, each participant task satisfied by external events; own SLA `region_response: 15d per_branch: true`) and `/processes/ack_and_forward_info.yaml` (triggered by `external.info_received` from complainant; ack → forward → end).

### 4.4 Event timeline

**The rule:** no state change happens without emitting a corresponding event in the same transaction.

**Envelope** (see §4.2 Event entity).

**Event-type catalog (v0):**

| Namespace | Events |
|---|---|
| `case.*` | `case.created`, `case.title_updated`, `case.description_updated`, `case.status_changed`, `case.closed`, `case.escalated`, `case.escalation_resolved` |
| `participant.*` | `participant.added`, `participant.removed`, `participant.role_updated` |
| `party.*` | `party.created`, `party.updated`, `party.anonymized` |
| `attachment.*` | `attachment.added`, `attachment.downloaded`, `attachment.deleted`, `attachment.category_changed`, `attachment.visibility_changed`, `attachment.content_verified` (sha256 re-verify on download matched stored checksum), `attachment.content_mismatch` (sha256 re-verify failed — evidence integrity alert) |
| `process.*` | `process.started`, `process.state_entered`, `process.state_exited`, `process.ended`, `process.error` |
| `task.*` | `task.created`, `task.claimed`, `task.completed`, `task.cancelled`, `task.reassigned` |
| `external.*` | `external.info_received`, `external.reply`, `external.forward_sent` |
| `template.*` | `template.rendered` |
| `reminder.*` | `reminder.sent` |
| `sla.*` | `sla.started`, `sla.breached` (shape only; firing deferred) |
| `comment.*` | `comment.added`, `comment.edited`, `comment.deleted` |
| `ai.*` | `ai.queried` (companion invocation logged), `ai.invocation_blocked` (attempt on an AI-disabled case, for audit), `case.ai_disabled`, `case.ai_enabled` (re-enablement), `party.ai_opt_out_set`, `party.ai_opt_out_cleared`. AI never acts *as* an actor — this namespace is for audit of human-invoked AI usage and of the consent/opt-out lifecycle. See §4.8. |
| `admin.*` | `admin.process_reloaded`, `admin.reassignment_override`, `admin.ai_agent_created`, `admin.ai_agent_updated` |

Each type has a JSON Schema validator in `backend/src/corpus/events/schemas/`. Unknown types fail validation at the service boundary — typo prevention.

**Invariants:**

1. Append-only. `UPDATE`/`DELETE` on `events` revoked from app role.
2. Per-case monotonic `sequence_number`, starting at 1.
3. Transactional pairing: any business-layer state mutation + its event happen in a single DB transaction.
4. Deterministic replay: applying events in order reconstructs the current state. v0 doesn't *use* this (mutable state alongside), but preserves the property for a future event-sourced migration.
5. Every event has an actor (`user`, `system`, or `party`).

**Concurrency model:**

- Single writer per case via `pg_advisory_xact_lock(namespace=1, hashtextextended(case_id::text, 0))` — namespace=1 reserved for case-dispatch; other concerns (timers=2, future triggers subsystem=3, etc.) use distinct namespaces so lock domains never collide. Documented in `backend/src/corpus/core/locking.py`.
- Multiple readers; no lock on reads.
- Idempotency: external event schemas declare idempotency keys where applicable; v0 staff-entered externals get a UUID.

**Dispatch loop** — see §4.3 runtime semantics.

**Reading:** `SELECT * FROM events WHERE case_id = ? ORDER BY sequence_number ASC`. Frontend renders each event via a typed renderer registry keyed by `event_type`, with a generic fallback for unknown types — forward-compatible when the backend ships new types ahead of the frontend.

### 4.5 API surface

**Style:** REST with `:verb` action endpoints for non-CRUD actions. JSON in/out (except attachment upload multipart, download stream). Cursor pagination. RFC 7807 Problem Details for errors. UUIDs and ISO 8601 UTC timestamps. OpenAPI generated by FastAPI; frontend consumes it to produce typed TS clients via `openapi-typescript-codegen`. Single `/api` prefix, no version yet.

**Auth seam:**

```python
def current_user(request, auth: AuthBackend = Depends(get_auth_backend)) -> User:
    return auth.resolve_user(request)
```

Two implementations: `DevStubAuthBackend` (reads `X-Dev-User-Email` or configured default — v0 ships this) and `EntraOIDCAuthBackend` (validates JWT bearer — future auth sub-project).

Route-level authz via composable dependencies: `require_admin`, `require_team_member("dsa")`, `require_case_access(case_id)`. In v0 mostly permissive for Complaint Team / admin (see §4.6.1 for the one-sentence rule); tightened in a later RBAC sub-project.

**Endpoint catalog (v0):**

**Cases**
- `GET /api/cases` — list (filters: status, category, assigned_team, q, received_from/to; cursor-paginated).
- `POST /api/cases` — create.
- `GET /api/cases/{id}` — detail (with participants, active instances, recent events summary).
- `PATCH /api/cases/{id}` — update.
- `POST /api/cases/{id}:close` — close action.
- `POST /api/cases/{id}:escalate` — `{level, reason}`.
- `POST /api/cases/{id}:resolve-escalation` — `{note}`.
- `POST /api/cases/{id}:send-reminder` — `{target_kind, target_id, template_slug, note?}` (v0: logs event only; later: dispatches email).

**Timeline**
- `GET /api/cases/{id}/events` — paginated with type prefix filter, actor filter, date range.

**Parties & participants**
- `GET/POST /api/parties`, `GET/PATCH /api/parties/{id}`.
- `GET/POST /api/cases/{id}/participants`, `PATCH/DELETE /api/cases/{id}/participants/{id}`.

**Attachments**
- `POST /api/cases/{id}/attachments` — multipart upload.
- `GET /api/cases/{id}/attachments` — list with filters (category, is_internal, event_id).
- `GET /api/cases/{id}/attachments/{id}` — metadata.
- `GET /api/cases/{id}/attachments/{id}/download` — streamed binary. **Re-computes sha256 during streaming**; on match emits `attachment.downloaded` + `attachment.content_verified`; on mismatch emits `attachment.content_mismatch`, surfaces in `/api/admin/health`, rejects the download with 500. Event payload carries `request_ip` + `user_agent` for audit.
- `PATCH /api/cases/{id}/attachments/{id}` — metadata edits.
- `DELETE /api/cases/{id}/attachments/{id}` — soft delete with `reason`.

**Processes**
- `GET /api/processes`, `GET /api/processes/{id}/{version}` — read-only (YAML is source).
- `POST /api/admin/processes:reload` — admin.

**Process instances**
- `GET /api/cases/{id}/process-instances`, `GET .../{id}`.
- `POST /api/cases/{id}/process-instances` — start on a case.
- `POST /api/cases/{id}/process-instances/{id}:cancel` — admin.

**Tasks**
- `GET /api/tasks` — my inbox + team queue, with filters.
- `GET /api/tasks/{id}`, `:claim`, `:complete`, `:cancel`, `:reassign` (admin).

**External events**
- `POST /api/cases/{id}/external-events` — `{event_type, from_party_id, occurred_at, summary, attachment_ids?}`. Shared pattern: v0 manual, email sub-project automates, same endpoint.

**Comments**
- `POST /api/cases/{id}/comments`, `PATCH/DELETE /api/cases/{id}/comments/{id}`.

**Templates**
- `GET /api/templates`, `POST /api/templates/{slug}:render`.

**Admin / support** *(v0 scope narrowed per decision C2-mixed — deferred items land in a later admin-polish sub-project or §6.4)*
- `GET /api/admin/health` — DB, process-registry, and evidence-integrity status (any `attachment.content_mismatch` alerts).
- CRUD: `/api/admin/categories/attachments` (v0 — explicitly requested).
- `GET /api/admin/categories/cases` (read-only v0; CRUD deferred — new case categories arrive via seed/migration).
- `GET /api/teams`, `GET /api/users` (read-only v0).
- `GET /api/me`.
- **Deferred from v0 admin surface** (land with §6.4 / admin-polish): `POST/PATCH /api/admin/categories/cases`, `POST/PATCH /api/admin/ai-agents`, `GET /api/admin/ai-usage`.

**Dashboard**
- `GET /api/dashboard/complaint-team` — aggregated widget data in one response (filters: category, date_range). Complaint Team / admin only.

**AI Companion**
- `GET /api/ai/agents` — list active agents available to the current user.
- `POST /api/cases/{id}/ai/interactions` — invoke. **Returns 403 `ai_disabled_for_case` if the case has `ai_disabled=true`**, emitting an `ai.invocation_blocked` event for audit. Body: `{agent_id, kind, prompt_slug?, prompt_override?, include: {case?, event_ids?, attachment_ids?}, user_message?}`. Returns rendered output + `AIInteraction` metadata.
- `GET /api/cases/{id}/ai/interactions` — list prior interactions on this case.
- `GET /api/ai/interactions/{id}` — detail (including `prompt_rendered`, visible only to the invoker + admins).
- `GET /api/ai/prompts?process_id=&state=` — list globally + workflow-scoped prompts available in the given context. Returns empty list for AI-disabled cases.
- `POST /api/cases/{id}:disable-ai` — body `{reason}`. Complaint Team or admin. Sets `ai_disabled=true`. Emits `case.ai_disabled`.
- `POST /api/cases/{id}:enable-ai` — body `{reason}`. **Admin only** (re-enabling after opt-out requires oversight). Emits `case.ai_enabled`. Refuses if any complainant Party on the case has `ai_opt_out=true` unless the Party's opt-out is cleared first.
- `PATCH /api/parties/{id}` — already supports `ai_opt_out` toggle. When flipped on a complainant Party, **all cases the party is on are automatically marked `ai_disabled=true`** (transactionally, emits `case.ai_disabled` per case). When cleared, disabled cases are **not** automatically re-enabled — staff must explicitly call `:enable-ai` per case, forcing a human decision.
- `GET /api/admin/ai-agents` — read-only listing. **v0 ships only the seeded `dummy` agent**; `POST`/`PATCH` and agent CRUD UI ship with §6.4 behind the DPO-approval gate for real providers.
- Usage rollups (`/api/admin/ai-usage`) **deferred to §6.4** — meaningless under dummy-only A3.

**Error shape (RFC 7807, single envelope everywhere):**

```json
{
  "type": "about:blank",
  "title": "Validation failed",
  "status": 422,
  "detail": "One or more fields invalid",
  "request_id": "abc-123",
  "errors": [
    { "field": "received_at", "code": "invalid_format", "message": "Expected ISO 8601" }
  ]
}
```

**Validation:** Pydantic v2 on every request body; `ValidationError` → 422 with `errors[]`.

### 4.6 Staff UI

**Stack:** React + Vite + TypeScript (strict). shadcn/ui + Tailwind. React Router. TanStack Query. React Hook Form + Zod. `openapi-typescript-codegen` committed output (reviewable in PRs). i18next wired with EN locale only (retrofit-free expansion later). Lucide icons. Fonts via `fontsource` (no Google Fonts — no third-party fetches).

**Accessibility:** WCAG 2.1 AA target. Semantic HTML, landmark regions, keyboard nav, focus rings, 4.5:1 contrast, labeled form errors. `axe-core` snapshots in CI. Manual screen-reader pass before any "live" milestone.

#### 4.6.1 Role-based visibility (v0 rule)

> If `current_user.is_admin` or `current_user.team.slug == 'complaint'`, case list and dashboard are unscoped; otherwise filtered to cases where the user's team is (currently or historically) assigned a task, or where the user is (or was) assigned directly.

Simple, testable, generalizable when the RBAC sub-project lands.

#### 4.6.2 Page inventory

| Path | Audience | Content |
|---|---|---|
| `/login` | all | Dev stub input; replaced by OIDC flow later. |
| `/` | Complaint + admin | **Dashboard** (widgets below). |
| `/` | other staff | **My Work** — open tasks + team queue + recent cases I touched. |
| `/cases` | all (scoped) | Filterable table of cases. |
| `/cases/new` | all | Create form. |
| `/cases/:id` | scoped | **Case detail** — tabbed (Timeline default / Details / Participants / Attachments / Processes / Comments). Sticky header with reference, status, escalation badge, "Escalate", "Send reminder", "Close case" actions. Persistent **"AI Assistant"** button opens the companion drawer (§4.8). |
| `/tasks`, `/tasks/:id` | all | Inbox + task detail with dynamic form. |
| `/parties`, `/parties/:id` | all | Directory + detail. |
| `/processes`, `/processes/:id/:version` | all | Loaded processes + validation status. |
| `/admin/categories/attachments` | admin | CRUD (v0 — explicitly requested). |
| `/admin/categories/cases` | admin | **Read-only v0** — CRUD deferred; new categories via migration. |
| `/admin/teams`, `/admin/users` | admin | Read-only v0. |
| `/admin/ai-agents` | admin | **Read-only v0** (shows the seeded dummy agent); CRUD ships with §6.4. |
| `/admin/ai-usage` | admin | **Deferred to §6.4** — meaningless under dummy-only A3. |

#### 4.6.3 Complaint Team dashboard widgets

- **Portfolio overview** — cards per category (DSA, AI Act, Data Act, …) with counts: Open, Stalled, At risk, Breached, Closed this month.
- **Escalations** — currently escalated cases with level, reason, days open.
- **Stalled cases** — top N by "days since last event" across all categories. **v0 fixed 14-day threshold**; per-category config lands with §6.1 Timers.
- **Upcoming deadlines** — **cut from v0** — placeholder without SLA firing; widget lights up when §6.1 Timers ships.
- **Recent activity** — last 20 events across all cases, grouped by case.
- **My tasks / My team's queue** — personal inbox.

Row actions from dashboard tables: *open*, *escalate*, *send reminder*, *assign*.

#### 4.6.4 Escalate & Send Reminder

**Escalate** — case-level flag (`is_escalated` + `escalation_level` + reason). UI: badge in header + dashboard column; modal for raise/resolve. Events: `case.escalated`, `case.escalation_resolved`.

**Send Reminder** — no new entity; a communication action that logs an event in v0 and dispatches email in the email sub-project (same endpoint, same event shape). UI: modal with target picker (party / user / team), template picker filtered to `reminder.*`, live preview in recipient's language, optional staff note. Event: `reminder.sent`.

#### 4.6.5 Key UI patterns

| Pattern | Implementation |
|---|---|
| Timeline rendering | Registry `eventType → renderer(event) → {icon, title, body?, meta?}`. One file per event-type namespace in `frontend/src/events/renderers/`. Generic fallback row for unregistered types. |
| Party picker | Combobox: search `/api/parties?q=...`; inline "create new party" flow; language mandatory on create. |
| Task form rendering | `FormRenderer` walks typed JSON form schema from DSL, maps to shadcn inputs. |
| Attachment upload | Drag-drop + picker; client-side size/type courtesy check; server is authoritative; progress bar; optimistic row insert on success. |
| External event form | Modal: event_type, from_party (picker), occurred_at, summary, attachments. Submits to external-events endpoint. |
| **AI Assistant drawer** | Right-edge slide-out drawer on case and task pages. Contents: agent picker (if >1 active), **prompt picker** (contextual workflow prompts at top when on a task page, global prompts below), "what to include" checkboxes (case text / event list / attachment list — `is_internal` excluded by default), editable prompt textarea, Send. Response panel with per-paragraph and whole-response copy buttons. Previous interactions on this case listed at the top (collapsed, most recent first). See §4.8. **When `Case.ai_disabled=true`, the button is replaced by a locked icon with tooltip "AI assistance disabled for this case" + the recorded reason.** |
| **AI-disabled badge** | Red badge on case header when `ai_disabled=true`, showing the reason (`complainant opt-out` / `staff decision` / `policy`). Toggle-AI action in the case menu (visible to Complaint Team + admin), with distinct "Enable AI" requiring admin role. |
| **Complainant opt-out control** | On the Party detail page (for individuals), an "AI processing opt-out" checkbox with explanatory text: "When enabled, CORPUS will not let any AI tool process this person's complaints." Source field captures how the opt-out was recorded (intake form / staff / email). |
| **PII warning** | When the selected agent's `region` is `non-eu` and the selected includes attachments containing PII or personal Party data, a yellow banner in the drawer reminds the user that this invocation shares personal data with an external processor outside the EU. Non-blocking; logged in the interaction payload. |
| Confirmation modals | Destructive/externally-visible actions require a `reason` text. |
| Toasts | Single global system; errors render 7807 envelope's title/detail. |

#### 4.6.6 State management

Server state via TanStack Query (cache keys include case-id for clean invalidation). Local UI state in React state; small cross-component store in Zustand (current user, language, toast queue). No Redux.

#### 4.6.7 Frontend layout

```
/frontend/src
  /api              Generated OpenAPI client + typed wrappers
  /app              Root, router, providers
  /auth             Current-user context, login page
  /components       shadcn/ui re-exports + primitives
  /pages            cases, tasks, parties, processes, admin
  /events/renderers One file per event-type namespace
  /forms            FormRenderer + field components
  /hooks            TanStack Query hooks per domain
  /lib              utils, formatting
  /styles           Tailwind config, globals
  /i18n             locales/en.json, i18next config
```

### 4.7 Infra, CI/CD, local dev workflow

**First-run experience:**

```bash
git clone corpus.git && cd corpus
cp .env.example .env
docker compose up -d
# wait ~15s for containers + migrations + seed
open http://corpus.localtest.me
open http://traefik.localtest.me:8080
```

**Day-to-day via `justfile`** (optional QoL; underlying commands documented):

`up`, `down`, `logs [svc]`, `shell [svc]`, `test`, `test-fe`, `lint`, `typecheck`, `reset-db`, `seed`, `migrate`, `mig-new <msg>`, `e2e`, `codegen`.

**Backend tooling:**

- Python 3.12 + `uv` for deps/venv.
- FastAPI + Pydantic v2 + SQLAlchemy 2 + Alembic.
- `ruff` (lint + format), `mypy` (strict on core modules).
- `pytest` + `pytest-asyncio` + a real-Postgres fixture (choice deferred: `pytest-postgresql` vs `testcontainers-python` — see Open Questions §8).
- `structlog` → JSON to stdout.
- `celpy` for DSL expressions.
- `python-magic` for content-type sniffing.
- `jinja2` for template rendering.
- **No background-job runtime in v0.** APScheduler (or Celery+Redis / pg_boss) lands with the timers sub-project alongside actual SLA firing.

**Frontend tooling:**

- `pnpm` package manager.
- Vite + React 19 + TypeScript strict.
- shadcn/ui + Tailwind + Lucide + fontsource.
- TanStack Query + React Router + React Hook Form + Zod.
- `openapi-typescript-codegen` (or `openapi-typescript`) generates types at `just codegen`.
- Vitest (unit), Playwright (e2e), axe-playwright (a11y).
- ESLint + Prettier (or Biome, team choice).

**Docker Compose shape:** see §4.1. Profiles: default, `tools` (adminer), `e2e` (Playwright runner), `prod` (overlay for prod-like local), `remote` (adds Tailscale sidecar for remote access — see §4.7.1), `demo` (extends `remote`: enables Basic Auth middleware and Tailscale Funnel for time-boxed public exposure).

**Database & migrations:** single Postgres, with logical schema grouping by module family (proposed: `core` for cases/parties/events/attachments/templates/users/teams/categories; `workflow` for process_instances/tasks). The exact schema boundary is a detail for implementation — the intent is readable grouping, not microservice-like isolation. Alembic `env.py` aware of the split via multiple metadata objects. Autogenerate + human review. `api` entrypoint waits for db → runs `alembic upgrade head` → seeds if empty → starts Uvicorn. CI runs `alembic check` on every PR.

**Seed data:** `backend/scripts/seed.py`, idempotent, detects empty `teams` table and runs. Re-runnable with `just seed` after `just reset-db`.

**CI (GitHub Actions, one workflow `ci.yml`):** lint + typecheck (parallel) → backend tests (spins up Postgres service) → frontend unit → **OpenAPI-spec drift check** (fails if `just codegen` produces diff) → **`pip-audit` + `npm audit`** (non-blocking warnings unless CVE severity ≥ high) → e2e (Compose up with e2e profile) → **axe-playwright accessibility snapshots on key pages (case detail, dashboard, task detail)** → build images (artifacts; no push yet).

**Container image strategy:** multi-stage Dockerfiles for both. Backend: base (python:3.12-slim) → deps (`uv sync`) → runtime. Frontend: deps (pnpm) → build (vite) → runtime (static-web-server with `dist/`). Non-root containers.

**Pre-commit (fast, local, deterministic):** ruff, mypy (staged), eslint + prettier (staged; ESLint config includes a rule flagging hardcoded strings in JSX to neutralize i18next drift), check-large-files, no-commit-to-main, `gitleaks`. *(Moved to CI per decision C4-trim-network: `openapi-spec-drift`, `pip-audit`, `npm audit` — network-dependent or drift-prone checks run on PR, not locally.)*

**Secrets & config:** `.env` in dev (gitignored), `.env.example` is source-of-truth; pre-commit asserts every `os.getenv` reference has a documented entry. GitHub Secrets in CI. Azure Key Vault references in future deploy sub-project.

**Observability in dev:** structured JSON stdout logs, viewed via `docker compose logs` piped through `jq` by a `just logs-pretty` target. OpenTelemetry SDK initialized with console exporter in dev; real exporter wired in deploy sub-project.

**Branching:** trunk-based with short-lived feature branches; PR required for `main`; conventional commits encouraged, not required.

#### 4.7.1 Remote access via Tailscale (profiles `remote` + `demo`)

The default stack is local-only on `*.localtest.me`. When we want to reach CORPUS from another machine — or run a time-boxed demo for non-local viewers — we layer a Tailscale sidecar into the same Compose file via the `remote` profile. The host's existing Tailscale daemon and the prod Tailscale container are **not** touched.

**Shape:**

```yaml
# docker-compose.yml  (excerpt — remote profile)
services:
  tailscale:
    image: tailscale/tailscale:latest
    profiles: [remote, demo]
    hostname: corpus-dev                 # becomes corpus-dev.<tailnet>.ts.net
    environment:
      TS_AUTHKEY: ${TS_AUTHKEY:?set a one-time pre-auth key}
      TS_AUTH_ONCE: "true"
      TS_STATE_DIR: /var/lib/tailscale
      TS_EXTRA_ARGS: "--advertise-tags=tag:corpus-dev"
      # enables tailscale serve + funnel configuration via env:
      TS_SERVE_CONFIG: /config/serve.json
    volumes:
      - tailscale_state:/var/lib/tailscale
      - ./infra/tailscale:/config:ro
    cap_add: [net_admin, sys_module]
    devices: ["/dev/net/tun:/dev/net/tun"]
    restart: unless-stopped

  traefik:
    profiles: [remote, demo]              # default profile uses "localtest" traefik instead
    network_mode: "service:tailscale"    # shares tailscale's netns; only reachable at corpus-dev.<tailnet>.ts.net
    # ... labels + config unchanged otherwise
```

**Access modes:**

| Mode | Activation | Who reaches CORPUS | URL |
|---|---|---|---|
| Default (local only) | `docker compose up` | This host only | `http://corpus.localtest.me` |
| Remote (tailnet private) | `docker compose --profile remote up` | Any device on the tailnet | `https://corpus-dev.<tailnet>.ts.net` |
| Demo (public, time-boxed) | `docker compose --profile demo up` + `tailscale funnel 443 on` (run inside the sidecar) | Anyone on the public internet with the URL | Same `corpus-dev.<tailnet>.ts.net`, publicly resolvable with Tailscale-provided HTTPS |

**Auth layering in the `demo` profile:** Traefik adds a Basic Auth middleware on the `/` and `/api` routers. Users file at `infra/traefik/usersfile` (bcrypt-hashed, gitignored; `.example` committed). Basic Auth is **in addition to** the app's dev-stub auth — belt + suspenders before Entra ID lands. When the Real Auth sub-project ships, Basic Auth becomes optional.

**Isolation from prod and host:**

- Own Tailscale identity (`corpus-dev`), own tailnet device, own ACL tag.
- `network_mode: "service:tailscale"` means Traefik (and everything behind it) is only reachable on the `corpus-dev` tailnet IP — not on the host's public NIC, not on the host's tailnet IP, not on prod's tailnet IP.
- Stopping the CORPUS stack cleanly removes the `corpus-dev` device from the tailnet.
- Prod Traefik, prod Tailscale, and host Tailscale are untouched.

**`.env` additions:**

```
TS_AUTHKEY=tskey-auth-...       # one-time, created in Tailscale admin console
TS_HOSTNAME=corpus-dev
BASIC_AUTH_USERS_FILE=./infra/traefik/usersfile  # used by demo profile only
```

**Data-residency note:** Funnel terminates TLS on the local Tailscale daemon (on *this* host), not at Tailscale's servers. Traffic does not pass through Tailscale's infrastructure at the application layer — consistent with the no-MITM posture we want for a regulator's stack. Tailscale's control plane sees only connection metadata (WireGuard is peer-to-peer).

### 4.8 AI Companion

CORPUS ships with AI as a **companion**, not a workflow actor. AI never mutates case state and never sends external communications. It is a tool that humans invoke on demand to read, summarize, explain, translate, or draft text — the human copies what's useful into their own action. This keeps attribution clean (the human is always the actor), keeps the governance model tractable (no proposal-review machinery needed in v0), and keeps the regulator-defensibility story simple ("no autonomous AI action has ever been taken in this system").

The AI-as-workflow-actor model (proposals, review flow, DSL actions that produce AI outputs consumed by the engine) is deferred to the **AI Triage sub-project (§6.4)**.

#### 4.8.1 Platform principles

1. **AI never writes.** Output is text for humans to read. Any state change attributed to AI-assisted human action carries `Event.via_ai_agent_id` + `from_interaction_id` — the **human is the actor**, AI is the tool used.
2. **Every invocation is logged** with `agent_id`, `model`, `prompt_version`, `input_hash`, token counts, latency, and a summary of what was shared (`ai.queried` event + `AIInteraction` row).
3. **Explicit data sharing.** The user picks what to include in each call. No implicit case-wide context. `is_internal` attachments excluded by default; user must opt in per call.
4. **Non-EU endpoint warning.** The UI warns before sharing Party PII with an agent whose `region` is `non-eu`. Non-blocking (for dev flexibility), logged on the interaction.
5. **API keys are never in the DB.** `AIAgent.api_key_secret_ref` points to an env var or Key Vault secret name.
6. **Prompts versioned in git.** Workflow-scoped prompts live in process YAML (`ai_prompts:` block, §4.3); global prompts live in `backend/src/corpus/ai/prompts/*.j2`. Prompt changes go through PR review.
7. **Soft per-user rate limits** (env-configurable, default e.g. 60/hour) prevent runaway loops and accidental over-use on personal keys.
8. **Append-only interaction log.** `AIInteraction` rows are never deleted. Sensitive fields (`prompt_rendered`, raw output) are visible only to the invoker and admins via the API; the timeline's `ai.queried` event carries a summary, not the raw text.
9. **Complainant consent.** Every complainant may opt out of AI processing for their complaints. When a complainant Party has `ai_opt_out=true`, every case they participate in is automatically set `ai_disabled=true`, and no AI invocation is permitted — the API returns 403 and emits `ai.invocation_blocked` for audit. Re-enabling a case after an opt-out is admin-only and requires clearing the Party's opt-out first. The intake form (future sub-project §6.1) will include an explicit opt-out choice; until it ships, staff record the choice on the Party when registering the complaint. This is a platform default stance; BIPT Council may formalize or alter it later (flagged in §8).

#### 4.8.2 LLM client abstraction

```python
class LLMClient(Protocol):
    def complete(
        self,
        prompt: str,
        *,
        model: str,
        max_tokens: int | None = None,
        temperature: float | None = None,
        system: str | None = None,
        timeout_s: float = 60,
    ) -> LLMResponse: ...

@dataclass
class LLMResponse:
    output_text: str
    model: str
    prompt_tokens: int
    completion_tokens: int
    provider_raw: Any          # provider-specific response; for debugging / audit
```

**Concrete clients in v0:**

| Provider | Client | v0 role |
|---|---|---|
| `dummy` | `DummyLLMClient` | Returns canned responses keyed on `input_hash`; default agent; used in tests and CI. |
| `openai` | `OpenAIClient` | Direct OpenAI API; user's personal key for testing. |
| `anthropic` | `AnthropicClient` | Direct Anthropic API; user's personal key for testing. |
| `azure_openai` | `AzureOpenAIClient` | Shape only in v0; wired by the Azure Deploy sub-project. |

All four implement the same protocol. Structured outputs (JSON mode, tool-use) are **not in v0 scope** — companion invocations return free text. The AI Triage sub-project adds structured outputs when proposals need typed payloads.

#### 4.8.3 Prompt model

Two kinds of prompts, both rendered via Jinja2 with a sandboxed environment and a small context (`case`, `events`, `attachments_digest`, `user`, `task?`):

| Kind | Lives in | Scope | Surfaced in UI |
|---|---|---|---|
| **Global** | `backend/src/corpus/ai/prompts/*.j2` | Always available | In the AI Assistant drawer, always |
| **Workflow-scoped** | `ai_prompts:` block in the process YAML (§4.3) | Tied to a process; optional `available_at_states` narrows further | Only when the user is at a matching state; surfaced as one-click buttons at top of the drawer |

Validation at process-load time: each `ai_prompts` entry has unique slug within file, valid `available_at_states` refs, Jinja2 parse succeeds, variables referenced exist in the context.

#### 4.8.4 Interaction flow

1. User opens the AI Assistant drawer on a case (or from a task).
2. UI fetches available prompts from `GET /api/ai/prompts?process_id=X&state=Y` (if in task context) plus globals.
3. User clicks a prompt button (or writes a free one), reviews/edits "what to include" checkboxes, hits Send.
4. `POST /api/cases/{id}/ai/interactions` — backend:
   a. Authz check (user has access to the case).
   b. Resolve prompt template; render with the selected inclusions (redact `is_internal` attachments unless opted in).
   c. Compute `input_hash`.
   d. Check per-user rate limit.
   e. Emit PII-warning metadata if applicable.
   f. Call the LLM via the configured client.
   g. Persist `AIInteraction`; emit `ai.queried` event with summary payload.
   h. Return output + metadata.
5. UI renders the output with copy buttons.
6. When the user *uses* the output in an action (e.g., paste into an email draft and send it via the Send Reminder flow), that downstream action's event carries `via_ai_agent_id` + `from_interaction_id` — the AI-assist provenance.

#### 4.8.5 Testing

- `DummyLLMClient` used in all scenario and API tests — deterministic, no network, no keys.
- Fixture prompts in `backend/tests/fixtures/ai_prompts/` validate the rendering pipeline.
- Real-client tests gated behind an env flag (`TEST_LIVE_LLM=1`) and excluded from default CI — used when iterating on prompts with real providers.
- An integration scenario: user opens assistant → selects workflow prompt → sends → interaction created → event logged → output returned → user copies → downstream event carries `via_ai_agent_id`.

#### 4.8.6 Configuration (`.env` additions)

```
# Per-user invocation soft limit
AI_RATE_LIMIT_PER_USER_PER_HOUR=60
```

**v0 ships dummy-only** (decision A3). `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, and `AZURE_OPENAI_*` env vars and their corresponding `LLMClient` implementations land in §6.4 behind a DPO-approval gate: DPA with provider, per-agent redaction profile, lawful-basis documentation, `party.anonymized` → redaction hook for `AIInteraction.prompt_rendered`/`output_text`, and a boot-time assertion banning non-EU providers outside `CORPUS_ENV=dev`. The `AIAgent.api_key_secret_ref` field stays in the schema for forward compatibility but is unused in v0 (dummy needs no key).

---

## 5. Testing strategy

**Inverted pyramid for this event-driven system** — workflow scenario tests are the centerpiece, not the periphery.

| Layer | Volume (rough) | Speed | What it's for |
|---|---|---|---|
| Unit | 60% | <10ms | CEL eval, template render, content sniffing, event payload schemas, ID gen, filename sanitization. |
| **Engine scenario tests** | **25%** | **~50ms** | **The workflow engine against test YAML fixtures with replayed event streams. Most valuable tests in the repo.** |
| API contract tests | 10% | ~100ms | Each endpoint: happy path + top failure modes; real DB via transaction-per-test. |
| Frontend unit | small | fast | FormRenderer, event renderer registry, hooks, utils. |
| E2E | small | seconds | Happy-path flows against live Compose stack. |

**Workflow engine scenario tests** — live in `backend/tests/workflow/scenarios/`. Fixture processes in `backend/tests/fixtures/processes/`:

- `linear.yaml` — start → task → end.
- `branching.yaml` — task → condition → either_state.
- `parallel_all.yaml`, `parallel_any.yaml`.
- `parallel_external.yaml` — branches satisfied by external events.
- `sub_process.yaml` — parent calls child.
- `triggered.yaml` — process with `triggers:` block.
- `sla_shape.yaml` — SLAs declared; asserts validation passes without firing.

Each fixture gets dedicated scenario tests. New primitives land as a new fixture + scenario test — that's the contract.

**DB invariant tests** (real Postgres):

- `UPDATE events` / `DELETE events` rejected by privileges.
- **One deterministic 2-worker concurrency test** (decision D2-minimal): two workers hit one case simultaneously via `threading.Barrier`; assert `sequence_number` ends contiguous and both events land in-order. Not a stress/fuzz test — a single functional correctness assertion for the advisory-lock cornerstone.
- Every business-layer mutation pairs with exactly one event of the expected type.
- Soft-deleted attachments hidden from default list, present in timeline.
- Partial unique on `process_instances` (main running) holds under a concurrent double-start attempt (one wins, the other gets a constraint-violation error).
- Attachment `content_mismatch` path: a storage byte-flip triggers `attachment.content_mismatch` event + rejects the download.

**Process YAML validation tests:** parametrized walk over `/processes/` + fixtures; asserts zero errors on valid, specific errors on invalid.

**Event-rendering golden-file tests:** each renderer has a canonical payload → stored `.txt` output comparison. Round-trip tests on backend schemas.

**API contract tests:** pytest + `httpx.AsyncClient` against a real app + real DB. Per endpoint: happy + authz fail + validation fail + edge. OpenAPI schema snapshot-tested.

**Frontend tests:** Vitest for hooks/renderers/utils. Playwright for happy-path E2Es: login → create case → add party → upload attachment → start process → complete task → close. axe-playwright snapshots on key pages.

**Migration tests:** `alembic upgrade head` on fresh DB in CI; `alembic check` asserts no pending autogenerate.

**Performance budgets — deferred from v0 CI** (decision D1-defer).

At 100s/year, a 10k-case perf budget is ~100 years of data and the problems it would catch (N+1, missing indexes) surface long before 10k cases to any human using the UI. The seed-to-10k script stays in the repo; `just bench` runs it manually before each v0.x release as a sanity check. Full perf-testing infrastructure lands with the real-data sub-project after Foundations is in real use.

**Testing culture:** *Tests that run in CI are a spec written in code — if a behavior matters, there's a test that fails when it breaks.*

---

## 6. Deferred / future sub-projects

Each entry below is a candidate for its own brainstorm → spec → plan cycle. **Recommended phasing** under Option B (platform-first): Timers first, then Email I/O, then Public Intake, then AI Triage, then Share Bundles, etc. SLAs are a non-negotiable regulator obligation — they are set at the EU level, not internal targets — which is why §6.1 is Timers, not Public Intake.

### 6.1 Timers & SLAs **(recommended next after Foundations — mandatory)**

- **Why first:** CORPUS's obligations include EU-mandated deadlines (DSA acknowledgement windows, response-to-complainant timelines, Article-21-type review SLAs, regional-authority response clocks). These are externally-set regulator obligations, not internal targets. The platform's job is to track them reliably, surface breaches clearly, and generate the auditable record that BIPT can defend to the Council and to EU bodies. Everything else (Email I/O, Public Intake, AI Triage) depends on or is qualified by this.
- **What:** actual timer firing (durable scheduler); SLA breach detection + `notify_team` / `send_reminder` / `escalate` side-effects on breach; `all_or_timeout` join policy; per-branch SLAs on parallel fan-out; the `Upcoming deadlines` dashboard widget lighting up; `Stalled cases` widget becoming per-category configurable; `start_timer` / `cancel_timer` DSL actions implemented; `on_timeout` hook firing.
- **Depends on:** Foundations.
- **V0 hooks in place:** `slas:` block parsed and validated in DSL; `start_timer`/`cancel_timer`/`on_timeout` reserved action vocabulary; `slas:` stored on process definitions; `case.ai_disabled` / escalation lifecycle.
- **Key open questions:** durable scheduler choice (Celery+Redis? `pg_boss` via asyncpg? Postgres-backed custom using `LISTEN/NOTIFY` + advisory locks?); timezone handling (UTC in DB, render local); business-day vs calendar-day SLAs per-category; reminder-cadence defaults; what should happen on engine downtime during a scheduled firing (catch-up vs skip).

### 6.2 Email I/O (recommended second — naturally pairs with Timers)

- **What:** inbound mailbox parsing (new complaints, replies threaded to existing cases) and outbound sending (acks, info requests, reminders that §6.1 schedules, forwards). Probably Exchange Online via Microsoft Graph API given BIPT's Microsoft-shop context.
- **Depends on:** Foundations; timers (for reminder dispatch).
- **V0 hooks:** Template entity; `reminder.sent` and `external.*` events; `Attachment.event_id`; `external-events` endpoint as shared client.
- **Key open questions:** Graph vs IMAP; thread-matching strategy (`References`/`In-Reply-To` vs reference-in-subject); bounce handling; DKIM/SPF setup; legal archive retention; opt-in to store full raw emails as `.eml` attachments for evidence.

### 6.3 Public Intake Form (recommended third)

- **What:** multilingual public form (NL/FR/EN, DE later) for citizens/businesses to submit complaints; file upload; GDPR consent; **explicit AI-processing opt-out checkbox** (maps to `Party.ai_opt_out`); captcha / anti-spam; accessibility (AnySurfer).
- **Depends on:** Foundations; Email I/O (for acknowledgement email).
- **V0 hooks already in place:** `Case`, `Party`, `Attachment` entities; language per party; storage abstraction; external-events endpoint pattern; i18next framework; `Party.ai_opt_out` + `ai_opt_out_source` fields.
- **Key open questions:** anonymous vs email-verified vs eID/itsme; captcha choice (hCaptcha EU? Cloudflare Turnstile?); file type / size limits for public uploads; rate limiting; legal copy approval; public-URL strategy.

### 6.4 AI as workflow actor (proposal + review)

*Note: the **AI Companion** (humans invoke AI tools on demand; AI never writes) ships in **v0** — see §4.8. This sub-project adds the **workflow-actor** model: AI produces proposals that the engine surfaces as review tasks, and accepted proposals drive state changes.*

- **What:** `AIProposal` entity + mandatory human review flow; DSL actions (`ai_suggest_classification`, `ai_check_completeness`, `ai_draft_acknowledgement`) that create proposals; review-task flavor with Accept/Reject/Modify/Supersede actions; structured-output LLM client support (JSON mode / tool-use); cost caps + rate limits enforced at the engine level (v0 only has soft per-user caps); per-action review-policy in DSL (`require_review: true | false`, default true); confidence thresholds that can auto-flag low-confidence proposals; prompt library governance (per-category, versioned); multi-step agentic flows.
- **Depends on:** Foundations (AI Companion infra — LLM clients, AIAgent entity, AIInteraction log, events — already built); Email I/O (to create realistic unstructured inputs).
- **V0 hooks:** `AIAgent` + LLM clients in place; events namespace `ai.*` in place; `call_ai_triage` / `call_ai_classify` reserved DSL action names.
- **Key open questions:** model choice for each action (gpt-4o vs gpt-4.1 vs Claude vs open-source?); prompt library governance (who reviews, who approves); degradation policy if provider is down (fall back? pause process?); fine-tuning vs RAG for category taxonomy knowledge; **mapping complainant opt-out (§4.8 principle 9) to workflow-actor mode** — processes likely need a branch "run AI actions" vs "skip AI actions" at each AI-using state.

### 6.5 External Forwarding & Share Bundles

- **What:** staff select a subset of attachments; CORPUS generates a revocable, time-limited, tokenized share link per recipient party; forward email embeds link; zip streamed on access; every access logged.
- **Depends on:** Foundations; Email I/O.
- **V0 hooks:** `is_internal` + `category_id` on Attachment; storage abstraction with SHA-256 integrity; `external.forward_sent` event.
- **Key open questions:** PDF watermarking per recipient; email-code verification on first access; share-link revocation UX; default expiry (30/60/90 days configurable?); single-use vs reusable links; zip bundling streaming vs temp-file.

### 6.6 Real Auth (Entra ID / OIDC)

- **What:** replace DevStubAuthBackend with EntraOIDCAuthBackend; JWT validation; claim-to-user mapping; session management.
- **Depends on:** Foundations.
- **V0 hooks:** `AuthBackend` protocol; `User.external_id`; route-level auth dependencies.
- **Key open questions:** role / group mapping from Entra claims; MFA requirement; service accounts; session duration; logout semantics; token refresh strategy.

### 6.7 Dashboards, Analytics & Council Reporting

- **What:** richer aggregations for the Complaint Team dashboard; per-category analytics; Council-report exports (PDF/CSV); KPI dashboards.
- **Depends on:** Foundations + accumulated real usage data.
- **V0 hooks:** events-as-source-of-truth; `/api/dashboard/complaint-team` endpoint pattern; category taxonomy.
- **Key open questions:** export formats (PDF? docx? CSV?); pre-aggregated materialized views vs query-time; date semantics (received vs closed vs fiscal); who sees what (Council-level vs internal).

### 6.8 Workflow Designer UI

- **What:** no-/low-code editor so Complaint Team can author/edit processes without writing YAML; in-app editing or PR-generation flow.
- **Depends on:** Foundations; stable DSL (may require ~1 real-world iteration to stabilize primitives).
- **V0 hooks:** YAML is authoritative; validation pipeline already runs at load.
- **Key open questions:** in-app direct-edit vs PR-generation-via-UI (audit implications differ); permissions (who can author? propose? approve?); validation-error UX; state diagram visualization (`reactflow`?); rollback semantics.

### 6.9 Azure Deployment

- **What:** Azure Container Apps for api/web, Azure Database for PostgreSQL Flexible Server, Storage Account (Blob), Key Vault, Application Insights, Entra AD app registration, VNet + private endpoints, CI deploy pipeline, backup/restore runbook.
- **Depends on:** Foundations; Real Auth (for actual production use).
- **V0 hooks:** 12-factor config; container images; structured JSON stdout logs; OpenTelemetry SDK (console exporter switches to OTLP); `ObjectStore` interface with `AzureBlobObjectStore` to implement.
- **Key open questions:** VNet integration depth; blue/green vs revision-based rollout; cost budget; backup SLOs; Postgres HA tier; private endpoints; log-retention policy; alerting thresholds; on-call.

### 6.10 Anonymization Script

- **What:** retention-policy-driven job that anonymizes PII on closed-for-N-days cases by flipping `Party.is_anonymized` and nulling PII fields.
- **Depends on:** real usage + a retention policy decision.
- **V0 hooks:** `is_anonymized`, `anonymized_at` on Party; `party.anonymized` event.
- **Key open questions:** retention periods per category; what gets anonymized (PII only? full party?); what stays (statistical data, case narrative); reversibility (legal request); cross-case handling when a party appears in multiple cases at different retention stages.

### 6.11 Virus Scanning (required before public intake goes live)

- **What:** ClamAV (or Microsoft Defender ATP equivalent) integration; scanning in `ObjectStore.put` before commit; quarantine flow for infected files.
- **Depends on:** Foundations; Public Intake Form.
- **V0 hooks:** `ObjectStore.put` is the single choke point for all uploads; scan slots in cleanly.
- **Key open questions:** ClamAV (free, self-hosted) vs managed; signature-update cadence; quarantine retention; re-scan on signature update; performance on large files.

### 6.12 RBAC refinement

- **What:** proper role/permission system; per-category access; per-case ACL for sensitive cases; audit of access attempts.
- **Depends on:** Foundations; Real Auth.
- **V0 hooks:** simple team-membership-based rule; composable FastAPI dependencies.
- **Key open questions:** role taxonomy; ACL granularity (category? case? field?); separation-of-duties rules (no escalation author as own resolver?); admin-bypass logging.

### 6.13 Webhook out / integrations

- **What:** outbound webhooks to other BIPT systems on event types (e.g., case closed → notify CRM). Possibly inbound webhooks from partner systems.
- **Depends on:** Foundations.
- **V0 hooks:** event stream ready to subscribe against.
- **Key open questions:** delivery guarantees (at-least-once + idempotency?); auth/signing; retry policy.

### 6.14 Additional items noted but not yet scoped

- **Public-facing complainant portal** (beyond intake form) — see case status, reply, upload more — larger than any single sub-project.
- **Operator-facing portal** — companies log in to respond.
- **Real-time updates** on the case timeline (WebSocket / SSE) — not essential at 100s/year but maybe nice.
- **Mobile layouts** — responsive where free; native apps almost certainly not in scope.
- **Command palette / keyboard shortcuts** — power-user QoL.
- **Workflow visualizer** (state-machine diagram via `reactflow`) — helpful when processes get complex.
- **Dark mode** — not a regulator priority.
- **Process versioning & migration of in-flight instances** — when a process is updated mid-case, what happens? Need a policy.

---

## 7. Decisions log

Each decision records the *why*, so future-us can judge edge cases without re-litigating.

1. **Platform-first (Option B) over DSA-MVP-first (Option A).** Why: user has a clear mental model of the workflows; multiple regulatory domains on the roadmap; 100s/year scale makes the abstraction cost cheap; platform investment pays back on the second process.
2. **Python 3.12 + FastAPI + SQLAlchemy 2** (Approach 1 over .NET / Camunda-embedded). Why: AI ecosystem richness for the later triage sub-project; modern typing (Pydantic v2); developer velocity; permissive licensing throughout.
3. **Custom YAML workflow DSL over BPMN (Camunda 8) or SpiffWorkflow.** Why: DSL primitives ("forward to 3 regions with 15-day SLA, reminder N days before, escalate on breach") match regulator mental model directly; BPMN contorts the same semantics; Camunda's self-hosted footprint is disproportionate at 100s/year; SpiffWorkflow is LGPL (acceptable but less clean).
4. **CEL (Google Common Expression Language) for conditions.** Why: safe, deterministic, typed, widely used (Kubernetes, GCP IAM); `celpy` mature Python implementation.
5. **Modular monolith over microservices.** Why: 100s/year volume; small team; microservices would add coordination cost with no scale benefit.
6. **Postgres with logical schema grouping by module family** (proposed `core` + `workflow`). Why: clean module ownership; migration isolation; readable at a glance. Exact boundary refined during implementation.
7. **Mutable state + append-only event log** over full event sourcing. Why: right complexity for scale; preserves the replay property for a future migration.
8. **Per-case Postgres advisory lock** for concurrency. Why: cheap (microseconds); exactly-once per case; scales horizontally later with zero code change.
9. **Local-first dev on Docker Compose**, Azure deployment deferred to its own sub-project. Why: tight iteration loop; defers cloud-billing complexity; 12-factor patterns make eventual Azure deploy mechanical.
10. **Traefik over nginx as edge proxy.** Why: label-based service discovery; dashboard; automatic updates on service change; TLS via Let's Encrypt trivial later.
11. **React + Vite + TypeScript + shadcn/ui + Tailwind.** Why: modern, strong types, WCAG AA baseline from Radix, no lock-in (shadcn is source-copied), team-agnostic.
12. **i18next wired from day one** despite EN-only UI in v0. Why: retrofit pain avoided.
13. **Attachment storage via `ObjectStore` abstraction** (Filesystem dev / AzureBlob prod / in-memory tests). Why: clean swap at deploy time; uniform interface; testability.
14. **All attachment downloads stream through API** in v0 (no presigned URLs yet). Why: uniform auth + audit log across backends; add presigned as optimization later.
15. **`is_internal` and `category_id` on Attachment in v0** (before sharing sub-project lands). Why: retrofit-avoidance — classify as uploaded.
16. **Soft-delete attachments** (not hard delete). Why: evidence lifecycle — never truly lose evidence.
17. **Sniffed content-type + SHA-256 checksum** per attachment. Why: evidence integrity; "filename is metadata, checksum is truth."
18. **Party as unified abstraction** (vs separate Complainant/Company/Authority tables). Why: single deduplication point; roles handled by `CaseParticipant`; reuse across cases natural.
19. **Multiple concurrent `ProcessInstance`s per case** (main + spawned + triggered). Why: real regulator work is concurrent — forward + direct engagement + info-received flows overlap.
20. **SLAs as observers, not drivers.** Why: regulator-defensibility — "we noted the breach, we chose to keep waiting."
21. **Event-stream-as-source-of-truth for engine reactions.** Why: uniform dispatch; same logic for user action, external input, future email parse.
22. **Complaint Team oversight via membership** (v0 rule: `is_admin` or `team.slug == 'complaint'`). Why: simplest testable rule; RBAC refinement is its own sub-project.
23. **Escalate as case-level flag** (not a separate entity in v0). Why: history reconstructable from timeline events; avoid over-modeling.
24. **Reminder = communication action** (no entity; event + optional email). Why: timeline is source of truth; email sub-project swaps in dispatch later.
25. **`:verb` action endpoints** (Google AIP-ish). Why: clear intent; avoids awkward sub-resource shapes.
26. **Cursor pagination** (not offset). Why: stable under inserts; predictable.
27. **RFC 7807 Problem Details.** Why: single error shape; single frontend handler.
28. **OpenAPI-generated typed client**, committed to git. Why: no drift; reviewable in PRs.
29. **`uv`** for Python deps. Why: fast, modern, reproducible lockfile, 2026 default.
30. **`pnpm`** for frontend. Why: fast, disk-efficient.
31. **GitHub Actions for CI** (not Azure DevOps Pipelines, despite Azure-future). Why: team-ubiquitous; cloud-agnostic; matches OSS option.
32. **Migrations run in `api` container entrypoint.** Why: "app running" and "schema up-to-date" become the same sentence; safest deploy story.
33. **structlog JSON stdout logs.** Why: Azure Container Apps / App Insights compatible with zero code change.
34. **Inverted test pyramid with engine scenario tests central.** Why: behavior contract for an event-driven system lives at the scenario level; unit tests would pass while missing DSA semantics.
35. **Golden-file event renderer tests.** Why: subtle UI regressions on timeline are silent without them.
36. **`static-web-server` (Rust) for prod frontend** (not nginx). Why: user preference for Traefik over nginx; static-web-server is tiny, has SPA fallback, no runtime ecosystem concerns.
37. **`localtest.me` for dev hostnames.** Why: wildcard DNS to 127.0.0.1; no `/etc/hosts` edits.
38. **Remote access via dedicated Tailscale sidecar in CORPUS Compose** (not shared host Tailscale, not shared production Traefik, not Cloudflare Tunnel). Why: host has both a host-level Tailscale daemon and a separate production Tailscale container already; reusing either would couple CORPUS to infra we must not disturb. A dedicated `corpus-dev` sidecar gives its own tailnet identity, its own `serve`/`funnel` rules, and a clean lifecycle (stopping CORPUS cleanly removes the device). Cloudflare Tunnel rejected because it would MITM all traffic at Cloudflare edge — problematic for a regulator's data-handling posture even in dev. Tailscale Funnel terminates TLS on the local daemon, no third-party visibility at application layer.
39. **Basic Auth middleware active only in `demo` profile**, on top of the dev-stub auth. Why: the dev-stub auth (`X-Dev-User-Email` header) is trivially impersonable; when we expose the stack publicly via Funnel for a demo, Basic Auth provides a real access barrier. When Entra ID auth lands, Basic Auth becomes optional.
40. **AI as Companion in v0** (humans invoke, AI returns text, humans act); **AI as workflow actor deferred to §6.4.** Why: governance shape benefits from being designed against real AI usage, not imagined; one working companion action (§4.8) proves the plumbing end-to-end without the risks of autonomous AI action. The workflow-actor model (`AIProposal` + review tasks) lands in §6.4 when there's enough real traffic to calibrate it.
41. **Dedicated `AIAgent` entity, not `User.is_ai`.** Why: human actors and LLM endpoints are different concepts; conflating them muddies the audit log. Multiple agents (summarizer, translator, etc.) each have their own identity, model, prompt version, region.
42. **Human is always the actor; AI is the tool.** `Event.actor_type` stays `user | system | party` — no `ai` value. Instead, events carry `via_ai_agent_id` + `from_interaction_id` to capture AI assistance provenance without muddying attribution. Why: a regulator needs clear "who decided this" answers; that answer is always a human when actions change state.
43. **Multiple LLM client implementations in v0** (`openai`, `anthropic`, `azure_openai`, `dummy`) behind one `LLMClient` protocol. Why: user tests with personal OpenAI/Anthropic keys; `dummy` keeps tests hermetic; `azure_openai` is prod-ready config-only swap.
44. **Workflow-scoped `ai_prompts` block in process YAML.** Why: prompts are inherently tied to the process that uses them; co-locating keeps authorship, review, and versioning in one place; contextual surfacing in the UI (prompts appear at states where they apply) improves staff usefulness.
45. **Complainant AI opt-out is honored at case level.** Why: proposed as BIPT platform default (subject to Council confirmation): data subjects can refuse AI processing of their complaints. Technically enforced: `Party.ai_opt_out` on complainants propagates to `Case.ai_disabled`; blocks every AI invocation; re-enablement is an admin action. Building this from v0 avoids retrofitting consent plumbing when the public intake form ships.

---

**Critic-informed amendments (after three-agent independent review):**

46. **v0 ships only the `dummy` LLM client** (decision A3). Real-provider clients (OpenAI/Anthropic/Azure OpenAI) move to §6.4 behind a DPO-approval gate requiring DPA with provider, per-agent redaction profile, lawful-basis documentation, and a `party.anonymized` → `AIInteraction` redaction hook. Why: Security critic flagged that personal OpenAI/Anthropic keys in a regulator stack route complainant data to US processors with no DPA, no redaction, and no retention control — not DPIA-ready. Locking to dummy in v0 preserves the AI-first-class UX without the compliance hazard.
47. **Process definition snapshots on `ProcessInstance`.** Why: YAML files in git are authoritative, but running instances must execute against a frozen JSON snapshot captured at instance-creation. PRs editing `processes/*.yaml` never break in-flight cases; migration of existing instances becomes an explicit decision. Architecture critic flagged this as a day-1 concern, not a §6.14 "need a policy" item.
48. **Dispatch-loop bounds + cycle detection.** Why: the engine's "actions may queue more events in the same batch" language was under-specified. v0 implements a deferred-event queue with `ENGINE_MAX_DEFERRED_EVENTS=256` and a `(instance_id, state_key)` cycle detector raising `ProcessLoopError` on repeats. Architecture critic flagged unbounded recursion risk.
49. **`Event.schema_version` column** + renderer dispatch on `(event_type, schema_version)`. Why: event payloads will evolve; without a version column, old events render incorrectly after a renderer change and the "deterministic replay" invariant is silently broken. Cheap now, impossible to retrofit at 50k events.
50. **Auth-backend refuse-to-boot guard.** Why: the dev stub (`X-Dev-User-Email`) is trivially impersonable; nothing in the spec prevented its accidental deployment in staging/prod. v0 adds a startup assertion: `AUTH_BACKEND=dev` only permitted when `CORPUS_ENV ∈ {dev, test}`; Basic Auth mandatory when stub is active and not bound to loopback.
51. **Evidence integrity hardening.** Why: SHA-256 was computed on upload but never re-verified on download; system-actor events were unattributed; downloads weren't logged with IP/user-agent. v0 adds: SHA-256 re-verify on every download (emits `attachment.content_verified` / `content_mismatch`); `Event.system_actor_ref` captures `{component, build_sha}`; `Event.request_ip` + `Event.user_agent` on user events.
52. **Share-bundle exclusion flags on v0 categories and events.** Why: Security critic flagged that when §6.5 ships, a staff mistake could forward `internal_note` attachments or AI interaction content to external authorities. v0 adds `AttachmentCategory.is_shareable_default` (defaulting `internal_note` to false) and flags `AIInteraction` + `ai.*` events as `excluded_from_share_bundle` at schema level.
53. **Namespaced advisory locks.** Why: `pg_advisory_xact_lock(namespace=1, hash)` with reserved namespaces per concern (case-dispatch=1, timers=2, future triggers=3) prevents cross-concern deadlocks once other subsystems acquire their own advisory locks. Cheap and future-proof.
54. **DSL primitive restraint (B-moderate).** Why: `gateway`, `wait`, `spawn_and_continue`, `debounce` aren't exercised by v0 scaffolds. v0 validator parses them but `NotImplementedError` at instance-creation, pointing to the sub-project that will implement them. Non-`all` join policies kept (cheap enough) — preserves platform-first signal without dead-code tax.
55. **i18next wired day-one with ESLint-rule enforcement.** Why: retrofit cost when NL lands is far higher than day-one wire-up. ESLint rule flagging hardcoded JSX strings neutralizes the drift-surface concern Pragmatist raised.
56. **Admin UI scope trimmed (C2-mixed).** Why: under A3, AI agent CRUD and AI usage rollups have near-zero v0 value. Case-category CRUD is rarely exercised. v0 keeps: attachment-category CRUD (you asked for it), read-only teams/users/AI-agents/case-categories. Defers CRUD on the rest to a later admin-polish sub-project.
57. **§6.1 is Timers, not Public Intake.** Why: SLAs are EU-mandated regulator obligations, not internal targets; the platform's core value proposition is reliable deadline tracking. Timers lands first after Foundations so the regulatory SLA loop closes as soon as possible.
58. **Dashboard widget trimmed: 6 → 5** (drop `Upcoming deadlines` placeholder; keep `Stalled cases` with fixed 14-day default). Why: placeholder widget until §6.1 delivers real value; fixed default keeps `Stalled` useful now without requiring the per-category config decision to be made pre-emptively.
59. **Pre-commit trimmed to fast + local + deterministic hooks** (C4-trim-network). Why: `openapi-spec-drift`, `pip-audit`, `npm audit` are network-dependent or drift-prone; pre-commit should be fast; CI is the gate. Keeps protection, cuts Friday-evening pain.
60. **Tailscale `remote` + `demo` profiles both kept** (C5-keep-all). Why: user explicitly asked for remote access; `demo` profile is thin incremental cost on top of `remote`; deferring either means re-designing under time pressure later.
61. **Golden-file event renderer tests kept** (C6-keep). Why: `.txt` string-output snapshots of tiny pure functions are not DOM-snapshot-brittle; they catch silent timeline rendering regressions that manual review will miss; cheap to maintain.
62. **10k-case CI perf budget dropped** (D1-defer). Why: at 100s/year scale, the budget doesn't meaningfully protect. Manual pre-release benchmark via `just bench` covers the sanity check.
63. **Single deterministic 2-worker concurrency test** (D2-minimal). Why: the advisory lock is too central to go untested; a `threading.Barrier`-synchronized 2-worker assertion is much cheaper to maintain than an N-worker stress fuzzer.
64. **Axe-playwright snapshots in CI on 3 pages** (D3-keep). Why: WCAG 2.1 AA / AnySurfer is a Belgian federal-body obligation, not best-effort. Automating the ~30-40% that's automatable is cheap protection; manual screen-reader pass still required for the rest.

---

## 8. Open questions

To resolve either before planning or during implementation. Each has a reasonable default.

1. **Complainant authentication on future intake form** — anonymous / email-verified / eID / itsme? *Default until resolved: email-verified.*
2. **Formal BIPT escalation ladder** — are `team_lead | council | leadership` the right levels, or does BIPT have a formal mapping? *Default: proposed levels; rename on input.*
3. **"Stalled" case heuristic** — days-without-activity threshold; per-category? *Default: 14 days, configurable, not per-category initially.*
4. **Reminder template authorship** — Complaint Team WYSIWYG vs pre-approved catalog only? *Default: pre-approved catalog managed by admins; WYSIWYG later if needed.*
5. **Region / authority official names** — "Flanders Media Regulator" vs another canonical name? *Default: placeholder; replace during seed-data review.*
6. **SLA clock policy** — calendar days vs business days? Per-category? *Default: calendar days; revisit in timers sub-project.*
7. **Retention policy specifics** for anonymization (years per category). *Default: defer to anonymization sub-project.*
8. **OSS release decision** — not yet; may affect some dependency choices. *Default: no OSS-hostile deps chosen.*
9. **DB-in-test choice** — `pytest-postgresql` vs `testcontainers-python`. *Default: `testcontainers-python` (more portable, matches prod Postgres version exactly).*
10. **Case reference format** — `CORPUS-2026-0042`? Another convention? *Default: `CORPUS-YYYY-NNNN` zero-padded 4 digits.*
11. **File size limit** — 100 MB default; confirm with BIPT IT. *Default: 100 MB configurable via `MAX_UPLOAD_MB` env.*
12. **Demo / seed case content** — realistic enough to be useful; anonymized enough to be public? *Default: fictional "Example.com" complaint.*
13. **`just` adoption** — primary interface or optional? *Default: optional; shell scripts in `/scripts/` mirror.*
14. **Frontend lint tool** — ESLint + Prettier vs Biome? *Default: ESLint + Prettier (ubiquitous); revisit if Biome reaches critical mass.*
15. **Basic Auth users file management** — committed `.example`, gitignored real; how do we bootstrap the first credential and rotate later? *Default: `htpasswd -B` generates a bcrypt line for `.env` to write into `infra/traefik/usersfile` at container start; rotation is a file replace + Traefik reload.*
16. **Tailscale tailnet ACL for `tag:corpus-dev`** — what should it allow/deny? *Default: allow all enrolled devices in the tailnet; tighten when real data lands.*
17. **Non-EU endpoint detection for PII warning** — how does the UI know an agent is non-EU? *Default: admin sets `AIAgent.region` explicitly at agent creation; UI trusts that field; `dummy` agent is `eu` by convention.*
18. **Cost estimation source** — where do we get per-token pricing for the cost_estimate field? *Default: hard-coded pricing table per (provider, model) in `backend/src/corpus/ai/pricing.py`, updated with each release; flagged as approximate in the UI.*
19. **Complainant AI-opt-out policy, formally** — platform default now says opt-out is respected case-wide; does BIPT Council want to (a) endorse as-is, (b) require explicit opt-*in* instead (AI off by default), or (c) restrict opt-out to specific categories? *Default: opt-out respected; opt-in flip is a config toggle we can add if the Council wants.*
20. **Clearing a complainant opt-out** — who is allowed, and what evidence is required? *Default: admin role; staff records a `reason` text; the expected justification is "written confirmation from complainant"; no automatic re-enable of case-level AI.*
21. **Workflow-scoped prompt governance** — since `ai_prompts` live in YAML, do Complaint Team authors get direct edit access to workflows, or do they propose via PR? *Default: PR-style via git for v0; the later designer UI will formalize an in-app propose-and-approve flow.*
22. **`ENGINE_MAX_DEFERRED_EVENTS` threshold** — 256 seems generous for a per-inbound-event budget; revisit once we see real workflow depth. *Default: 256.*
23. **Event schema migration strategy when `schema_version` bumps** — do we provide upcaster functions in code, or explicit per-version renderers? *Default: per-version renderers; upcasters only added when two payload shapes must coexist across a real migration window.*
24. **BIPT Council opt-out policy ratification** — Geoffrey is proposing the default; Council may adjust to opt-in-only, or restrict opt-out to specific categories. *Default: opt-out respected; toggle is configurable so the default flips without code change if ratification diverges.*
25. **How to seed the first AI agent in §6.4** — when real providers land, which one first (Azure OpenAI in Belgium Central, an EU-hosted open-source model via a self-run endpoint, or Anthropic)? *Default: Azure OpenAI Belgium Central first — best DPA + residency + model quality story; revisit if Belgium region lags model availability.*

---

## 9. Glossary

| Term | Meaning |
|---|---|
| **CORPUS** | Complaint Office Routing & Processing Unified System — this project's name. |
| **BIPT** | Belgian Institute for Postal Services and Telecommunications — the federal regulator building CORPUS. |
| **DSA** | Digital Services Act — EU regulation; BIPT is a designated authority. |
| **AI Act** | EU regulation on artificial intelligence; BIPT handles certain complaint categories. |
| **Data Act** | EU regulation on fair data access and use (referenced in conversation as "Cloud Act"); BIPT Market Team handles related complaints. |
| **Case** | A single complaint; the aggregate entity. |
| **Party** | Any entity referenced in a case (individual, organization, authority). |
| **CaseParticipant** | A party's role on a specific case. |
| **Event** | An append-only timeline entry; the engine's source-of-truth. |
| **ProcessDefinition** | A YAML file in `/processes/`; authoritative description of a workflow. |
| **ProcessInstance** | A running process on a case (main / spawned / triggered). |
| **Task** | A work item produced by a process state, assigned to a user, team, or party. |
| **Workflow DSL** | Our custom YAML language describing states, transitions, parallels, sub-processes, triggers, SLAs. |
| **SLA** | Service-level deadline tracked as an observer (emits events on breach; does not drive flow). |
| **Dispatch loop** | The engine's per-case serialized routine that reacts to a new event by firing transitions, spawning triggered processes, evaluating SLAs. |
| **Advisory lock** | Postgres feature used to serialize dispatch per case; cheap and scalable. |
| **CEL** | Common Expression Language (Google) — safe expression language for DSL conditions. |
| **Traefik** | Docker-native reverse proxy used as the edge. |
| **FastAPI** | Python web framework used for the API. |
| **shadcn/ui** | React component library used for the staff UI. |
| **Entra ID** | Microsoft's name for Azure AD; the future staff auth provider. |
| **WCAG 2.1 AA / AnySurfer** | Accessibility standard/label; Belgian federal-body obligation. |
| **AzureBlob / ACR / Container Apps** | Azure services used in the future deploy sub-project. |

---

*End of spec.*
