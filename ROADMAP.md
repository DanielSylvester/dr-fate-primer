# Dr. Fate Roadmap

> *"A complex system that works is invariably found to have evolved from a simple system that worked."*
> — John Gall

## Philosophy

Build the smallest thing that works. Run it. Learn from it. Then add the next smallest thing. No big-bang releases. No parallel rewrites. One account, one workflow, one feature at a time.

---

## Phase 0: Foundation (Weeks 1-2)

**Goal:** Skeleton repos, defined contracts, hello-world pings between pillars.

| Task | Owner | Deliverable |
|------|-------|-------------|
| Primer repo | Primer | README, PHILOSOPHY, ARCHITECTURE, CONTRACTS, ROADMAP |
| Shared schemas | Primer | JSON Schema definitions + generated TS/Python types |
| Memory skeleton | Memory | FastAPI app with `/health`, `/ingest`, `/query` stubs |
| Orchestrator skeleton | Orchestrator | HTTP server with `/health`, `/workflows`, `/runs` stubs |
| Executor skeleton | Executor | HTTP server + MCP stub with `/health`, `/validate`, `/execute` |
| Cross-pillar ping | All | `memory` can call `orchestrator/health`, `executor` can call `memory/health` |
| CI setup | All | GitHub Actions for lint, typecheck, test (even if tests are empty) |

**Definition of Done:** `curl` any pillar's `/health` and get `{"status": "ok"}`. All four repos pushed to GitHub.

---

## Phase 1: Memory Works (Weeks 3-4)

**Goal:** Memory can ingest, query, and synthesize. It replaces `arago.py` + `centralintelligence/knowledge/`.

| Task | Deliverable |
|------|-------------|
| SQLite + FTS5 setup | `memory.db` with `memories` table and FTS5 virtual table |
| Document ingestion | `POST /ingest` stores, chunks, and indexes documents |
| Hybrid query | `POST /query` with FTS5 + vector (embeddings stubbed or real) |
| Tag filtering | Filter by `account_id`, `type`, `tags`, date range |
| Synthesis | `POST /synthesize` calls LLM to distill query results |
| Bulk import | Script to import all `centralintelligence/knowledge/*.md` into Memory |
| Graph relations | Basic entity extraction and relationship indexing |
| CLI tool | `fate memory ingest <file>` and `fate memory query "..."` |

**Definition of Done:** `fate memory query "MQL definition"` returns the correct answer from imported knowledge. Faster than `arago` and richer context.

---

## Phase 2: Executor Works (Weeks 5-6)

**Goal:** Executor can validate and execute commands against Google Ads and Meta. It replaces `marketing-ops-mcp` + `gads-mcp`.

| Task | Deliverable |
|------|-------------|
| Tool registry | `GET /tools` returns all available tools with JSON schemas |
| Google Ads adapter | OAuth, GAQL query, add negative keyword, budget update |
| Meta Ads adapter | Token auth, campaign list, pause/resume, budget update |
| Validation layer | Schema validation + safety gates (budget bands, protected terms) |
| Audit logging | Every command logged to SQLite with full request/response |
| Dry-run mode | `EXECUTE=0` runs validation but skips API calls |
| MCP transport | `stdio` mode for Claude Code / Kimi integration |
| Rate limiting | Per-platform token bucket with `Retry-After` respect |
| Circuit breaker | Fail fast after 5 consecutive errors, cooldown 5 min |

**Definition of Done:** `fate executor execute --dry-run --tool=gads_add_negative_keyword --arg campaign_id=123 --arg keyword=test` returns validated result without touching live APIs.

---

## Phase 3: Orchestrator Works (Weeks 7-8)

**Goal:** Orchestrator can run a full daily cycle for one account in `observe` mode. It replaces `adpilot/src/agent/runner.ts`.

| Task | Deliverable |
|------|-------------|
| Account config loader | YAML-based, hot-reload on SIGHUP |
| Workflow engine | Define workflows as DAGs of steps (plan → decide → validate → execute) |
| Scheduler | Cron trigger + manual trigger + event trigger |
| Context builder | Fetch live data, query Memory, assemble prompt |
| LLM integration | OpenRouter with model fallback chain, token tracking |
| Decision parser | Parse JSON decisions from LLM, validate with Zod |
| Safety pre-filter | Protected terms, competitor terms, budget bands, dedup |
| Dispatch to Executor | Send validated commands, handle responses |
| Outcome measurement | 7-day before/after comparison |
| Learning loop | Synthesize outcomes, write learnings to Memory |
| Reporting | Generate markdown report, send via email |
| Feedback ingestion | Accept human feedback, store in Memory |

**Definition of Done:** A full daily run for `smartworks-india` completes end-to-end in `observe` mode. Proposed decisions are logged, not executed. Report is generated.

---

## Phase 4: Integration & Polish (Weeks 9-10)

