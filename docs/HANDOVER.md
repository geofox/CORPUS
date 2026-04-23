# CORPUS — Handover Document

**For:** the next AI assistant (or human collaborator) continuing this project.
**From:** a Claude Opus 4.7 session that ran 2026-04-17 through 2026-04-23.
**User:** Geoffrey Richard, BIPT.
**Date of handover:** 2026-04-23.
**Status at handover:** brainstorming complete; Foundations design spec written, amended twice (after two three-agent review rounds), committed and pushed to `main` at `github.com:geofox/CORPUS`. Implementation plan not yet produced. `writing-plans` skill **not yet invoked**.

---

## Read this first

You are taking over mid-project. The work so far has followed the **superpowers** `brainstorming` skill flow, which is:

```
brainstorm → write spec → critic/review loops → writing-plans (produce implementation plan) → execute
```

We are at the **end of the spec phase, about to hand off to `writing-plans`**. The user is aware; they may say "go" or "invoke writing-plans" or similar. **Do not start writing code yet** — the next legitimate step is producing the implementation plan.

Before you do anything else:

1. Read this document completely.
2. Read the **authoritative spec** at `docs/superpowers/specs/2026-04-17-corpus-foundations-design.md`. It is 1443 lines; it is the single source of truth for design decisions.
3. Read the **memory files** — a committed snapshot lives at `docs/memory/` (in this repo) and the live copy (if you're on the original author's machine) is at `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/`. Start with `MEMORY.md`, `user_profile.md`, `project_corpus.md`, and the `feedback_*.md` files.
4. Check the git log (`git log --oneline`) — you should see six commits all starting with `docs:`. Nothing else.

Only after those four steps should you respond to any user request substantively.

---

## 1. The project in one paragraph

**CORPUS** (Complaint Office Routing & Processing Unified System) is a complaint management platform for **BIPT**, the Belgian Institute for Postal Services and Telecommunications — the Belgian federal regulator. BIPT has been designated as the competent authority for a growing set of EU digital regulations (DSA, AI Act, Data Act) in addition to its pre-existing telecom-consumer and postal mandates. CORPUS is **configurable workflow platform, not a hardcoded CRM**: BIPT's Complaint Team authors processes in YAML (and, eventually, through a no-code designer) for each regulatory domain; the engine runs them; every state change becomes an append-only event on a case timeline; AI assists humans but never acts autonomously. Scale is **100s of complaints per year**. Running on Linux via Docker Compose today; Azure Container Apps (Belgium Central) is the eventual production target.

## 2. The user — how Geoffrey works

Geoffrey Richard is leading CORPUS at BIPT. Read the memory file `user_profile.md` for the full profile. Summary:

- **Terse, decisive.** He answers A/B/C/D multiple-choice questions with "A3" or "B-moderate" or just "keep". When he says a choice, he means it — don't re-litigate unless the downstream work reveals a problem.
- **Thinks holistically.** He has a clear mental model of BIPT's workflows and can describe them operationally.
- **Pushes back concretely.** When he disagrees he gives reasons (e.g., "wrong decision to reuse traefik as i've some service in prod").
- **Regulator mindset.** Treats GDPR, AnySurfer accessibility, data residency, and audit defensibility as first-class design inputs. This matters for every decision.
- **Adds requirements mid-flight.** Expect him to say "oh, also we'll need X" during design — rework earlier sections rather than deferring when the addition is load-bearing.
- **Wants comprehensive specs** with deferred sub-projects + decisions log + open questions (with defaults). He explicitly said so.
- **Wants conversation memory preserved** — the memory system in use is `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/`. Keep it current.
- **Wants critic rounds.** He asked twice for three-sub-agent independent reviews of the spec. Expect him to ask again.

Communication rhythm to match: be direct, offer 2–4 concrete options with a recommendation, explain the *why* in one or two sentences, and move on when he chooses. Don't overexplain after he's decided. Don't narrate your tool calls excessively.

## 3. The authoritative spec

**Path:** `docs/superpowers/specs/2026-04-17-corpus-foundations-design.md`
**Last commit:** `71a460c` (2026-04-23)
**Size:** 1443 lines, 9 top-level sections, 128 numbered decisions, 35 open questions, 18 sub-projects in §6.

Section map (memorise this — you will navigate here constantly):

- **§1 Executive summary** — what v0 is, what's deferred, pointers to the two review rounds.
- **§2 Context** — BIPT, mandates, users, constraints.
- **§3 Architectural approaches considered** — A1 (chosen: Python modular monolith + custom YAML DSL), A2 (rejected: Camunda/SpiffWorkflow), A3 (rejected: .NET + Blazor).
- **§4 System design**
  - §4.1 Runtime shape (Compose + Traefik)
  - §4.2 Domain model *(big section — ~20 entities)*
  - §4.3 Workflow DSL
  - §4.4 Event timeline
  - §4.5 API surface
  - §4.6 Staff UI *(includes Complaint-Team dashboard + AI Companion drawer)*
  - §4.7 Infra, CI/CD, local dev workflow *(includes §4.7.1 Tailscale remote access)*
  - §4.8 AI Companion
