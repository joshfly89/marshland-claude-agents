# Philosophy

These nine agents exist because a junior dev who does what you ask isn't a team. A team has opinions. A team pushes back. A team knows its specialty better than you do and tells you when you're wrong.

That's what I wanted out of Claude Code, and it's not what I was getting by default. So I built it.

> **Note:** This repo ships **3 of the 9** agents (`challenge`, `architect`, `security`) as a free MIT-licensed sample. Full pack at <https://marshland.software/lab/agents>. The philosophy below applies to the whole set.

## Who this is for

If you're a solo dev or small-agency technical lead working in a TypeScript / AWS / SST / React stack, shipping real things with Claude Code, and tired of agreeable output that compounds into messy systems — this is for you. If you're on a different stack or looking for a configurable framework to drop into an enterprise team, it probably isn't.

## The constraint that shapes everything: solo maintenance

I run a one-person consultancy plus two SaaS products (with more coming). Everything I build, I have to maintain alone on a Saturday morning six months from now when I've forgotten how it works. That lens changes every decision.

It's why the agents are small and focused instead of one mega-agent with forty capabilities. It's why they assume a specific stack instead of trying to be universal. It's why there are nine of them, not thirty. Each one is something I actually reach for on real work, not something I imagined might be useful.

If an agent doesn't earn its keep in my own week, it doesn't ship. That's the filter.

## The stack these assume

SST v3 on AWS. TypeScript. DynamoDB single-table. React 19. 11ty for static sites. pnpm. Bash. Mac.

These aren't suggestions — they're assumptions baked into the agents' thinking. The `dynamo` agent assumes single-table design and will push back if you're modeling relational. The `lambda` agent assumes SST v3 conventions. The `ui` agent assumes React and Tailwind.

This is a deliberate choice. An agent that tries to be stack-agnostic is an agent that can't have opinions about your stack. Opinions are the point.

If you're outside this stack, the philosophy still transfers but the agents will need modification. That's fair; you're getting the source, not a SaaS.

## Design choices I'd defend

**Single responsibility per agent.** Nine focused specialists beat one generalist for the same reason nine real engineers beat one generalist on a real team. Each agent keeps its context tight and its opinions sharp. The `security` agent doesn't try to also think about UI. The `cost` agent doesn't also do SEO. When I need security thinking, I invoke security.

**Opinionated, not configurable.** These agents have stances. The `challenge` agent will tell you your plan is wrong. The `architect` agent will push back on your design. The `cost` agent treats a $40/month line item as a problem worth solving. You can modify them — they're just markdown files — but out of the box they behave like experienced teammates who've seen this movie before.

**Push back, don't perform.** The biggest failure mode of Claude by default is agreeing. A junior says "sure, I'll build it." A senior says "I'd do it differently, and here's why." Every agent here is built to do the second thing. If you want a yes-machine, these aren't it.

**Clean output that matches how I already write code.** The agents produce code in the style I'd write it — not some abstract ideal, not the most common Stack Overflow pattern. That means clear naming, small files, opinionated imports, real error handling only where it matters, comments where the "why" isn't obvious.

## Where these came from

Every agent in this package came out of real work at Marshland Software, Grid Getter (my Tesla Powerwall demand charge app), and Review La La (my reputation management SaaS in beta). They're shaped by:

- Designing actual DynamoDB schemas for production apps
- Running actual Screaming Frog audits for clients
- Shipping actual iOS and Android apps
- Arguing with actual Claude sessions about whether my architecture made sense

This isn't theory. If an agent is in this package, I've used it in anger. And yes, I've definitely called it stupid, just like I would a junior dev making that same mistake.

## What this isn't

- Not an enterprise framework. No team conventions, no RBAC, no governance layer.
- Not stack-agnostic. I already said it but it's worth repeating.
- Not a replacement for Claude's general capability. These agents sit on top of Claude Code and call on it; they don't try to wrap or replace it.
- Not configurable infrastructure. These are opinionated markdown files. You fork and modify. There's no settings panel.
- Not production-grade tooling. They're battle-tested in a solo context. A team of fifty would want something different.

If any of that is a dealbreaker, good — we saved each other some time.

## The trade I'm making with you

You get agents I actually use, opinions I actually hold, and a stack assumption that lets the agents be sharp instead of hedged. In exchange, you accept that if you're not in this stack or you want a yes-machine, this isn't the right package.

That's the whole deal.

— Josh / Marshland Software / Phoenix, AZ
