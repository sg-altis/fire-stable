# fire-stable — Agent Conventions

## What is this

Agent swarm management system. Dispatches work to AI agents, tracks costs, escalates when stuck. Named for the Year of the Fire Horse (2026).

## Project phase

Design phase — no application code yet. Architecture docs, plans, and design decisions only. Code comes after the design is reviewed and accepted.

## Principles

1. **Human dispatches all work** — no agent-managing-agent
2. **Every session is observable** — no black-box agents
3. **Cost visibility is mandatory** — surprise bills are operational failures
4. **Escalate, don't retry** — stuck agents ping Telegram
5. **Start manual, automate later** — markdown registry before database, Make targets before API

## Related repos

- **in3eption** (`sg-altis/in3eption`) — wiki/brain. Ideas, features, plans, lessons. fire-stable reads feature files from here.
- **butler** (`sgx/butler`) — personal ops hub. TODO/request system, TUI + web dashboard. Becomes the human-facing dispatcher layer.

## Docs structure

```
docs/
  architecture.md    — system design, pipeline, components
  economics.md       — cost model, projections, control levers
  agent-protocol.md  — agent lifecycle, escalation, reporting
plans/
  PLAN-*.md          — implementation plans (phased)
```

## Conventions

- Dates use ISO 8601: YYYY-MM-DD
- Cost data is ballpark, always dated
- ASCII diagrams over verbose descriptions
- Provenance on all files (author, driver, agents in frontmatter)
