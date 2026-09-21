---
name: agentic-design-blueprint
description: Use the Agentic Design Blueprint when designing, reviewing, or implementing agentic AI products: tool use, approvals, orchestration, background work, trust surfaces, and human-in-the-loop workflows.
---

# Agentic Design Blueprint

Apply these principles before finalizing agentic UX, orchestration, tool calling, approvals, handoffs, and review logic.

## When to use this skill

Use when the task involves:
- Agentic UX or workflow design
- Multi-step AI systems or agent orchestration
- Tool calling or MCP integrations
- Human approvals, blockers, or handoffs
- Background work visibility
- Trust, inspectability, or auditability
- Product readiness reviews or architecture validation

## Core Principles

### 1. Design for delegation rather than direct manipulation

Design experiences around the assignment of work, the expression of intent, the setting of constraints, and the review of results, rather than requiring users to execute each step manually.

### 2. Ensure that background work remains perceptible

When the system is operating asynchronously or outside the user’s immediate focus, it should provide persistent and proportionate signals that work is continuing.

### 3. Align feedback with the user’s level of attention

The system should calibrate the depth and frequency of feedback according to whether the user is actively engaged, passively monitoring, or temporarily absent.

### 4. Apply progressive disclosure to system agency

Provide the minimum information necessary by default, while enabling users to inspect additional detail when confidence, understanding, or intervention is required.

### 5. Replace implied magic with clear mental models

The product should help users understand what the system can do, what it is currently doing, what it cannot do, and what conditions govern its behaviour.

### 6. Expose meaningful operational state, not internal complexity

Present the state of the system in language and structures that are relevant to the user, rather than exposing low-level internals that do not support action or understanding.

### 7. Establish trust through inspectability

Users should be able to examine how a result was produced when confidence, accountability, or decision quality is important.

### 8. Make hand-offs, approvals, and blockers explicit

When the system cannot proceed, the reason should be immediately visible, along with any action required from the user or another dependency.

### 9. Represent delegated work as a system, not merely as a conversation

Where work involves multiple steps, agents, dependencies, or concurrent activities, it should be represented as a structured system rather than solely as a message stream.

### 10. Optimise for steering, not only initiating

The system should support users not only in starting tasks, but also in guiding, refining, reprioritising, and correcting work while it is underway.

## MCP retrieval

When the `aidesignblueprint` MCP server is connected, prefer live retrieval:

- `principles.list()`: all 10 principles with slugs
- `principles.search(query)`: semantic search
- `principles.get(slug)`: full principle with rationale and implications
- `clusters.list()`: principle clusters (delegation, visibility, trust, orchestration)
- `examples.search(query)`: curated implementation examples
- `guides.get(slug)`: deep application guides
- `architect.validate(implementation_context)` (Pro): single-pass doctrine review (~30-50s)
- `architect.certify(run_id, code)` (Pro): second-pass certify a production_ready run; mints the trust badge
- `design.validate(implementation_context)` (Pro): single-pass surface review against the 8 experience-design laws; recover timeouts via me.validation_history(run_id=...)
- `spec.validate(implementation_context)` (Pro): single-pass spec review of a WRITTEN specification against the 8 spec-quality laws (own weekly bucket, no topup); recover timeouts via me.validation_history(run_id=...)
- `me.validation_history(repository?, run_id?)` (Pro): per-repository trend, regression diff, or single-run recovery by run_id

## Two-step validate → certify

`architect.validate` returns a single-pass review fast. When it scores production_ready (A or B tier), the response carries `certification_status='not_evaluated'` and a `run_id`. Call `architect.certify(run_id=..., code=...)` to mint the certified production_ready badge. Cert is a separate Pro/Teams tool because it does adversarial second-pass review on the same code; running it inline would push validate past the ~60s MCP-client tool budget. Cert eligibility gate: caller-owned run, tier=production_ready, less than 24h old, not already certified, retry budget remaining. The sibling lenses, `design.validate` (rendered surfaces) and `spec.validate` (written specs), are single-pass in v1 with no consensus or certify: iterate until the grade holds.

## Timeout recovery

If your MCP client tool-call closes before `architect.validate`, `design.validate`, or `spec.validate` returns, the run still completes server-side. The first `notifications/progress` event fires at t=0 carrying the `run_id`; recover the result via `me.validation_history(run_id=...)` once the run completes. Per-user authorisation: returns only your own runs. Unavailable when `private_session=true` (nothing persists).

## References

- Site: https://aidesignblueprint.com/en/principles
- MCP endpoint: https://aidesignblueprint.com/mcp
- Examples: https://aidesignblueprint.com/en/examples
- For agents: https://aidesignblueprint.com/en/for-agents
