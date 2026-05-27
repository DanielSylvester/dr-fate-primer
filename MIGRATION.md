# Migration Guide: Legacy → Dr. Fate

> **Zero-touch migration.** The old system keeps running. Dr. Fate grows alongside it. No rip-and-replace.

## The Mapping

| Legacy Repo | Legacy Role | Dr. Fate Replacement | Migration Path |
|------------|-------------|---------------------|----------------|
| `centralintelligence` | Knowledge base, arago RAG, conventions | **`dr-fate-memory`** | Phase 1: Import all `.md` files. Phase 4: `arago` → `fate memory query` |
| `adpilot` | Orchestrator, scheduler, LLM decisions | **`dr-fate-orchestrator`** | Phase 3: Port daily run logic. Phase 6: Cutover account-by-account |
| `marketing-ops-mcp` | MCP server for Meta/Google/LinkedIn | **`dr-fate-executor`** | Phase 2: Port all tools. Phase 4: Run parallel, compare outputs |
| `gads-mcp` | Google Ads MCP (read + write) | **`dr-fate-executor`** (gads tools) | Phase 2: Port tools with same safety gates |
| `daily-reporting-v3` | Lead pipeline, conversion upload, audience sync | **Stays as-is** | Apps Script is the right tool for Sheets. Integrates via `fate memory observe` |
| `spend-update-automation` | Spend tracking for India + Singapore | **Stays as-is** | Apps Script for Sheets. May feed data to `memory` for context |
| `strapi-lead-query` | Strapi → Sheets sync | **Stays as-is** | Lead ingestion pipeline. Events observed by `memory` |
| `meta-smartworks-lead-alert-v1` | Meta lead alerts (India) | **Stays as-is** | Real-time alerts. May trigger `orchestrator` events in future |
| `meta-workstudio-lead-alert-v1` | Meta lead alerts (Singapore) | **Stays as-is** | Same as above |
| `personal-config` | Environment config, skills | **Stays as-is** | Source of truth for agent skills. Dr. Fate config lives in `/etc/dr-fate/` |

## What Stays

The following are **not** being replaced. They do their job well:

- **Apps Script projects** (`daily-reporting-v3`, `spend-update-automation`, `strapi-lead-query`, lead alerts) — Sheets integration is their superpower. Dr. Fate observes their outputs, doesn't replace them.
- **personal-config** — Skills and environment config remain the source of truth.
- **Danmarketing-vercel** — Personal website, unrelated to operations.

## What Moves

Three things get a new home:

1. **Knowledge** (`centralintelligence/knowledge/`, `arago.py`) → `dr-fate-memory`
2. **Decisions** (`adpilot/src/agent/`) → `dr-fate-orchestrator`
3. **Execution** (`marketing-ops-mcp/`, `gads-mcp/`) → `dr-fate-executor`

## Parallel Run Strategy

```
Week 1-2:   Dr. Fate deployed in OBSERVE mode for smartworks-india
            Legacy adpilot runs normally
            Compare outputs daily

Week 3-4:   Fix discrepancies, tune prompts, refine safety gates
            Still OBSERVE mode

Week 5-6:   Switch to HYBRID mode (high-confidence auto-execute)
            Monitor closely

Week 7-8:   Switch to EXECUTE mode for smartworks-india
            Legacy adpilot disabled for this account

Week 9+:    Repeat for workstudio-singapore, then next account
```

## Data Migration

| Data | Source | Destination | Method |
|------|--------|-------------|--------|
| Knowledge docs | `centralintelligence/knowledge/*.md` | `dr-fate-memory` | Bulk ingest script |
| Action history | `adpilot` Turso `actions` table | `dr-fate-orchestrator` | SQL export → import |
| Audit log | `marketing-ops-mcp` SQLite | `dr-fate-executor` | SQL export → import |
| Account config | `adpilot/src/config.ts` | `/etc/dr-fate/accounts/*.yaml` | Manual translation |
| Safety rules | `adpilot/src/analyst/safety.ts` | `/etc/dr-fate/safety.yaml` | Manual translation |

## Rollback Plan

If Dr. Fate fails for any account:
1. Disable Dr. Fate workflow for that account (`fate workflow disable --account=X`)
2. Re-enable legacy adpilot workflow for that account
3. All data remains in both systems during parallel run
4. No data loss — audit logs are immutable in both systems

## Timeline

See [ROADMAP.md](./ROADMAP.md) for the full 14-week plan.

**Current status:** Phase 0 complete. Architecture defined, repos created, skeletons pushed.
