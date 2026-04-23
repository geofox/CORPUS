---
name: Specs must preserve deferred items, decisions, and open questions
description: Spec documents for CORPUS should include detailed deferred/future sub-project sections, a full decisions log with rationale, and an open questions section with defaults
type: feedback
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
When writing spec documents for CORPUS, **always include** three sections in addition to the core design:

1. **Deferred / future sub-projects** — organized catalog of what's been flagged but not in this sub-project's scope. For each: what it adds, dependencies on earlier sub-projects, which v0 hooks already enable it, key open questions.
2. **Decisions log** — every meaningful architectural or tooling choice with a one-line *why*. Aim for exhaustiveness so future-us can judge edge cases without re-litigating.
3. **Open questions** — items to resolve later, each with a reasonable *default until resolved* so planning isn't blocked.

**Why:** Geoffrey explicitly asked ("In the spec docs, will you also write what is deferred, in details, and remember what we talked about?"). The concern is that brainstorming context gets lost between sessions, leaving only half-formed artifacts. Capturing these three dimensions in the spec makes the document self-sufficient months later.

**How to apply:**
- On every new CORPUS sub-project spec, these three sections are mandatory, not optional.
- Deferred entries should map one-to-one with identifiable future sub-projects so the roadmap stays legible.
- Decisions log entries should use the pattern "X chosen over Y because Z" — include alternatives considered.
- Open questions should always provide a default.
