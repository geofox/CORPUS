# CORPUS memory snapshot

These files are a **committed snapshot** of the Claude Code session-scoped memory maintained at `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/` on the original author's machine.

They are point-in-time artifacts of the brainstorming and spec-writing phase — user profile, project-state summary, and durable user preferences recorded as `feedback_*.md` files. They're here so a successor taking over the project (human or another Claude session) can read the same context without needing access to the original author's `~/.claude/` directory.

## Files

- **`MEMORY.md`** — index of the other files.
- **`user_profile.md`** — how Geoffrey works; collaboration style; role.
- **`project_corpus.md`** — project-state snapshot as of 2026-04-23.
- **`reference_spec.md`** — pointer to the authoritative spec with usage notes.
- **`feedback_traefik.md`** — Traefik over nginx preference.
- **`feedback_local_first.md`** — Docker Compose before cloud; Azure later.
- **`feedback_infra_isolation.md`** — never reuse prod Traefik/Tailscale; ship CORPUS-owned instances.
- **`feedback_ai_consent_first.md`** — AI Companion not Actor; complainant opt-out honored.
- **`feedback_spec_structure.md`** — specs must carry deferred + decisions log + open questions.

## Two canonical locations

When the *original author's* Claude session is running, the live canonical copy of these memories is at `/home/geoffrey/.claude/projects/-home-geoffrey-CORPUS/memory/`. Claude updates that copy automatically as new preferences and project-state facts are learned.

For a successor session on a different machine — or reviewing via GitHub — this committed copy under `docs/memory/` is authoritative.

**Keep them in sync** when material changes happen: after updating `~/.claude/.../memory/`, copy the modified files into `docs/memory/` and commit. Alternatively, treat `docs/memory/` as the canonical location going forward and mirror back to `~/.claude/`.

## Format

Every memory file (except `MEMORY.md` itself) uses YAML frontmatter:

```markdown
---
name: <short title>
description: <one-line relevance hint>
type: user | feedback | project | reference
---

<memory body>
```

`type` values:
- `user` — facts about the user (role, preferences, domain).
- `feedback` — durable "do this / don't do that" guidance from corrections or validated approaches.
- `project` — project-state facts (current phase, decisions, etc.).
- `reference` — pointers to authoritative resources.

Don't write ephemeral task-state or debug findings here. Don't duplicate spec content. Do record "why" for feedback items.
