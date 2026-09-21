---
name: team-agents-orchestration
description: How to run the governed cross-functional trio (PM / Engineer / Designer role lenses) over one Governed Session with team_agents enabled. The co-planning gate, typed handoff events, steer-anytime, hard gates at irreversible actions, and when NOT to use team mode (~15x token cost; breadth-first work only).
---

# Team Agents Orchestration

This skill is for operators composing the three AI Design Blueprint role
lenses; PM (`spec-pm` wielding `spec.validate`), Engineer (`spec-engineer`
wielding `architect.validate`), Designer (`spec-designer` wielding
`design.validate`); into a governed team over ONE Governed Session. Team
mode is additive topology: the standalone tools are untouched, and a session
without `team_agents` behaves exactly as before.

## §1 · Task-shape honesty · when NOT to use team mode

Team mode helps **breadth-first** work: three lenses on one artefact (a spec
to govern, code to build, a surface to render). It hurts tightly-coupled
edits; expect roughly **15x the tokens** of a single lens. Default OFF.
If the work is one file, one lens, or one quick fix: use the standalone
validator and skip the team entirely.

## §2 · Setup

1. Create a Governed Session in the web app (`/app/sessions`), optionally
   anchored to a repo and an OpenSpec change ref.
2. Enable **team mode** on the session page; the app shows the cost/fit
   disclosure before it flips.
3. In your harness, the three role definitions from the claude-code pack
   (`claude-code/agents/spec-pm.md`, `spec-engineer.md`, `spec-designer.md`)
   become addressable teammates. Give every lens the session's `session_id`.

## §3 · The collaboration contract

1. **One upfront co-planning gate.** The PM lens drafts the governed spec
   and posts a `plan_preview` event; the human previews and edits the spec
   BEFORE execution. Nothing builds until the human approves the plan.
2. **Handoffs are visible, not blocking.** Every role transition posts a
   `handoff` event via `me.session_event` (who → who, what was passed).
   Disagreements post `pushback`. These are audit events; work continues.
3. **Steer anytime.** The human can address any single lens mid-run; the
   steered lens posts a `steer` event and adjusts. Every lens re-reads the
   operator's latest instruction before spending its next validator run.
4. **Hard gates ONLY at irreversible side-effects.** Repo writes, external
   calls, publishing: the acting lens posts a `gate` event and WAITS for
   the human. Everything else flows.
5. **Optional strict mode**: teams that want per-handoff approval treat
   every `handoff` as a gate; a dial the operator sets in their harness
   instructions, not a different system.

## §4 · The governed sequence

```
PM: author spec → spec.validate (session_id) → plan_preview event → HUMAN GATE
↓ handoff event
Engineer: build → architect.validate (session_id) → handoff/pushback events
Designer: build surface → design.validate (session_id) → handoff/pushback events
↓
Trio readiness: the session page shows all three verdicts side by side
```

The session timeline interleaves validation runs and team events into one
inspectable system view; decisions and transitions, never a chat log. Post
events at transitions only.

## §5 · Recovery + discipline

- Validator timeout: never retry; recover via
  `me.validation_history(run_id=...)` (the run_id arrives in the first
  progress event).
- Posting an event to a session with team mode OFF is refused; enable the
  flag in the web app first.
- Quotas: each lens spends its OWN weekly bucket; team mode multiplies
  spend across three buckets. The weekly quota remains the hard cap.

## Quick reference

| Question | Answer |
|---|---|
| When team mode? | Breadth-first, three lenses on one artefact. Otherwise standalone. |
| Cost? | ~15x a single lens. Disclosed before the flag flips. |
| What blocks? | Only the co-planning gate and `gate` events at irreversible actions. |
| Handoffs? | Visible `handoff` events, never blocking approvals (strict mode optional). |
| Steering? | Address any lens directly; it posts `steer` and adjusts. |
| Where do I watch? | The session page: timeline (runs + events) + trio readiness. |
