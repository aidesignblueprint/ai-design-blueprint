---
name: architect-validation-orchestration
description: When and how to call architect.validate, architect.validate_consensus, and architect.certify in sequence. The same-repository chaining rule, the decision tree, and the validate → consensus → certify pattern.
---

# Architect Validation Orchestration

This skill is for operators who call the AI Design Blueprint architect tools more than once on the same code. One call is easy; the canonical loop is two-tool + three-step and has one non-obvious rule that breaks the loop silently if you miss it.

## §1: The iteration loop · same-repository chaining

Every call to `architect.validate` looks up the most recent prior run on `(your user_id, the repository field, scope_signature)` and injects it as `prior_run_baseline` into the next LLM call. Same code with a prior baseline can score 60/C anchored vs 98/A unanchored; anchoring is how the architect tracks improvement across iteration rounds.

**The rule**: pass the **same `repository` value** across every call in the iteration. The validator chains on exact-string match. Any change to the `repository` string starts a fresh arc; the next call sees no prior, runs unanchored, the iteration lineage is silently severed.

Concrete failure mode: an `iter-1`-suffix vs `iter-2`-suffix repository name produces two separate repository rows in the user's validation history, two disconnected scores, no delta surface; even when the bytes between the two calls are identical. Verified empirically on the 2026-05-22 dispatch that produced this skill (§3 below).

### Iteration round shape

```
Round 1: architect.validate(code_v1, repository="<stable-name>") → score_1
↓
Fix the production_blockers cited.
↓
Round 2: architect.validate(code_v2, repository="<same-stable-name>") → score_2 anchored to round 1
↓
Fix the remaining blockers.
↓
Round N: architect.validate(code_vN, repository="<same-stable-name>") → score_N anchored to round N-1
```

`repository="<same-stable-name>"` is the load-bearing literal.

### Recovery on timeout

MCP clients often close the call before the server returns. Don't retry; the run_id is in the first `notifications/progress` event at t=0s. Capture it; on timeout call `me.validation_history(run_id="<that-id>")` to fetch the persisted result. Server-side budget is 6 minutes. Edge case: if the transport drops before the first progress event (sub-second), recover with `me.validation_history(repository="<same value>")` to find your most recent run.

> For Tasks-aware clients (MCP 2025-11-25 · SEP-1686), the `taskId` returned from a task-augmented call is the same UUID as `run_id`; so `me.validation_history(run_id=<taskId>)` and `tasks/result(taskId)` route to the same persisted row. See the next subsection for the canonical Tasks flow.

### Task-augmented invocation (MCP 2025-11-25, SEP-1686)

If your client advertises the `tasks` capability in its `initialize` response, you can task-augment any `architect.validate` call by including `task: {ttl: <ms>}` in the request params. The server returns a `CreateTaskResult` immediately (with `taskId == run_id`) and runs the validation in the background; no more long-running synchronous calls. Spec-correct long-running pattern:

```
Client: architect.validate(implementation_context=..., task: {ttl: 360000})
Server: returns CreateTaskResult(taskId=<uuid>, status="working", pollInterval=500) immediately
Client: tasks/get(taskId) → poll for status updates (or listen for notifications/tasks/status push)
        when status="completed" → tasks/result(taskId) → fetch the validate response envelope
```

Behaviors:
- `_meta.progressToken` from the original request stays valid for the entire task lifetime; existing `notifications/progress` events keep firing alongside the new `notifications/tasks/status` events
- The `taskId` equals the `run_id`, so `me.validation_history(run_id=<taskId>)` continues to work as the canonical recovery path for clients that haven't migrated to `tasks/result` yet
- `tasks/cancel(taskId)` cleanly cancels a running validation (reuses the existing `cancel_requested` flag; the worker checks at phase boundaries)
- Sync (non-augmented) calls behave exactly as before; backwards-compatible by construction

