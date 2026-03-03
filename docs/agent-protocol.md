# Agent Protocol

How agents in the stable behave, from dispatch to completion.

## Agent lifecycle

```
                    ┌──────────────┐
          register  │              │  terminate
       ┌───────────>│    idle      │<────────────────┐
       │            │              │                  │
       │            └──────┬───────┘                  │
       │                   │ dispatch                 │
       │            ┌──────▼───────┐                  │
       │            │              │  human pauses    │
       │            │   working    │──────────┐       │
       │            │              │          │       │
       │            └──┬───────┬───┘          │       │
       │               │       │              │       │
       │          opens PR   gets stuck       │       │
       │               │       │              │       │
       │               │  ┌────▼───────┐  ┌───▼────┐ │
       │               │  │            │  │        │ │
       │               │  │  blocked   │  │ paused │ │
       │               │  │            │  │        │ │
       │               │  └────┬───────┘  └───┬────┘ │
       │               │       │              │       │
       │               │   human unblocks  human resumes
       │               │       │              │       │
       │               │       └──────┬───────┘       │
       │               │              │               │
       │               ▼              ▼               │
       │         ┌─────────────────────┐              │
       └─────────│    task complete    │──────────────┘
                 │  (PR opened/merged) │
                 └─────────────────────┘
```

## Dispatch protocol

When an agent is dispatched, it receives:

```
1. Feature file (markdown)
   - What, Why, Scope, Implementation notes, Acceptance criteria
   - From: in3eption/ideas/<slug>/features/YYYY-MM-DD_<feature>.md

2. Target repo access
   - Git clone URL
   - Branch name: feat/<idea>/<feature>
   - Base branch (usually main)

3. CLAUDE.md / project context
   - Repo conventions, test commands, build steps

4. Constraints
   - Model to use (haiku/sonnet/opus)
   - Cost cap for this session
   - Escalation instructions (Telegram thread)
```

## Agent behavior rules

### Working

1. **Read the feature file first.** Understand What, Why, Scope, and Acceptance criteria before writing code.
2. **Stay in scope.** Do not add features, refactor code, or make improvements beyond what the feature file describes.
3. **Create the branch.** `feat/<idea>/<feature>` — always branch from the base branch.
4. **Commit incrementally.** Small, atomic commits with descriptive messages.
5. **Open a PR when done.** Include a summary referencing the feature file. Tag the human for review.

### Blocked

When stuck (build fails, unclear requirements, missing access, dependency issue):

1. **Do not silently retry.** Three attempts max, then escalate.
2. **Post to Telegram** using the escalation template:

```
🔴 BLOCKED: {agent-id}
Feature: {feature-slug}
Branch: {branch}
Tried: {what was attempted, briefly}
Stuck on: {specific blocker}
Need: {what would unblock — access, clarification, dependency}
Time blocked: {how long since first attempt}
```

3. **Pause and wait.** Do not burn tokens spinning. The human will respond in the Telegram thread.
4. **When unblocked**, acknowledge in thread and resume.

### Completing

When done:

1. **Open PR** with summary referencing the feature file
2. **Report to butler:**
   - PR link
   - Time spent (started_at → now)
   - Estimated token usage
   - Any notes for the reviewer
3. **Update feature file status** to `pr-open` with PR link in `prs:` field
4. **Return to idle** — agent is available for next dispatch

### Cost awareness

Agents should be cost-conscious:

- **Don't read entire codebases** when a targeted search works
- **Don't regenerate** what can be edited
- **Don't use Opus-level reasoning** for Sonnet-level tasks (but this is set by the dispatcher, not the agent)
- **Log token estimates** in completion report

## Escalation severity

| Level | Trigger | Channel | Response time |
|---|---|---|---|
| Info | Progress update, PR opened | Telegram thread | No response needed |
| Warning | Approaching cost cap (80%) | Telegram cost-alerts | Within 1 hour |
| Blocked | Agent stuck, needs human input | Telegram agent thread | Within 30 min |
| Critical | Cost cap hit, agent auto-paused | Telegram cost-alerts | Within 15 min |

## Handoff protocol

If an agent is terminated or times out mid-task:

1. Butler logs the interruption (daily_log with category='interruption')
2. Feature status stays `in-progress` (not dropped — work may be salvageable)
3. Next agent dispatched to the same feature:
   - Reads the existing branch (if any commits were pushed)
   - Reads the Telegram thread for context
   - Continues from where the previous agent left off
4. **Context survives agent death** — this is the whole point of feature files + branches + Telegram threads

## Session access

Every agent must be reachable for human observation:

| Environment | Access method |
|---|---|
| Local tmux | `tmux attach -t <session-name>` |
| Local VM (OrbStack) | `orb ssh <vm-name>` |
| Cloud VM (Fly.io) | `fly ssh console -a <app-name>` |
| Cloud VM (generic) | `ssh <user>@<host>` |

The `session_access` field in the agent registry stores the exact command.
