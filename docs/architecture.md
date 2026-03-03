# Architecture

## System overview

```
┌─────────────────────────────────────────────────────┐
│  in3eption (wiki / brain)                           │
│  ideas/ → features/ → plans/ → lessons/             │
│  "what exists, what we learned, what costs what"    │
└──────────────────────┬──────────────────────────────┘
                       │ feature files (markdown)
                       │ sync or watch
                       ▼
┌─────────────────────────────────────────────────────┐
│  butler (dispatcher)                                │
│  feature.md → TODO → assign agent → track status    │
│  human submits tickets here too (requests)          │
│  SQLite today, Supabase when push matters           │
└──────────┬──────────────────────────┬───────────────┘
           │ dispatch                 │ escalation
           ▼                          ▼
┌─────────────────────┐    ┌──────────────────┐
│  fire-stable         │    │  Telegram         │
│  (agent stalls)      │    │  thread-per-agent │
│  ┌───┐ ┌───┐ ┌───┐ │    │  stuck? → ping    │
│  │VM1│ │VM2│ │VM3│ │    │  cost alert? → ⚠  │
│  └─┬─┘ └─┬─┘ └─┬─┘ │    └──────────────────┘
│    │      │      │   │
│    ▼      ▼      ▼   │
│   PR     PR     PR   │
└────┬──────┬──────┬───┘
     │      │      │
     ▼      ▼      ▼
┌─────────────────────────────────────────────────────┐
│  GitHub                                             │
│  PRs + CI/tests → human reviews → merge             │
│  dashboard reads PR status back                      │
└─────────────────────────────────────────────────────┘
```

## Components

### 1. Agent registry

The data layer. Tracks every agent in the stable.

```
┌─ Agent Registry (SQLite → Supabase) ──────────────────────┐
│                                                            │
│  agent-id   model            vm       status    cost/day   │
│  ─────────  ──────────────   ───────  ────────  ────────   │
│  opus-1     claude-opus-4-6  local    working   $12.30     │
│  sonnet-2   claude-sonnet-4-6 fly-1   idle      $0.00      │
│  sonnet-3   claude-sonnet-4-6 local   blocked   $4.50      │
│  haiku-4    claude-haiku-4-5 local    working   $0.25      │
│                                                            │
│  Fields: agent-id, model, vm, status, current-feature,     │
│          current-branch, session-access, cost-today,        │
│          cost-total, telegram-thread, started-at            │
└────────────────────────────────────────────────────────────┘
```

Schema:

```sql
CREATE TABLE agents (
    id              TEXT PRIMARY KEY,
    model           TEXT NOT NULL,
    vm              TEXT DEFAULT 'local',
    status          TEXT DEFAULT 'idle'
                    CHECK (status IN ('idle','working','blocked','paused','terminated')),
    current_feature TEXT,
    current_branch  TEXT,
    session_access  TEXT,
    cost_today      REAL DEFAULT 0.0,
    cost_total      REAL DEFAULT 0.0,
    telegram_thread INTEGER,
    started_at      TEXT,
    updated_at      TEXT DEFAULT (datetime('now'))
);

CREATE TABLE cost_log (
    id              INTEGER PRIMARY KEY AUTOINCREMENT,
    agent_id        TEXT NOT NULL REFERENCES agents(id),
    feature_slug    TEXT,
    model           TEXT NOT NULL,
    input_tokens    INTEGER DEFAULT 0,
    output_tokens   INTEGER DEFAULT 0,
    cost_usd        REAL DEFAULT 0.0,
    logged_at       TEXT DEFAULT (datetime('now'))
);
```

### 2. Dispatcher

Assigns features to agents. Integrates with butler's TODO system.

```
Feature file (in3eption)                  Butler TODO
───────────────────────                   ──────────────
status: accepted          ──sync──>       status: pending
priority: high                            priority: high
idea: transcript-intel                    source: in3eption
                                          feature_slug: transcript-intel/speaker-diarization

                              │
                         make dispatch
                         FEATURE=... AGENT=...
                              │
                              ▼

Agent picks up work:
  1. Reads feature file from in3eption
  2. Creates branch: feat/<idea>/<feature>
  3. Clones target repo into sandbox
  4. Works autonomously within scope
  5. Opens PR when done
  6. Reports back: PR link, time spent, tokens used
```

Dispatch flow:

```
make dispatch FEATURE=<idea>/<feature> AGENT=<id> [MODEL=sonnet]

  1. Validates feature exists and is accepted
  2. Marks agent as working + sets current_feature
  3. Creates butler TODO linked to feature
  4. Starts agent session (tmux or VM)
  5. Injects: feature file, repo access, branch name, CLAUDE.md
```

### 3. Telegram bot

Escalation channel. One group, thread-per-agent.