When to task-augment vs call sync:
- Augment when your client supports MCP 2025-11-25 Tasks and the validation may exceed your client's tool-call idle timeout (typically ~60s)
- Augment when you want push notifications on state changes (rather than polling `me.validation_history`)
- Stay sync for short validations on clients that haven't adopted Tasks yet; recovery via `me.validation_history(run_id=...)` still works

#### Current task-augmentation scope (PR-1)

At present, only `architect.validate` is task-augmented. `architect.validate_consensus` and `architect.certify` follow the same long-running profile (60-180s+ LLM calls each) and will extend to task augmentation in the next release with the full design needed: parent-child task semantics for the N-parallel-children consensus path, and cert-lifecycle integration with the existing `cert_lifecycle_status` column on `UserValidationRun`.

Until the next release lands, use the sync flow + the `me.validation_history(run_id=<that-id>)` recovery pattern for `architect.validate_consensus` and `architect.certify`. The recovery handle still resolves immediately via the first `notifications/progress` event at t=0s. The asymmetry is intentional canary-then-extend sequencing, not an oversight.

## §2: Decision tree · validate vs validate_consensus vs certify

Three tools, three jobs, ordered:

| You want to… | Call | Cost | Anchored? |
|---|---|---|---|
| Run a fast iteration round + see delta vs prior | `architect.validate` | 1× LLM (~60-180s) | YES; auto-injects prior baseline |
| Surface the honest variance band on a candidate run before treating it as a badge | `architect.validate_consensus` | N× LLM (~80-120s for N=3) | NO; each child runs unanchored on purpose |
| Mint the certified production_ready badge after a clean validate | `architect.certify(run_id, code)` | 1× LLM (~60-150s) | Adversarial second pass |

### When to call which

- **Round-by-round iteration → `architect.validate`**. Cheap, anchored, surfaces the delta against your previous round. Always.
- **Pre-badge stability check → `architect.validate_consensus`**. One single-shot result is one roll of a non-deterministic LLM at high reasoning effort; empirical variance band of ~20-67 points on byte-identical input. Before treating any single-shot 100/A as the badge anchor, run consensus on the same code to surface the variance band. If `mode_stability_min_pct < 80%`, the consensus is unstable; keep iterating, don't badge yet.
- **Final certification → `architect.certify`**. Only after a clean validate AND (recommended) a stable consensus check. Cert runs an adversarial second pass that can downgrade a 100/A validate to emerging/C with `cert_downgraded` if a missed production_blocker surfaces.

### Combined sequence (the canonical operator flow)

```
validate (round 1) → fix → validate (round 2 anchored to round 1) → fix → ...
   → validate_consensus (stability check at iteration end)
   → if stable (mode_stability ≥ 80%) → architect.certify → badge
   → if unstable → return to validate loop
```

### Truth about mixing validate + consensus on the same repository

`architect.validate_consensus` runs its N children with `private_session=True` so each child runs UNANCHORED. But the CONSOLIDATED outer row IS persisted with `lifecycle_status="completed"` and the same `scope_signature` as a regular validate run. **The next single-shot `architect.validate` on the same repository WILL find the consensus consolidated row as the prior_run_baseline**: verified at `validation_service.py:660-684` (`lookup_and_anchor_baseline` filters on `user_id + repository + is_user_deleted=False + lifecycle_status NULL-or-completed + scope_signature`; the consensus row matches all four).

In plain terms: the consensus checkpoint becomes the new anchor for the next iteration round. The arc continues.

## §3: Worked example (the iteration arc that produced this skill)

On 2026-05-22 we submitted `content/example-library/sources/agents/agent-complexity/4-agent-harness.py` to `architect.validate` on prod. The arc:

