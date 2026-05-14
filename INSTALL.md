# Installation

This installs the 3 sample agents (challenge, architect, security). For the full 9-agent pack, see <https://marshland.software/lab/agents>.

## Quick Install

```bash
mkdir -p ~/.claude/agents
cp agents/challenge/agent.md  ~/.claude/agents/challenge.md
cp agents/architect/agent.md  ~/.claude/agents/architect.md
cp agents/security/agent.md   ~/.claude/agents/security.md
```

## Using the Agents

Spawn them with the Agent tool inside Claude Code:

```typescript
Agent({
  subagent_type: "challenge",
  prompt: "Review this code for correctness..."
})
```

Available `subagent_type` values from this sample: `challenge`, `architect`, `security`.

## Location

Claude Code looks for custom agents in:
- `~/.claude/agents/` (user agents)
- `.claude/agents/` (project agents)

## Troubleshooting

If agents don't appear in Claude Code:
1. Confirm each agent file lives at `~/.claude/agents/<name>.md`.
2. Confirm the frontmatter `name:` field matches the filename.
3. Restart Claude Code to reload agent definitions.