**Goal:** The three pillars work seamlessly. Replace `arago` with `fate memory query`. Replace adpilot daily run with `fate run`.

| Task | Deliverable |
|------|-------------|
| Fate CLI | `fate` command with subcommands: `run`, `memory`, `executor`, `tail`, `status` |
| Event stream | Shared SQLite `fate_stream` table, all pillars write to it |
| Live introspection | `/proc` endpoints on all pillars |
| Health checks | Composite health: `fate status` checks all three |
| Parallel run | Run Dr. Fate alongside legacy adpilot for 1 account, compare outputs |
| Error handling | Circuit breakers, retries, dead-letter queues, alerting |
| Documentation | API docs (OpenAPI), usage guides, migration guide |
| Performance | Memory query <100ms, Orchestrator run <5 min, Executor call <2s |

**Definition of Done:** Dan runs `fate run --workflow=google-ads-daily-agent --account=smartworks-india` and gets the same quality of decisions as legacy adpilot, but with full observability and no code/config coupling.

---

## Phase 5: Multi-Account & Scale (Weeks 11-12)

**Goal:** Run for multiple accounts. Scale horizontally where needed.

| Task | Deliverable |
|------|-------------|
| Multi-account scheduler | Matrix workflows across accounts, no resource contention |
| Account isolation | Separate SQLite DBs per account in Memory, prefix tables in Orchestrator |
| Workstudio Singapore | Onboard second account with different currency, timezone, platforms |
| Template accounts | Clone account config from template, customize per client |
| Dashboard | Simple web UI showing runs, decisions, outcomes per account |
| Alerting | Slack/email alerts for failures, anomalies, threshold breaches |
| Auto-learned rules | Executor suggests new safety rules based on patterns |
| Cost optimization | Cheaper model routing, caching, prompt compression |

**Definition of Done:** Both Smartworks India and Workstudio Singapore run daily in `observe` mode with zero cross-contamination.

---

## Phase 6: Cutover (Weeks 13-14)

**Goal:** Switch from legacy to Dr. Fate for production accounts.

| Task | Deliverable |
|------|-------------|
| Hybrid mode | High-confidence decisions auto-execute, low-confidence propose |
| Execute mode | All safe decisions execute automatically |
| Legacy decommission | Archive old repos, migrate remaining data, document lessons |
| Security audit | Review all credential handling, audit logs, access controls |
| Disaster recovery | Backup/restore procedures, rollback plan |
| Team onboarding | Docs for new developers, runbooks for ops |

**Definition of Done:** Legacy adpilot workflows disabled. Dr. Fate runs all accounts in production. Team can onboard new clients in <1 hour.

---

## Phase 7: Beyond (Future)

| Idea | Description |
|------|-------------|
| LinkedIn Ads agent | Full campaign lifecycle for LinkedIn |
| Projection engine | Forecast spend, leads, MQL/SQL based on historical rates |
| Creative auditor | Auto-analyze Meta creative performance, suggest variants |
| Lead scoring | ML-based lead quality prediction from Strapi data |
| Conversational interface | Chat with the system: "Why did you pause that campaign?" |
| Self-healing | Auto-recover from API errors, token expiry, rate limits |
| Cross-account learning | Patterns learned in one account apply to similar accounts |
| Client portal | Web UI for clients to view their campaign performance |

---

## Risk Register

| Risk | Likelihood | Impact | Mitigation |
|------|-----------|--------|------------|
| LLM costs explode | Medium | High | Free-tier default, cost caps, cheaper model routing |
| API rate limits block runs | Medium | High | Exponential backoff, circuit breakers, staggered scheduling |
| Safety gates too permissive | Low | Critical | Deterministic gates, human review in hybrid mode, audit everything |
| Dr. Fate slower than legacy | Medium | Medium | Benchmark early, optimize hot paths, cache aggressively |
| Team resistance to new system | Low | Medium | Run parallel, prove value, incremental cutover |
| Data loss during migration | Low | Critical | Immutable audit logs, backup before cutover, rollback plan |

---

## Success Metrics

| Metric | Baseline (Legacy) | Target (Dr. Fate) |
|--------|-------------------|-------------------|
| Daily run time | ~8 min | <5 min |
| Decision quality (CPL improvement) | -15% avg | -20% avg |
| False positive rate (bad negatives) | ~5% | <2% |
| Time to onboard new client | 2-3 days | <1 hour |
| Time to debug a failed run | 30-60 min | <10 min |
| Test coverage | 0% | >70% |
| Human interventions per week | ~10 | <5 |

---

## Current Status

🚧 **Phase 0: Foundation**

- [x] Primer documents written
- [ ] Schemas defined
- [ ] Memory skeleton
- [ ] Orchestrator skeleton
- [ ] Executor skeleton
- [ ] GitHub repos created
- [ ] CI configured
