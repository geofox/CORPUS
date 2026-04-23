---
name: AI consent-first stance for CORPUS
description: Complainants must be able to opt out of AI processing; AI is a companion tool (no autonomous action) until the workflow-actor model is formally approved
type: feedback
originSessionId: 9af3a2ba-57a5-49cf-9926-8d8dd78bc8d3
---
For CORPUS, AI is treated with a consent-first, human-in-the-loop-first stance by default.

**Two durable rules:**

1. **AI is a Companion, not an Actor.** AI returns text that humans choose to use. AI never directly mutates case state, never sends external communications, never drives workflow transitions. Attribution on state-changing events is always the human; AI assistance is additive metadata (`Event.via_ai_agent_id`, `Event.from_interaction_id`).

2. **Complainants may opt out of AI processing for their own complaints.** `Party.ai_opt_out` (on complainant-role parties) propagates to `Case.ai_disabled`; no AI invocation is permitted on opted-out cases (API returns 403, audit event emitted). Re-enablement is admin-only and requires clearing the Party opt-out first.

**Why:** Geoffrey (as BIPT's Advisor) is proposing to the BIPT Council that complainants have the right to refuse AI processing of their complaints. This hasn't been formally adopted by the Council yet, but he wants the platform to support it from day one — "If we receive a complaint with the complainant opting out, we would need to deactivate the tools."

**How to apply:**
- Don't propose AI features that bypass these rules. If a feature would require AI to act autonomously, propose it as part of §6.4 (AI-as-workflow-actor sub-project) with the necessary guardrails.
- Every new AI capability or prompt must respect `Case.ai_disabled` — the check is centralized; don't reinvent it.
- When BIPT Council formally decides the opt-out policy (opt-out vs opt-in vs category-specific), be prepared to adjust the default via config rather than re-architect.
- If a workflow needs to branch on opt-out status ("if AI disabled, skip this AI state"), that's a DSL feature we'll add alongside the workflow-actor sub-project.
