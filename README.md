# fire-stable 🐴🔥

*The agent stable. Year of the Fire Horse, 2026.*

An AI agent swarm management system. Dispatches work from a ticketing system to AI agents running in sandboxed environments, tracks costs, and escalates to humans when agents get stuck.

## What it does

```
ticket in → dispatch to agent → agent works → PR out → human reviews
```

Today: 15+ tmux Claude Code sessions, manually juggled by one human.
Tomorrow: a stable of agents, each in a stall (VM/sandbox), dispatched by butler, observed via dashboard.

## The stack

```
in3eption (wiki/brain)        ← what work exists (feature files)
        │
fire-stable (this repo)       ← manages the agents
  ├── agent registry          ← who's running, on what, costing how much
  ├── dispatcher              ← assigns features to agents
  ├── cost tracker            ← per-agent, per-day, per-feature
  ├── telegram bot            ← escalation for stuck agents
  └── dashboard               ← observe everything
        │
butler (sgx/butler)           ← human-facing ops hub, feeds tickets in
        │
GitHub                        ← PRs, CI, reviews
```

## Principles

1. **Human dispatches all work** — no agent-managing-agent
2. **Every session is observable** — tmux attach, logs, dashboard
3. **Cost visibility is mandatory** — per-session caps, per-day caps, projections
4. **Escalate, don't retry** — stuck agents ping Telegram, not silently burn money

## Status

**Design phase.** Architecture docs in `docs/`, implementation plans in `plans/`.

## Docs

- [Architecture](docs/architecture.md) — system design, pipeline, component breakdown
- [Economics](docs/economics.md) — cost model, projections, control levers
- [Agent Protocol](docs/agent-protocol.md) — how agents behave, escalate, and report

## Related

- [in3eption](https://github.com/sg-altis/in3eption) — the wiki/brain, where features and ideas live
- [sgx/butler](../sgx/butler) — personal ops hub, becomes the human-facing dispatcher
