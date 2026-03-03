# Economics of the Stable

*Cost data as of 2026-03. Pricing changes — always verify.*

## Per-task cost by model

The dispatcher's highest-leverage decision is **model selection per task**. Getting this right saves 5–10× per ticket.

| Task complexity | Model | Cost per task | Examples |
|---|---|---|---|
| Trivial | Haiku | ~$0.50 | typo fix, .gitkeep, rename, simple config |
| Standard | Sonnet | ~$3–5 | new feature, test suite, docs, bug fix |
| Complex | Opus | ~$15–30 | architecture, multi-file refactor, new system |
| Research | Opus | ~$20–50 | open-ended exploration, design doc, analysis |

## Model pricing (Anthropic, 2026-03)

| Model | Input | Output | Cache write | Cache read |
|---|---|---|---|---|
| Opus 4.6 | $15/MTok | $75/MTok | $18.75/MTok | $1.50/MTok |
| Sonnet 4.6 | $3/MTok | $15/MTok | $3.75/MTok | $0.30/MTok |
| Haiku 4.5 | $0.80/MTok | $4/MTok | $1/MTok | $0.08/MTok |

*MTok = million tokens. Extended thinking multiplies output token usage.*

## Monthly projection at 15 agents

```
Cost layer                   Unit cost              Monthly estimate
─────────────────────────    ────────────────────   ────────────────
Claude Opus (heavy work)     ~$15/hr input-heavy    $3,000–6,000
Claude Sonnet (light work)   ~$3/hr input-heavy     $500–1,500
Claude Haiku (trivial)       ~$0.25/hr              $50–100
VMs (local Mac)              $0                     $0
VMs (cloud, e.g. Fly.io)    ~$30/mo per 2CPU/4GB   $450 for 15
GitHub (CI minutes)          free tier → $4/1000m   ~$50
Telegram Bot API             free                   $0
Supabase (if migrated)       free tier → $25/mo     $25
─────────────────────────    ────────────────────   ────────────────
Conservative (mostly Sonnet)                        ~$1,500/mo
Moderate (mixed Opus+Sonnet)                        ~$4,000/mo
Heavy (mostly Opus)                                 ~$8,000/mo
```

## Cost per feature

This is the metric that matters — **"was this feature worth building?"**

```
Feature cost = agent hours × model rate + CI minutes + VM time

Example — trivial Haiku task (typo fix):
  Agent work:     10 min × $0.25/hr = $0.04
  CI:             1 run × $0.01     = $0.01
  VM:             local              = $0.00
  ─────────────────────────────────────────
  Total:                              ~$0.05

Example — standard Sonnet feature:
  Agent work:     45 min × $3/hr    = $2.25
  CI:             2 runs × $0.01    = $0.02
  VM:             1 hr × $0.04      = $0.04
  ─────────────────────────────────────────
  Total:                              ~$2.31

Example — complex Opus architecture:
  Agent work:     3 hr × $15/hr     = $45.00
  CI:             5 runs × $0.01    = $0.05
  VM:             3 hr × $0.04      = $0.12
  ─────────────────────────────────────────
  Total:                              ~$45.17
```

## Cost control levers

| Control | Default | Action |
|---|---|---|
| Per-session cap | $25 | auto-pause agent, Telegram alert |
| Per-day cap | $200 | auto-pause all agents, Telegram alert |
| 80% warning | — | Telegram warning |
| Projection alert | >2× daily cap pace | Telegram warning |
| Model downgrade | — | manual decision (dispatcher suggests, human approves) |

**Never hard-kill a session.** Always pause + alert. The agent might be mid-commit.

## Cost tracking sources

| Provider | How to get cost data | Granularity |
|---|---|---|
| Anthropic | API usage dashboard, or count tokens in responses | Per-request |
| OpenAI | API usage dashboard | Per-request |
| Google (Gemini) | API usage dashboard | Per-request |
| Fly.io / cloud VMs | Billing API | Per-VM per-hour |
| GitHub Actions | Usage page | Per-minute |

Phase 1 (manual): Agents self-report token counts at end of task.
Phase 2 (automated): Wrapper around API calls counts tokens, logs to cost_log table.
Phase 3 (integrated): Real-time cost streaming to dashboard.

## Break-even analysis

The stable is worth building if it saves more than it costs to build and run.

```
Cost to build fire-stable MVP:
  ~1 week of Opus agent work             ~$500–1,000
  Human design + review time              ~10 hours

Value of the stable (monthly):
  Human time saved:
    Manual tmux juggling: ~2 hr/day × 20 days = 40 hr/mo
    With stable:          ~30 min/day × 20 days = 10 hr/mo
    Saved: 30 hr/mo of context-switching

  Wasted agent cycles avoided:
    Unsupervised agents going sideways: ~20% of spend
    At $4K/mo spend: ~$800/mo saved
    With stable (observed, capped): ~5% waste = $200/mo
    Saved: ~$600/mo

Break-even: month 1 if moderate usage.
```

## Budget scenarios

| Scenario | Monthly budget | Agents | Model mix | Output |
|---|---|---|---|---|
| Bootstrap | $500 | 3-5 | Mostly Sonnet | 100-150 features/mo |
| Growth | $2,000 | 8-12 | Sonnet + Opus for complex | 50-80 features/mo (higher quality) |
| Scale | $5,000 | 15-20 | Full mix | 200+ features/mo |
| Max burn | $10,000 | 20+ | Opus-heavy | High-quality architecture + features |

Feature counts are rough — a "feature" varies wildly in scope. The real metric is **merged PRs per dollar**.
