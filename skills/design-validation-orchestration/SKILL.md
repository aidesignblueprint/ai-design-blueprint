---
name: design-validation-orchestration
description: When and how to call design.validate on a UI surface. The same-repository trend rule, timeout recovery, the accessibility floor, and how the surface lens differs from architect.validate. design.validate is single-pass in v1. No certify, no consensus, no from-repo.
---

# Design Validation Orchestration

This skill is for operators who call `design.validate` more than once on the
same UI surface. `design.validate` is the surface-craft twin of
`architect.validate`: it grades a **frontend artefact** (component, screen,
flow) against the **8 experience-design laws** instead of agent code against the
10 agentic principles. One call is easy; the iteration loop has the same
non-obvious same-repository rule that severs the lineage if you miss it.

## §1 · The iteration loop · same-repository trend grouping

**The rule**: pass the **same `repository` value** across every call in the
iteration. Rounds group into one trend under the `'surface'` dimension in
your validation-history dashboard and in
`me.validation_history(repository=...)`. Any change to the `repository`
string starts a fresh, unrelated trend. An `iter-2`-suffix repository name
looks like good provenance but is exactly the edit that severs the trend.
Round numbering belongs in the `task` field or in commit messages, never in
`repository`.

**What chaining does NOT do in v1**: `design.validate` does not inject a
prior run as an LLM baseline. Every round is scored fresh; the trend is a
dashboard/history view, not an anchor. (Baseline-anchor injection is an
`architect.validate`-only feature.)

### Iteration round shape

```
Round 1: design.validate(surface_v1, repository="<stable-name>") → grade_1
↓
Fix the production_blockers cited (accessibility first; it is the floor).
↓
Round 2: design.validate(surface_v2, repository="<same-stable-name>") → grade_2
↓
Round N: … → grade_N (fresh scoring each round; one trend in the dashboard)
```

`repository="<same-stable-name>"` is the load-bearing literal.

### Recovery on timeout

`design.validate` is a long-running LLM call and your MCP client often closes
the call before the server returns. **Do not retry**: a retry re-runs the whole
call. The `run_id` arrives in the first `notifications/progress` event at t=0s;
capture it, then on timeout call `me.validation_history(run_id="<that-id>")` to
fetch the persisted result. If the transport drops before that first event
(sub-second window), recover with `me.validation_history(repository="<same value>")`.
Recovery is unavailable when `private_session=true`: no run is stored, so
there is nothing to recover.

## §2 · There is no decision tree: design.validate is single-pass in v1

Architecture has three tools (validate / validate_consensus / certify). Design
has **one**. In v1 there is deliberately:

- **No `design.certify`.** A surface review does not mint a certified badge.
- **No `design.validate_consensus`.** No N-run variance band yet.
- **No from-repo scan and no Tasks augmentation.** Sync single-pass only.

So the loop is just: `design.validate` → fix → `design.validate` →
repeat. Do not attempt certify/consensus flows on a surface run; those tools do
not accept surface runs.

## §3 · The accessibility floor and the headline grade

The 8 experience-design laws are scored per-law, but the headline grade is
driven by `severity_class`, not raw verdict counts:

- `production_blocker` = an experience floor is breached in a way that ships a
  broken or inaccessible surface. The headline grade penalises **only**
  `production_blocker` findings (and the legacy `high_risk` verdict).
- A high grade therefore means "the experience floors hold", not "the surface is
  polished". Accessibility is non-negotiable and is the most common
  `production_blocker`; aesthetic polish is not penalised.

When a round surfaces a `production_blocker`, address it before moving on;
silent shipping of a breached floor is exactly what the loop exists to catch.

## §4 · Which lens · design.validate vs architect.validate

Two lenses, one validator, separate weekly quota buckets:

| The artefact under review is… | Call | Rubric |
|---|---|---|
| A frontend surface (component, screen, flow) | `design.validate` | 8 experience-design laws |
| Agent code (autonomy, trust boundaries, orchestration) | `architect.validate` | 10 agentic principles |

They do not share a quota, and a run of one does not anchor a run of the other
(the `scope_signature` and validator differ). Pick the lens that matches the
artefact; if you are shipping both a UI and the agent behind it, run both.

## Quick reference (when you only have 60 seconds)

| Question | Answer |
|---|---|
| What chains rounds together? | Same `repository` string; a dashboard trend, not an LLM anchor (no baseline injection in v1). |
| My call timed out; do I retry? | No. Capture the `run_id` from the first progress event; recover via `me.validation_history(run_id=...)`. |
| Is there a design.certify or consensus? | Not in v1. design.validate is single-pass. |
| What drives the grade? | `production_blocker` findings (breached experience floors, most often accessibility); not aesthetic polish. |
| Does design.validate share quota with architect.validate? | No. Separate weekly buckets. |
| Why is round 1 missing from the trend? | You changed `repository`, OR you passed `private_session=true` on round 1 (the stored run is gated, so nothing was recorded). |
