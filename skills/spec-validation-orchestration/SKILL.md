---
name: spec-validation-orchestration
description: When and how to call spec.validate on a written specification. What to concatenate for an OpenSpec change, the same-repository trend rule, timeout recovery, the testability floor, and how the spec lens routes against architect.validate and design.validate. spec.validate is single-pass in v1. No certify, no consensus, no from-repo.
---

# Spec Validation Orchestration

This skill is for operators who run `spec.validate` on a written specification
before anyone builds from it. `spec.validate` is the what-to-build lens of the
validator trio: it grades a **spec artefact** (proposal, design doc, task
breakdown, or an OpenSpec-style change bundle) against the **8 spec-quality
laws**; upstream of `architect.validate` (built agentic code) and
`design.validate` (rendered surfaces). Doctrine applied at the spec costs a
sentence; the same finding at review time costs a rebuild.

## §1 · What to submit · the OpenSpec concatenation rule

Send the FULL spec text verbatim as `implementation_context`. For an
OpenSpec-style change, concatenate in this order:

```
proposal.md + design.md + tasks.md + delta specs
```

No truncation and no `…` placeholders; the reviewer cites specific
requirements, decisions, and tasks, and treats placeholders as literal
content. If the bundle is very large, split into multiple calls scoped by
document (each call consumes a metered run).

Wrong-input routing is designed, not punished: source code or UI artefacts
return `spec_classification=non_spec` with `tier=not_applicable`, NOT a
failing grade. Submit those to `architect.validate` or `design.validate`
instead of spending a spec run.

## §2 · The iteration loop · same-repository trend grouping

Pass the **same `repository` value** across every call on the same spec.
Rounds group into one trend under the `'spec'` dimension in your
validation-history dashboard and in `me.validation_history(repository=...)`.
Round numbering belongs in the `task` field, never in `repository`; a
changed string starts a fresh, unrelated trend.

**What chaining does NOT do in v1**: `spec.validate` does not inject a prior
run as an LLM baseline. Every round is scored fresh; the trend is a
dashboard/history view, not an anchor. (This matches `design.validate`;
baseline-anchor injection is an `architect.validate`-only feature.)

### Iteration round shape

```
Round 1: spec.validate(spec_v1, repository="<stable-name>") → grade_1
↓
Fix the production_blockers cited (testability first; it is the floor).
↓
Round 2: spec.validate(spec_v2, repository="<same-stable-name>") → grade_2
↓
Round N: … → grade_N (fresh scoring each round; one trend in the dashboard)
```

### Recovery on timeout

`spec.validate` is a long single-pass LLM call (~60-180s) and MCP clients
often close the call first. **Do not retry**: the run completes and persists
server-side. The `run_id` arrives in the first `notifications/progress` event
at t=0s; capture it, then call `me.validation_history(run_id="<that-id>")`.
If the transport drops before the first event, recover with
`me.validation_history(repository="<same value>")`. Recovery is unavailable
when `private_session=true`: no run is stored, so there is nothing to
recover.

## §3 · The testability floor and the headline grade

The 8 spec-quality laws are scored per-law, but the headline grade is driven
by `severity_class`:

- `production_blocker` = a spec floor is breached in a way that ships a wrong
  or ungoverned build: a load-bearing requirement with no observable
  acceptance signal, an irreversible or externally visible step with no named
  human gate, or a downstream role that cannot start without guessing.
- A high grade means "the spec floors hold", not "the prose is polished".
  Checkability is non-negotiable; prose elegance is never penalised.

When a round surfaces a `production_blocker`, fix it before anyone builds;
that is the entire point of running the lens upstream.

Calibration note (v1): the scoring prompt is a first cut, not yet corpus-tuned
the way architect.validate is (the tool's own description discloses this).
Treat the grade as a directional quality signal, not a certified verdict.

## §4 · Which lens · routing the trio

Three lenses, one scorer, separate weekly quota buckets:

| The artefact under review is… | Call | Rubric |
|---|---|---|
| A written spec (proposal, requirements, tasks, OpenSpec change) | `spec.validate` | 8 spec-quality laws |
| Agent code (autonomy, trust boundaries, orchestration) | `architect.validate` | 10 agentic principles |
| A frontend surface (component, screen, flow) | `design.validate` | 8 experience-design laws |

The governed sequence for a new piece of work is spec → build → validate the
build: `spec.validate` on the change before implementation, then
`architect.validate` and/or `design.validate` on what was built. Attach all
of them to one Governed Session (`session_id`) and the whole arc reads as one
timeline in `me.sessions`.

## §5 · Single-pass in v1: no decision tree

Like `design.validate`: no `spec.certify`, no consensus mode, no from-repo
scan, no Tasks augmentation. The loop is `spec.validate` → fix → re-run.
Certify/consensus flows are `architect.validate`-only and do not accept spec
runs.

## Quick reference (when you only have 60 seconds)

| Question | Answer |
|---|---|
| What do I send for an OpenSpec change? | proposal.md + design.md + tasks.md + delta specs, concatenated verbatim. |
| What groups rounds together? | Same `repository` string; a dashboard trend, not an LLM anchor (no baseline injection in v1). |
| My call timed out; do I retry? | No. Capture the `run_id` from the first progress event; recover via `me.validation_history(run_id=...)`. |
| I sent code by mistake? | It returns `not_applicable`, not a fail; but it spent a run. Route code to architect.validate. |
| What drives the grade? | `production_blocker` findings: unverifiable load-bearing requirements and ungated irreversible steps. |
| Is there a spec.certify or consensus? | Not in v1. Single-pass only. |
| Does spec.validate share quota with the other lenses? | No. Its own weekly bucket. |