- **§5 Testing strategy** — inverted pyramid with engine scenario tests as centrepiece.
- **§6 Deferred / future sub-projects** — 18 entries.
- **§7 Decisions log** — 128 entries across four bands:
  - #1–45: original design decisions
  - #46–64: first three-critic review (architecture / security / pragmatism)
  - #65–82: second three-agent review (operational practice / templating / process catalog)
  - #83+ reserved for future
- **§8 Open questions** — 35 entries, each with a default.
- **§9 Glossary** — 30-ish regulator-specific and tech-specific terms.

**How to amend:** follow the established pattern. After a discussion where decisions are reached:
1. Make targeted `Edit`s to affected sections.
2. Append new decisions to §7 in the next numbered band.
3. Append new open questions to §8.
4. Update `**Last amended:**` in the header.
5. Self-review with the brainstorming-skill checklist (placeholders, consistency, scope, ambiguity).
6. Commit with a descriptive message in the existing style, push to `origin/main`.

## 4. The sub-project roadmap

Platform-first phasing (Option B — user's explicit choice over an MVP-first Option A) drives the order. After **Foundations** (this spec):

| # | Sub-project | Why this order |
|---|---|---|
| §6.1 | **Timers & SLAs** | **Next after Foundations.** EU-mandated deadlines are non-negotiable regulator obligations, not internal targets — bumped from original §6.3 to §6.1. |
| §6.2 | **Email I/O** | Naturally pairs with Timers (reminders need a channel); also carries the PDF letterhead pipeline. |
| §6.3 | **Public Intake Form** | Depends on Email I/O for acknowledgement. |
| §6.4 | **AI as workflow actor** (proposals + review) | Extends the AI *Companion* (already in v0) with autonomous-action governance behind a DPO gate. Real LLM provider clients (OpenAI/Anthropic/Azure OpenAI) also land here. |
| §6.5 | External Forwarding & Share Bundles | Tokenised revocable links for forwarding attachments. |
| §6.6 | Real Auth (Entra ID / OIDC) | Replaces the dev-stub auth backend. |
| §6.7 | Dashboards, Analytics & Council Reporting | Full reporting + DSA Transparency DB pipeline. |
| §6.8 | Workflow Designer UI | No-code authoring; YAML is the source. |
| §6.9 | Azure Deployment | Container Apps, managed Postgres, Key Vault, App Insights, Entra AD, VNet. |
| §6.10 | Anonymisation Script | PII retention lifecycle. |
| §6.11 | Virus Scanning (ClamAV) | Required before public intake goes live. |
| §6.12 | RBAC refinement | Real roles beyond the v0 team-membership rule. |
| §6.13 | **Formal Enforcement** (Decisions, appeals, publication) | *Added during second review round.* |
| §6.14 | **In-app Collaboration** (notifications, @mentions, subscriptions) | *Added during second review round.* |
| §6.15 | **Operational Search** (FTS, attachment text extraction) | *Added during second review round.* |
| §6.16 | **Bulk Operations & Data Migration** | *Added during second review round.* Includes legacy import for go-live. |
| §6.17 | Webhooks / integrations | Outbound events to other BIPT systems. |
| §6.18 | Additional items not yet scoped | Catch-all. |

Every sub-project entry documents: **what it adds, what it depends on, which v0 hooks enable it, and key open questions**. When you open any of these sub-projects in the future, read the §6 entry first.

## 5. Architecture (the condensed version)

For full detail see the spec. The inescapable facts:

- **Python 3.12 + FastAPI + SQLAlchemy 2 + Pydantic v2 + Alembic** for the backend.
- **React + Vite + TypeScript + shadcn/ui + Tailwind + TanStack Query + React Hook Form + Zod** for the frontend.
- **Postgres 16**, single DB, logical schema grouping by module family (`core`, `workflow`).
- **Custom YAML workflow DSL with CEL expressions** (not BPMN, not Camunda, not SpiffWorkflow). The DSL primitives: `start` / `task` / `parallel` / `sub_process` / `end` state types; `gateway` / `wait` / `spawn_and_continue` / `debounce` **reserved but not implemented**; `join: all | any | threshold:N`; triggers (event + scheduled shape-reserved); `slas:` block (parsed, firing in §6.1); `ai_prompts:` block (workflow-scoped prompts). CEL via the `celpy` library.
- **Append-only `events` table** with per-case monotonic `sequence_number`; `schema_version` column for payload evolution; `system_actor_ref` jsonb for system-generated events; `request_ip` + `user_agent` on user-originated events; `via_ai_agent_id` + `from_interaction_id` for AI-assisted human actions.
- **Per-case dispatch** via `pg_advisory_xact_lock(namespace=1, hashtextextended(case_id::text, 0))` with **deferred-event queue bounded at 256** and a `(instance_id, state_key)` cycle detector raising `ProcessLoopError`.
- **Frozen process definition snapshots** on `ProcessInstance` at creation — the engine runs against the snapshot, not the live YAML; PRs editing `processes/*.yaml` never break running cases.
- **Docker Compose for local dev** with **own Traefik** bound via Docker labels; `*.localtest.me` for hostnames (wildcard DNS → 127.0.0.1). Two optional profiles: `remote` (adds an isolated Tailscale sidecar identifying as `corpus-dev`, reachable at `corpus-dev.<tailnet>.ts.net`) and `demo` (extends `remote` with Basic Auth + `tailscale funnel` for time-boxed public exposure — TLS terminates on local daemon, no MITM).
- **AI Companion (v0)** with only the `dummy` LLMClient; real provider clients (OpenAI / Anthropic / Azure OpenAI) are locked down to §6.4 behind a **DPO-approval gate** requiring: DPA, per-agent redaction profile, lawful-basis documentation, `party.anonymized` → `AIInteraction` redaction hook, non-EU-provider ban outside `CORPUS_ENV=dev`.
- **Humans are always the actor** for state-changing events. AI is the tool used (`Event.via_ai_agent_id`). `actor_type` stays `user | system | party` — no `ai` value.
- **Complainant AI opt-out honoured case-wide**: `Party.ai_opt_out=true` (on complainant-role parties) propagates to `Case.ai_disabled`; all AI invocations return 403 + emit `ai.invocation_blocked`. Re-enablement is admin-only.
- **Regulator-grade templating**: `TemplateSet` + `TemplateVersion` (draft → in_review → approved → retired state machine, with `source_version_id` driving stale-translation detection) + `TemplateFragment` + `LetterheadProfile`. `variables_schema` JSON-Schema-validated at DSL-load time and render time. Rendered output audited (text stored in event payload ≤16KB, or pointer to an `Attachment` row for PDF letters).
- **Evidence integrity** on attachments: SHA-256 computed on upload, **re-verified on every download** (emits `attachment.content_verified` or `attachment.content_mismatch`); soft-delete only (no hard delete); `is_internal` + `category.is_shareable_default` flags exclude from future share bundles; `storage` abstraction (filesystem in dev, Azure Blob in prod).
- **Auth-backend refuse-to-boot guard**: `DevStubAuthBackend` refuses to load outside `CORPUS_ENV ∈ {dev, test}`.

## 6. The journey so far (session history)

This is the arc you missed. Milestones in rough chronological order:

### Pre-spec

1. **Discovery phase.** User described BIPT's situation: greenfield, no existing system, Complaint Team triages then forwards to domain teams (Users/DSA, Market, AI), sometimes to external authorities (Flanders/Wallonia/Brussels), AI-assisted triage desired, bidirectional email needed. The critical sentence that reframed everything was: *"I want a tool where the Complaint team could describe the workflows at the system would allow and enable that."* — this is what made CORPUS a **platform** instead of a CRM.

2. **Phasing choice.** Presented Options A (DSA-MVP-first) / B (platform-first) / C (operator-tool-only). **User chose B explicitly** despite the deadline pressure, because he has a clear mental model of multiple regulatory processes and the platform approach pays back on the 2nd+ process.

3. **Tech-stack choice.** Presented three approaches:
   - **A1 Python/FastAPI/React + custom YAML DSL** — chosen.
   - A2 Python + Camunda/SpiffWorkflow — rejected (BPMN contorts the regulator mental model).
   - A3 .NET/Blazor + custom DSL — rejected (Python's AI ecosystem pays off heavily for §6.4).

### Spec writing

4. **Section-by-section design.** Built Section 1 through Section 8 with user sign-off after each. Mid-flight course-corrections:
   - User requested **Traefik instead of nginx** — integrated into Section 1.
   - User requested **local-first on Docker Compose** instead of cloud-native Section 1 — deferred Azure to its own sub-project §6.9.
   - User flagged **attachments will be heavy** (contracts, screenshots, email exchanges) — made Attachment first-class with evidence-integrity guarantees, introduced `ObjectStore` abstraction.
   - User flagged **future attachment-sharing needs** (forward-to-authorities with download links) — added `is_internal` flag and category-based share eligibility as v0 hooks.
   - User flagged **Complaint Team as ongoing overseer** (not just intake) — added cross-portfolio dashboard, escalate + send-reminder actions, cross-team visibility rule.
   - User flagged **configurable workflows with sub-workflows** — extended DSL with `sub_process`, triggered processes, parallel fan-out/fan-in, SLAs-as-observers.
   - User flagged **AI governance concern** ("NO_AI flag") — added complainant opt-out at Party + Case level.

5. **Initial spec committed** (`02721b2`, 2026-04-17). 951 lines.

### Staging infrastructure

6. **User mentioned `geo.ovh` domain** pointing at the dev machine — triggered a discussion of remote access. Evolution:
   - First proposed: use existing production Traefik. User rejected (*"wrong decision to reuse traefik as i've some service in prod"*).
   - Proposed three isolation options: Tailscale private, own Traefik on alternative ports, SSH tunnel.
   - User raised Cloudflare Tunnel → we evaluated → rejected (MITM at Cloudflare edge conflicts with regulator data posture).
   - User asked about **Tailscale Funnel** → that's the answer (no-MITM because Funnel terminates TLS on the local daemon).
   - Discussed host-Tailscale-daemon vs. own-Tailscale-container in Compose → chose **own Tailscale sidecar (`corpus-dev` identity)** for lifecycle isolation and auditability.
   - Committed as the `remote` + `demo` Compose profiles (`04fa844`).

### AI direction

7. **AI Companion vs AI Actor discussion.** I initially pushed back on AI-first-class-in-v0 as over-scoping — proposed deferring full governance to §6.4. User pushed back: *"defer but let's have it now more as a companion"*. This yielded the current shape:
   - AI is a tool, humans invoke, AI returns text, humans act.
   - AIAgent + AIInteraction entities.
   - `ai_prompts:` block in workflow YAML for state-scoped prompts.
   - Complainant opt-out enforced case-wide.
   - Committed (`c3f61e7`).

### Review rounds

8. **First three-critic review** — three parallel agents critiqued the spec (architecture / security / pragmatism). Findings folded into spec as decisions #46–64:
   - **A3 chosen** for AI: only the `dummy` LLM client ships in v0; real providers gated behind DPO approval in §6.4.
   - Process definition snapshotting, dispatch-loop bounds + cycle detection, Event.schema_version, auth-backend refuse-to-boot guard, evidence integrity hardening, share-bundle exclusion flags, namespaced advisory locks.
   - DSL primitive restraint (B-moderate): gateway/wait/spawn_and_continue/debounce reserved; non-`all` joins kept.
   - Scope trims (C1-C6), testing trims (D1-D3).
   - §6 re-prioritized: **Timers bumped to §6.1** (EU-mandated SLAs are non-negotiable).
   - Committed (`c9b57d2`).

9. **Header refresh** (`23c3d92`, 2026-04-19). Updated "Last amended" and exec summary to reflect amendments.

10. **Second three-agent review** — three parallel agents reviewed for **missing components** (operational practice / templating & missing components / process catalog). Findings folded in as decisions #65–82:
    - New domain fields: `Case.closure_reason` / `admissibility_decision` / `priority`, `Party.is_trusted_flagger`, `User.out_of_office_until` / `delegate_user_id`, extended `CaseParticipant.role`.
    - New entities: `CaseLink`, `Inquiry`, `CaseRecusal`, `Decision` (shape only), `ConfidentialityClaim`, `FormalDocument` (shape only).
    - **Templating subsystem redesigned**: `TemplateSet` + `TemplateVersion` + `TemplateFragment` + `LetterheadProfile`, state machine, multi-language staleness detection, rendered-output audit invariant, DSL binding contract with `variables_schema`.
    - DSL `schedule_triggers:` block (shape only; firing in §6.1).
    - Four new sub-projects (§6.13 Formal Enforcement, §6.14 In-app Collaboration, §6.15 Operational Search, §6.16 Bulk Ops & Data Migration).
    - Seed now includes `telecom-consumer` + `postal` categories, `admissibility_triage.yaml` scaffold, seeded TemplateSets.
    - New events, new API endpoints, new UI pages, extended glossary.
    - Committed (`71a460c`, 2026-04-23).

## 7. Git & repo state

- **Repo:** `github.com:geofox/CORPUS`
- **Branch:** `main` (only branch); trunk-based workflow.
- **User:** Geoffrey Richard (`git config user.name`)
- **Commits so far (all `docs:`-only — no code exists yet):**

```
71a460c  docs: fold second three-agent review into foundations spec
23c3d92  docs: refresh spec header and executive summary
c9b57d2  docs: incorporate three-critic review feedback into foundations spec
c3f61e7  docs: add AI Companion (v0) + complainant AI opt-out to foundations spec
04fa844  docs: add remote-access model to foundations spec (Tailscale sidecar)
02721b2  docs: initial design spec for CORPUS foundations sub-project
```

- **`.remember/`** directory exists (from a `SessionStart:startup` hook on the host); it's gitignored-by-convention — not useful for CORPUS work, leave alone.
- **Working tree is clean** as of handover.

## 8. Infrastructure on the host

The dev machine is already running prod services that must not be touched. These are **not CORPUS** — CORPUS is greenfield:

- Host: **Sagan** (hostname `sagan-host`, user `geoffrey`).
- **Host-level Tailscale** daemon at tailnet IP `100.92.190.94`.
- **Production containers** (Docker Compose project `srv-*`): `srv-traefik-1` (Traefik v3 on :80/:443, uses `srv_traefik` Docker network and `srv_external_network`), `srv-tailscale-1` (another Tailscale identity `Sagan-Docker`, tag `tag:container`), `srv-dozzle-agent-1`, `srv-code-server-1`, `srv-ntfy-1`, `srv-uptime-kuma-1`, `srv-mediamtx-1`.
- Prod Traefik config: `/srv/traefik/traefik.yml` (uses Actalis ACME + OVH DNS challenge for `*.geo.ovh`). Middlewares in `/srv/traefik/conf/` include `basic-auth-global`, `security-headers`, etc. **Do not edit these.**
- **User's domain** `geo.ovh` — A + AAAA records for `geo.ovh` and wildcard `*.geo.ovh` point at this server. Prod Traefik serves from it.

CORPUS rules, non-negotiable:
1. CORPUS spins up its **own Traefik**, its **own Tailscale sidecar** (identity `corpus-dev`), its **own Postgres**, all in its own Compose file.
2. CORPUS binds only to its own Docker network and, for remote access, only to the Tailscale-sidecar netns — never to the public NIC or the host Tailscale.
3. A misconfig in CORPUS must not break prod.

## 9. Where the code will live (structure)

Currently empty. Planned monorepo layout (from spec §4.1):

```
/docker-compose.yml              Default dev stack (traefik + db + api + web-dev)
/docker-compose.prod.yml         Prod-like overlay
/.env.example                    Config template (includes CORPUS_ENV guard)
/backend                         FastAPI app
  /Dockerfile
  /pyproject.toml (managed by uv)
  /alembic/
  /src/corpus/
    /auth, /parties, /cases, /events, /attachments, /storage,
    /workflow, /templates, /admin, /dashboards, /reminders,
    /escalations, /ai, /api, /core
  /tests/unit /tests/integration /tests/fixtures/processes /tests/fixtures/ai_prompts
  /scripts/seed.py /scripts/reset.py
/frontend                        React SPA
  /Dockerfile
  /package.json (pnpm)
  /src/api (generated) /src/app /src/auth /src/components /src/pages
  /src/events/renderers /src/forms /src/hooks /src/lib /src/styles /src/i18n
/processes                       YAML workflow definitions (git = authoritative source)
/infra                           Reserved for future Azure IaC (Bicep/Terraform)
/infra/traefik                   Any file-provider config for local Traefik
/infra/tailscale                 Tailscale sidecar config
/docs                            Specs, plans, ADRs — you are here
/scripts                         Dev helpers
/justfile                        Optional command shortcuts (`just up`, `just test`, …)
```

## 10. Memory system

**Committed snapshot:** `docs/memory/` (in this repo — the preferred entry point for successors).
**Live canonical copy** (on the original author's machine only): `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/`.

Keep them in sync when material facts change — update `~/.claude/.../memory/` as you normally would, then copy modified files into `docs/memory/` and commit. `docs/memory/README.md` explains the dual-location convention.

When you start, read `MEMORY.md` — it's an index. The files:

- `MEMORY.md` — index, always loaded into context. Keep it short (≤200 lines).
- `user_profile.md` — Geoffrey's role, preferences, collaboration style.
- `project_corpus.md` — CORPUS state snapshot (update this after material changes).
- `reference_spec.md` — pointer to the authoritative spec with usage notes.
- `feedback_traefik.md` — Traefik over nginx preference; detailed reason.
- `feedback_local_first.md` — Docker Compose before cloud; Azure is a later spec.
- `feedback_infra_isolation.md` — never reuse prod Traefik/Tailscale; ship CORPUS-owned instances.
- `feedback_ai_consent_first.md` — AI Companion not Actor; complainant opt-out honored.
- `feedback_spec_structure.md` — specs must carry deferred + decisions log + open questions.

**When to update memory:**
- User corrects an approach → save as new `feedback_*.md`.
- User confirms a durable preference that wasn't obvious → same.
- A new sub-project spec ships → update `project_corpus.md`.
- A new major decision changes the direction → update `project_corpus.md`.

**What not to save:** anything derivable from the spec or git log.

## 11. What skills have been used and what comes next

**Used so far:**
- `superpowers:brainstorming` — entered when the user asked to build CORPUS. The skill's terminal state is "invoke writing-plans" — we are *at* that terminal state.
- `superpowers:dispatching-parallel-agents` — used twice (first critic round, second missing-components round). Each round was three general-purpose agents in parallel with distinct non-overlapping prompts.

**Not yet used, but next in sequence:**
- `superpowers:writing-plans` — this produces an implementation plan from the spec. **Invoke this when the user says go / confirms they're ready to move to implementation.**

**Likely downstream** (not yet in scope, read the skill docs when you get there):
- `superpowers:subagent-driven-development` OR `superpowers:executing-plans` — to actually execute the plan.
- `superpowers:test-driven-development` — for any code-writing step.
- `superpowers:verification-before-completion` — before reporting any implementation as done.
- `superpowers:finishing-a-development-branch` — when we have a mergeable branch.

## 12. The 35 open questions in §8

These each have a **default** recorded in the spec so planning isn't blocked. But many will need explicit user input as sub-projects ship. The ones most likely to need answers **during or just before writing-plans** (read §8 for the exhaustive list):

- Q9: `pytest-postgresql` vs `testcontainers-python` — spec default is testcontainers.
- Q10: Case reference format (`CORPUS-YYYY-NNNN`) — spec default stands.
- Q13: `just` adoption as primary interface or optional — spec default: optional with shell-script mirror.
- Q26: Closure reason taxonomy exhaustiveness — spec default stands.
- Q27: Is admissibility mandatory-first or convention-based — spec default: convention-based.
- Q35: Priority auto-promotion rules — spec default: trusted-flagger = high only.

Most others are deferrable to the sub-project where they bite. Don't force resolution on all 35 before writing-plans.

## 13. User preferences and patterns — long list

Things the user has shown consistently (in addition to what's in `user_profile.md`):

- Prefers **"option X"** response format (short list + recommendation + brief *why*).
- Answers with a letter/label only ("A3", "B-moderate", "mixed", "keep"). Treat this as a firm decision.
- Will sometimes answer **"following your rec"** — take this as explicit endorsement of the recommendation.
- Will sometimes add a clarification after choosing ("C3-keep-all and 6.3 is really important as we do have to follow sla as a regulator") — listen hard for these; they're platform-shaping.
- **Asks for critic/reviewer rounds** when the design starts to feel big. If you sense the spec is getting unwieldy, pre-emptively offer one.
- **Wants the spec to capture deferred items, decisions, open questions comprehensively.** He said so explicitly.
- When technical options are offered, he leans toward the **self-hostable / EU-residency-preserving / prod-isolation-preserving** choice.
- He references **his own domain knowledge**: "I will propose to the Council", "I am their Advisor", "SLAs are set at the European level". Take these as grounded, not speculative.
- He **does not** ask for excessive hand-holding on technical choices — just present options clearly.
- He values **honest pushback**. When I argued to defer AI-full-governance, he explicitly invited me to argue my case ("or if you think it's overkill now and should be a subproject, please argue"). Give him your real take, then respect his decision.

## 14. Task list at handover

The `TaskCreate` task list in the prior session had these items:

```
#1 [completed]  Explore project context
#2 [completed]  Ask clarifying questions (scope, users, tech, constraints)
#3 [completed]  Propose decomposition of CORPUS into sub-projects
#4 [completed]  Propose 2-3 architectural approaches for first sub-project
#5 [completed]  Present design sections and iterate to approval
#6 [completed]  Write design doc to docs/superpowers/specs/ and commit
#7 [in_progress] User reviews written spec and approves
#8 [pending]    Invoke writing-plans skill for first sub-project
```

Task #7 is effectively complete (two review rounds + all amendments folded in). When you pick up and the user signals readiness, move #7 → completed and #8 → in_progress, then invoke `superpowers:writing-plans`.

## 15. Decisions log band map (for quick navigation)

If you need to find the *why* behind any design choice:

- **§7 #1–45** — original design, including:
  - #1 Option B (platform-first)
  - #2 Python over .NET
  - #3 Custom YAML DSL over BPMN/Camunda
  - #4 CEL for expressions
  - #10 Traefik over nginx
  - #20 SLAs as observers (emit events on breach; don't drive flow)
  - #22 Complaint Team sees everything (membership-based rule)
  - #40 AI as Companion (humans invoke; not actor)
  - #41 Dedicated AIAgent entity (not User.is_ai)
  - #42 Human is always actor; AI is tool
  - #45 Complainant AI opt-out default
- **§7 #46–64** — first critic review (architecture / security / pragmatism):
  - #46 A3 — dummy-only LLM in v0
  - #47 Process definition snapshots
  - #48 Dispatch-loop bounds + cycle detection
  - #49 Event schema_version
  - #50 Auth-backend refuse-to-boot guard
  - #51 Evidence integrity hardening
  - #52 Share-bundle exclusion flags
  - #53 Namespaced advisory locks
  - #54 DSL primitive restraint (B-moderate)
  - #57 §6.1 is Timers (bumped)
- **§7 #65–82** — second missing-components review:
  - #65 Closure taxonomy
  - #66 Admissibility gate
  - #67 Case.priority with urgent value
  - #68 CaseLink + merge/split
  - #69 Inquiry entity
  - #70 Trusted-flagger support
  - #71 CaseRecusal + OOO + delegate
  - #72 Templating subsystem redesign
  - #74 Decision entity
  - #75 ConfidentialityClaim entity
  - #76 FormalDocument entity
  - #77 schedule_triggers DSL block
  - #78 Four new §6 sub-projects

## 16. Things the spec does NOT commit to (important)

So you don't accidentally assume them:

- **No Entra ID wiring in v0** — dev stub auth only, guarded by `CORPUS_ENV`.
- **No real LLM provider clients in v0** — only `dummy`.
- **No SLA firing in v0** — parsed and stored, not executed.
- **No timer firing / background jobs in v0** — APScheduler or Celery lands with §6.1.
- **No email I/O in v0** — reminders log events only.
- **No public intake form in v0** — staff-entered cases only.
- **No share bundles in v0** — flags in place (`is_internal`, `is_shareable_default`); bundles land with §6.5.
- **No full decision workflow in v0** — entity shape exists; workflow in §6.13.
- **No PDF rendering in v0** — `LetterheadProfile` entity exists; WeasyPrint pipeline lands with §6.2.
- **No cross-case search in v0** — §6.15.
- **No in-app notifications** — §6.14.
- **No Azure deployment** — §6.9; v0 targets Docker Compose on a Linux host.
- **No real RBAC** — team-membership rule only in v0; §6.12.
- **No scheduled-trigger firing** — shape reserved in DSL; fires with §6.1.

## 17. What to do next (decision tree)

Most likely first user message to you:

- **"Go ahead with writing-plans"** or equivalent → invoke `superpowers:writing-plans`. The plan is written from the spec; expect it to take several interactions.
- **"Make a small change to the spec"** → read the spec, edit, self-review, commit, push, update memory if material.
- **"Launch critics again"** → dispatch parallel agents via `superpowers:dispatching-parallel-agents`. Use distinct angles that don't overlap with the two prior rounds.
- **"Explain X"** → the spec is likely authoritative. Read it first.
- **"I've been thinking about…"** → listen carefully; user course-corrections mid-flight are a signature pattern. Apply as an amendment if load-bearing.

**Do not:**
- Start writing code without a plan.
- Reverse a committed decision without explicit user approval.
- Touch production services on the host (`srv-traefik-1`, `srv-tailscale-1`, etc.).
- Re-review the spec for the sake of it — two critic rounds are enough unless the user asks.
- Assume scope you don't see; when in doubt, ask.

## 17.5. v0 "build" vs "shape-only" vs "deferred" map

This is the clearest way to know what the spec actually *commits* v0 to build. Entities/features fall into three buckets:

**Build in v0 (fully implemented):**
- Case, Party, CaseParticipant, CaseCategory, Event, Attachment, AttachmentCategory, User, Team, ProcessInstance, Task, CaseLink (with merge/split), Inquiry, CaseRecusal.
- Full templating subsystem: TemplateSet, TemplateVersion (with state machine), TemplateFragment. Jinja2 rendering for plain text.
- Engine: start / task / parallel (join=all/any/threshold:N) / sub_process (spawn_and_wait) / end; triggers (event-type); dispatch loop with advisory lock + deferred-event queue + cycle detection; frozen process definition snapshots.
- AI Companion: AIAgent + AIInteraction (dummy-only), drawer UI, workflow-scoped `ai_prompts`, opt-out enforcement, `ai.*` events.
- Escalate + Send-Reminder (logs events; actual email in §6.2).
- Dashboard with 8 widgets (Complaint Team scoped).
- Auth: dev-stub with CORPUS_ENV guard.
- Storage: FilesystemObjectStore + InMemoryObjectStore; uploads with SHA-256 + content-sniff + soft-delete.
- API (~70 endpoints across all domains), RFC 7807 errors, cursor pagination, OpenAPI + generated TS client.
- Tests: engine scenario (fixture-driven), DB invariants, API contract, Vitest unit, Playwright e2e happy path, axe-playwright on 3 pages.
- Compose with own Traefik, dev profile + `remote` profile (Tailscale sidecar) + `demo` profile (Basic Auth + Funnel).

**Shape-only in v0 (entity/schema exists, workflow/runtime deferred):**
- `Decision`, `ConfidentialityClaim`, `FormalDocument` — entities + basic CRUD, no workflow integration.
- `LetterheadProfile` — entity, no PDF rendering pipeline (WeasyPrint in §6.2).
- `schedule_triggers:` DSL block — parsed and validated, no firing (§6.1).
- `slas:` block — parsed and validated, no firing (§6.1).
- DSL reserved keywords: `gateway`, `wait` state types; `spawn_and_continue` sub_process mode; `debounce` on triggers. Parser recognises them and raises `NotImplementedError` on instance creation.
- Real LLM clients (`OpenAIClient`, `AnthropicClient`, `AzureOpenAIClient`) — protocol + stubs; real implementations in §6.4.

**Deferred entirely (not in v0 at all):**
- Timer firing, SLA execution (§6.1).
- Email I/O, PDF letterhead rendering (§6.2).
- Public intake form (§6.3).
- AI proposals + review flow + real provider clients (§6.4).
- Share bundles with tokenised links (§6.5).
- Entra ID authentication (§6.6).
- Reporting / Council dashboards / DSA Transparency DB (§6.7 + §6.13).
- Workflow Designer UI (§6.8).
- Azure deployment (§6.9).
- Anonymisation script (§6.10).
- Virus scanning (§6.11).
- RBAC refinement (§6.12).
- Decision workflow + appeals + publication (§6.13).
- In-app notifications + @mentions (§6.14).
- Cross-case search + attachment extraction (§6.15).
- Bulk ops + data migration tooling (§6.16).
- Webhooks (§6.17).

## 17.6. Consolidated tooling list

Python side:
- `python:3.12-slim` base.
- `uv` for dependency management (not poetry, not pip-tools).
- `fastapi` + `uvicorn` + `pydantic>=2` + `sqlalchemy>=2` + `alembic`.
- `celpy` for CEL expression evaluation.
- `jinja2` for template rendering (sandboxed, autoescaped).
- `python-magic` for content-type sniffing.
- `structlog` for JSON logs to stdout.
- `ruff` for lint + format (replaces black/isort/flake8).
- `mypy` in strict mode on core modules.
- `pytest` + `pytest-asyncio` + `testcontainers-python` (default choice for DB-in-test — see Q9).
- `httpx.AsyncClient` for API contract tests.

Frontend side:
- Vite + React 19 + TypeScript strict.
- `pnpm` for package management (not npm, not yarn).
- shadcn/ui (copied into `/frontend/src/components`) + Tailwind.
- TanStack Query, React Router, React Hook Form, Zod.
- Lucide icons, fontsource for fonts (no Google Fonts).
- `i18next` wired (EN only for v0).
- `openapi-typescript-codegen` (or `openapi-typescript`) for client type generation; generated output committed to git.
- Vitest for unit tests, Playwright + axe-playwright for e2e + a11y.
- ESLint + Prettier (could swap for Biome per Q14).

Infra:
- Docker + Docker Compose (host has both already).
- Traefik v3 (own instance for CORPUS — not `srv-traefik-1`).
- Tailscale sidecar (own identity `corpus-dev` — not host Tailscale, not `srv-tailscale-1`).
- Postgres 16 (`postgres:16-alpine`).
- `static-web-server` (Rust, tiny image, SPA fallback) for prod frontend.
- `adminer` (optional `tools` profile) for DB inspection.
- GitHub Actions for CI.
- `just` (optional) for command shortcuts; shell scripts in `/scripts/` as fallback.
- `pre-commit` for local hooks (ruff, mypy staged, eslint, prettier, check-large-files, no-commit-to-main, gitleaks).

## 17.7. Where the implementation plan will live

When `superpowers:writing-plans` is invoked, it will produce a plan document. Convention (from the skill):

- Path: `docs/superpowers/plans/YYYY-MM-DD-corpus-foundations-plan.md`
- Committed to git in its own commit.
- Review-checkpointed: the skill's flow is to split work into stages and review after each.

After the plan is written, `superpowers:executing-plans` or `superpowers:subagent-driven-development` picks up from there.

## 17.8. How to iterate with Geoffrey (the collaboration pattern)

Based on 6 days of interaction, this is the pattern that worked:

1. **Understand the ask.** Read the message carefully; he often packs multiple concerns into one sentence.
2. **Offer options.** When multiple paths are viable, present 2–4 labelled options (A/B/C or variant names) with one-sentence *why* each + a recommendation.
3. **Justify the recommendation.** Briefly — one or two sentences of reasoning, not a paragraph.
4. **Accept the decision.** He'll answer with a label ("C2-mixed", "keep", "following your rec"). Treat as firm.
5. **Apply.** Make the edits, update memory if durable, commit with a descriptive message, push.
6. **Acknowledge briefly.** One-line confirmation + what's next. Don't over-summarise.
7. **If scope is drifting:** offer to stop and review; he often appreciates it. The two critic rounds we ran both started from me sensing the spec was getting big.

Do **not**:
- Re-explain choices he's already approved.
- Expand scope unilaterally — ask first.
- Skip the commit-and-push step after material amendments.
- Forget to update memory when you learn a new preference.

## 18. Quick-reference facts

| Fact | Value |
|---|---|
| Working dir | `/home/geoffrey/CORPUS` |
| Spec path | `docs/superpowers/specs/2026-04-17-corpus-foundations-design.md` |
| Memory (committed) | `docs/memory/` (authoritative for successors) |
| Memory (live) | `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/` (author's machine only) |
| Git remote | `git@github.com:geofox/CORPUS.git` |
| Main branch | `main` |
| User | Geoffrey Richard, BIPT |
| User's email | `t8hfy846km@privaterelay.appleid.com` (redacted relay; don't rely on this being stable) |
| Domain | `geo.ovh` (wildcard A/AAAA points at this server) |
| Host hostname | `sagan-host` |
| Host tailnet IP | `100.92.190.94` |
| Prod Traefik container | `srv-traefik-1` — **do not touch** |
| Prod Tailscale container | `srv-tailscale-1` (hostname `Sagan-Docker`, tag `tag:container`) — **do not touch** |
| Expected CORPUS tailnet identity | `corpus-dev` (new, your own sidecar) |
| Expected dev hostnames | `corpus.localtest.me` (default), `corpus-dev.<tailnet>.ts.net` (remote profile) |
| Azure region target | Belgium Central |
| LLM in v0 | `dummy` only |
| Scale | 100s complaints/year |
| Languages (templates) | NL, FR, EN (DE later) |
| Staff UI language | EN only (i18next wired) |
| Accessibility target | WCAG 2.1 AA / AnySurfer |

---

## 19. Appendix — conversation artifacts not in the spec

A few things discussed but not committed to the spec, kept here for fidelity:

- **Verbatim user phrasing worth preserving:**
  - *"I want a tool where the Complaint team could describe the workflows at the system would allow and enable that"* — the sentence that made CORPUS a platform.
  - *"wrong decision to reuse traefik as i've some service in prod"* — source of the infra-isolation principle.
  - *"defer but let's have it now more as a companion"* — source of the AI Companion v0 shape.
  - *"I'd like to have it (or if you think it's overkill now and should be a subproject, please argue)"* — shows he welcomes honest pushback.
  - *"SLAs are usually set at the european level"* — source of §6.1 Timers being bumped first.
  - *"admin could add category later"* — source of AttachmentCategory being a reference table, not an enum.
- **Rejected alternatives worth remembering** (so we don't re-open them without reason):
  - **nginx** (user rejected in favour of Traefik).
  - **Cloudflare Tunnel** (rejected because TLS termination at CF = MITM over complainant data).
  - **Reusing production Traefik** (rejected — prod isolation).
  - **Reusing production Tailscale container** (rejected — coupling to prod Compose).
  - **Host-level Tailscale daemon** (rejected in favour of own sidecar — audit clarity and lifecycle).
  - **Camunda 8 / SpiffWorkflow / BPMN** (rejected in approach comparison).
  - **.NET + Blazor** (rejected in approach comparison).
  - **Full AI-actor governance in v0** (deferred to §6.4 after arguing both sides).
- **Named regulatory touch points** user has mentioned or implied that should be top-of-mind when relevant:
  - DSA Art. 22 (trusted flaggers) — integrated via `Party.is_trusted_flagger`.
  - DSA Art. 24 (transparency DB + RFI to platforms) — deferred §6.13.
  - AI Act Art. 70 (RFI) — planned for §6.1.
  - AnySurfer (Belgian federal accessibility label, WCAG AA-equivalent) — a11y gate.
  - BEREC (telecom regulators network) — context for future cross-border coordination.
  - EBDS / Digital Services Coordinator Board — DSA coordination.
  - Market Court (Belgian appellate body) — appeals flow in §6.13.
  - APD/GBA (Belgian DPA) — likely redirect target for GDPR-only complaints.
  - Ombudsman for Telecommunications — interplay with BIPT's consumer mandate.

---

*End of handover. Good luck — this project is a good one.*