| Round | Repository | Bytes | Score | Lineage |
|---|---|---|---|---|
| Iter-1 (`5fdfcd1c`) | `aidesignblueprint/agent-harness-curated-example-2026-05-22` | original 167 LOC | **7 / F draft** | First shot, no prior. Found 9 production_blockers including refund-execution in autonomous allow-list (P8 sev 95) and prompt-only policy guard (P5 sev 90). |
| Iter-2 mistake (`45b6e2ff`) | `aidesignblueprint/agent-harness-curated-example-iter-2-2026-05-22` | refactored 374 LOC | **74 / C emerging** | Repository string changed; no anchor found, `baseline_status="none_found"`. Blind shot. Two separate repo rows in history. Improvements invisible. |
| Iter-2 corrected (`8bf6cc31`) | `aidesignblueprint/agent-harness-curated-example-2026-05-22` (original) | same refactored 374 LOC | **74 / C emerging** | Anchored to iter-1's 7/F via auto-resolved `prior_run_baseline`, `baseline_status="used"`. **All 10 principles surfaced explicit severity deltas** (P5: 90→0 aligned, P8: 95→0 aligned, full arc visible). |

**The non-obvious finding**: same code submitted twice scored the same 74/C both times. The score did NOT change between mistake and corrected. But the LINEAGE catastrophically differs. The mistake row sits alone in the dashboard with zero context; the corrected row carries the +67 severity-collapse arc explicitly. **Anchoring doesn't determine the score; it determines the provenance.**

This is why the `repository` string is load-bearing; not for the headline, but for the iteration arc to remain inspectable across rounds. An `iter-2` suffix on the repo key looks like good provenance ("clear which iteration this is!") but is exactly the edit that severs the lineage. **Round numbering belongs in the `task` field or in commit messages, never in the `repository` string.**

The remaining gap on iter-2: P7 (inspectability) at sev 55; the audit trail is still an in-memory `list[AuditEvent]`, not a durable append-only ledger. Iter-3 would target the audit-ledger persistence; the production_ready badge stays gated on that round.

## §4: Payload completeness · stub imports, never sketch the agent

`architect.certify` reads the EXACT bytes that produced the validate `run_id` and grades what would actually run if that payload were the code in production. Two completeness axes are load-bearing, and they are different:

1. **Imports.** Stub the public surface of every imported module. `from app.db import models` → include a `class models:` namespace stub with the columns/methods you reference; module-level imports of `dataclass`, `Literal`, `json`, `datetime`, `timezone` must be present, or the module would `NameError` on import. A run that scored 100/A at validate can still cert-reject pre-LLM with `payload_incomplete` when a dependency's surface isn't visible.
2. **Enforcement branches.** The code under cert *itself* must be the real logic, fully written. A placeholder body (`# ... execute approved action ...`, `pass  # TODO`, a bare `...`) is not a compression of the control; it is its *absence*. Cert grades the missing branch against you, not as shorthand for one that exists.

**Submit like production:** stub your dependencies, never sketch the agent you are certifying. If you'd have to add an import or fill in a branch to make the file actually run, it belongs in the payload. Abbreviated or placeholder submissions are penalised by design.

## Quick reference (when you only have 60 seconds)

| Question | Answer |
|---|---|
| Should I use consensus or single-shot? | Single-shot for iteration rounds. Consensus once, before you treat a run as your badge anchor. |
| What chains rounds together? | Same `repository` string across calls. Different string = severed chain. |
| Why didn't my second call see the first as prior? | Either you changed `repository`, OR you passed `private_session=true` on the first call (it's persisted but private_session-marked, lookup skips it). |
| Does a consensus run become the prior for the next validate? | Yes. The consolidated row participates in the lookup. The children don't (they're private_session). |
| Can I skip the consensus checkpoint before certify? | Technically yes; cert will still run. Strategically no; without consensus you don't know if your 100/A is stable or a lucky single roll. |
| Can I sketch the code under cert to save tokens? | No. Stub *imports*, but the enforcement branches themselves must be real and complete. A `# ...` placeholder is graded as a missing control, not as shorthand. |