```
Group: fire-stable-ops
├── Thread: opus-1
│   ├── [BOT] Dispatched: transcript-intel/speaker-diarization
│   ├── [opus-1] Stuck: can't find the audio processing lib
│   ├── [HUMAN] Try librosa, it's in pyproject.toml
│   └── [opus-1] Resolved, continuing
├── Thread: sonnet-2
│   └── [BOT] Idle since 2h ago
├── Thread: cost-alerts
│   ├── [BOT] ⚠ opus-1 at 80% of $25 session cap ($20.12)
│   └── [BOT] Daily spend: $45.30 / $200 cap (23%)
└── Thread: daily-digest
    └── [BOT] EOD: 4 agents, 3 PRs opened, $67 spent, 1 stuck
```

Escalation message template:

```
🔴 BLOCKED: {agent-id}
Feature: {feature-slug}
Branch: {branch}
Tried: {what was attempted}
Stuck on: {specific blocker}
Need: {what would unblock}
Time blocked: {duration}
```

### 4. Dashboard

Web UI for observing the stable. Could be:
- A new view in butler's web dashboard (:7777)
- A page in altis-dashboard
- A standalone service

Must show:

```
┌─ fire-stable dashboard ────────────────────────────────────┐
│                                                            │
│  AGENTS                              COST TODAY            │
│  ────────────────────────            ─────────             │
│  ● opus-1   working  $12.30         Total: $45.30         │
│  ○ sonnet-2 idle     $0.00          Cap: $200             │
│  ⚠ sonnet-3 blocked  $4.50         Pace: $67/day          │
│  ● haiku-4  working  $0.25         Projected: $2,010/mo   │
│                                                            │
│  RECENT PRs                          FEATURES              │
│  ────────────                        ────────              │
│  #42 speaker-diarization  ⏳ review  3 accepted            │
│  #41 bootstrap-convention ✅ merged   2 in-progress         │
│  #40 agent-registry       ❌ dropped  1 pr-open             │
│                                                            │
│  COST HISTORY (7 days)                                     │
│  $80 ┤                    ╭─╮                              │
│  $60 ┤              ╭─╮  │ │                               │
│  $40 ┤  ╭─╮  ╭─╮  │ │  │ │  ╭─╮                         │
│  $20 ┤  │ │  │ │  │ │  │ │  │ │                          │
│   $0 ┼──┴─┴──┴─┴──┴─┴──┴─┴──┴─┴──                       │
│       Mon  Tue  Wed  Thu  Fri  Sat  Sun                    │
└────────────────────────────────────────────────────────────┘
```

### 5. VM / sandbox management

Agents need isolated environments. Scaling path:

```
Phase 1: Local tmux sessions
  - Each agent = a tmux pane
  - Session access: tmux attach -t <session>
  - Cost: $0 infra, limited by Mac resources
  - Max: ~5-8 concurrent heavy agents

Phase 2: Local VMs (OrbStack / Tart)
  - Each agent = a lightweight macOS/Linux VM
  - Snapshot + restore for reproducible environments
  - Cost: $0 infra, better isolation
  - Max: ~10-15 depending on RAM

Phase 3: Cloud VMs (Fly.io / Railway)
  - Each agent = a cloud container
  - Auto-scale up/down based on queue
  - Cost: ~$30/mo per 2CPU/4GB VM
  - Max: limited by budget, not hardware
```

## Data flow

```
                    ┌──────────┐
                    │ in3eption │
                    │ features  │
                    └────┬─────┘
                         │ watch/sync
                    ┌────▼─────┐
                    │  butler   │
                    │  TODOs    │
                    └────┬─────┘
                         │ dispatch
              ┌──────────┼──────────┐
              ▼          ▼          ▼
          ┌───────┐  ┌───────┐  ┌───────┐
          │agent-1│  │agent-2│  │agent-3│
          └───┬───┘  └───┬───┘  └───┬───┘
              │          │          │
              │    ┌─────▼─────┐   │
              │    │  Telegram  │   │   (escalation)
              │    └───────────┘   │
              │          │          │
              ▼          ▼          ▼
          ┌──────────────────────────┐
          │        GitHub PRs        │
          └────────────┬─────────────┘
                       │ status webhooks
                  ┌────▼─────┐
                  │ dashboard │
                  │ (observe) │
                  └──────────┘
```

## Integration points

| Source | Destination | Mechanism | Data |
|--------|------------|-----------|------|
| in3eption features | butler TODOs | File watch or periodic sync | Feature slug, priority, idea |
| butler | agent | `make dispatch` → tmux/VM | Feature file, repo, branch, CLAUDE.md |
| agent | GitHub | `git push` + `gh pr create` | Code changes, PR |
| agent | cost_log | Token counting or API billing | input/output tokens, USD |
| agent | Telegram | Bot API post | Escalation messages |
| GitHub | butler | Webhook or polling | PR status (open/merged/closed) |
| cost_log | dashboard | SQL query | Aggregated cost data |
| agent registry | dashboard | SQL query | Agent status |
