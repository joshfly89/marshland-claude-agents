# Marshland Claude Agents — Sample Pack (3 of 9)

Three of the nine Marshland Claude agents, free under MIT. Take them, fork them, see if they fit your workflow before you decide to grab the rest.

## What's in the sample

- **challenge** — Code correctness review. Bugs, edge cases, race conditions, type safety holes, error handling gaps.
- **architect** — Architecture and design reviews. Abstractions, data modeling, service boundaries, dependency direction, build-vs-buy tradeoffs.
- **security** — Adversarial security review. Auth bypasses, injection vectors, exposed secrets, overly broad IAM, client-side trust issues.

## Installation

```bash
cp agents/challenge/agent.md  ~/.claude/agents/challenge.md
cp agents/architect/agent.md  ~/.claude/agents/architect.md
cp agents/security/agent.md   ~/.claude/agents/security.md
```

Detailed instructions: [INSTALL.md](INSTALL.md).

## Philosophy

Read [PHILOSOPHY.md](PHILOSOPHY.md) for the design principles. Short version: opinionated, focused, push back, assume a specific stack.

## License

MIT

---

**Want all 9 agents?** The full pack adds `cost`, `dynamo`, `lambda`, `ui`, `seo-content`, and `seo-technical` — plus a project-scoped CLAUDE.md system. Available at <https://marshland.software/lab/agents>.
