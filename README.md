# AI Design Blueprint

Public agentic AI doctrine tools plus authenticated architecture, design, and spec validators.

This repository is generated. Do not open pull requests against it; the source
lives in the AI Design Blueprint platform and is republished from there. Found an
error in the doctrine? Open an issue here, or use the `signals.feedback` MCP tool.
We route it back to the source.

## What you get

- Skills that apply the doctrine while you design and build agentic products.
- A connection to the AI Design Blueprint MCP server at `https://aidesignblueprint.com/mcp`. It is a
  remote server, so no server process runs on your machine.
- In Claude Code, three ready-made subagents (PM, Designer, Engineer) that build
  or review against the doctrine and grade the result with the validator tools.

The doctrine tools (principles, clusters, examples, guides) work without an account.
The validators (`architect.validate`, `design.validate`, `spec.validate`) need a
signed-in AI Design Blueprint account with an active Pro, Pro Plus, Teams, or beta plan. Your tool
will ask you to sign in the first time you call one.

## Install

**Claude Code**

```
/plugin marketplace add aidesignblueprint/ai-design-blueprint
/plugin install ai-design-blueprint@aidesignblueprint
```

**Gemini CLI**

```
gemini extensions install https://github.com/aidesignblueprint/ai-design-blueprint
```

**OpenAI Codex**

```
codex plugin marketplace add aidesignblueprint/ai-design-blueprint
```

**GitHub Copilot and Cursor** read the `plugin.json` at the root of this
repository. Install it from the plugin marketplace inside your editor.

## Skills

- `agentic-design-blueprint`
- `architect-validation-orchestration`
- `design-validation-orchestration`
- `spec-validation-orchestration`
- `team-agents-orchestration`

## Links

- Documentation: https://aidesignblueprint.com/en/for-agents
- Plans and access: https://aidesignblueprint.com/en/pricing
- Data handling: https://aidesignblueprint.com/en/for-agents/trust-and-data-handling
- Privacy: https://aidesignblueprint.com/en/privacy
- Terms: https://aidesignblueprint.com/en/terms
- Support: https://aidesignblueprint.com/en/support

## Licence

MIT. See `LICENSE`.
