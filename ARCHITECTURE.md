# Dr. Fate Architecture

> *"The system is the message. The message is the system."*

## Overview

Dr. Fate is three autonomous services that communicate through well-defined JSON messages over HTTP. Each service is a standalone binary with its own storage, its own lifecycle, and its own single responsibility.

```
┌─────────────────────────────────────────────────────────────────────┐
│                           The User / CLI                             │
│                         (fate, cron, webhook)                        │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                    ┌───────────────┼───────────────┐
                    ▼               ▼               ▼
           ┌─────────────┐  ┌─────────────┐  ┌─────────────┐
           │   Memory    │  │ Orchestrator│  │  Executor   │
           │  (Python)   │◄─┤  (TypeScript)├──►│ (TypeScript)│
           │   :8000     │  │   :8080     │  │   :9000     │
           └──────┬──────┘  └──────┬──────┘  └──────┬──────┘
                  │                │                │
           ┌──────┴──────┐  ┌──────┴──────┐  ┌──────┴──────┐
           │  SQLite/    │  │   Turso/    │  │   SQLite    │
           │  FTS5/      │  │   SQLite    │  │  (audit)    │
           │  Vector     │  │  (state)    │  │             │
           └─────────────┘  └─────────────┘  └─────────────┘
```

**All communication is REST + JSON.** No gRPC, no GraphQL, no WebSockets (except SSE for streaming). Simple, debuggable, universal.

---

## The Three Pillars

### 1. Memory (`dr-fate-memory`)

**Port:** `8000`
**Storage:** SQLite (`memory.db`) with FTS5 + vector extensions
**Language:** Python 3.12+
**Framework:** FastAPI

**What it does:**
- **Ingest:** Accept documents, chunks, tags them, stores raw + embeddings
- **Retrieve:** Hybrid search (FTS5 keyword + vector semantic + graph relational)
- **Synthesize:** Distill patterns from retrieved context using LLM
- **Observe:** Watch event streams and auto-update relevant indexes

**Core API:**

```
POST /v1/ingest           ← Store a document/memory
GET  /v1/query            ← Hybrid RAG search
POST /v1/synthesize       ← Distill learnings from results
GET  /v1/graph/{entity}   ← Entity relationship query
POST /v1/observe          ← Stream event ingestion
GET  /v1/proc/stats       ← Live introspection
```

**Data model:**

```python
class Memory:
    id: str                    # UUID
    source: str                # Where it came from (e.g., "daily-reporting-v3")
    type: str                  # "document", "decision", "outcome", "feedback"
    content: str               # Raw text
    embedding: list[float]     # 768-dim vector (BAAI/bge-base-en-v1.5)
    metadata: dict             # Arbitrary JSON
    tags: list[str]            # e.g., ["smartworks-india", "google-ads", "negative-keyword"]
    created_at: datetime
    expires_at: datetime       # TTL for ephemeral memories
    confidence: float          # 0.0-1.0, for synthesized memories
```

**Key insight:** Memory doesn't know about marketing. It knows about text, vectors, and relationships. The orchestrator asks "What do we know about X?" and Memory returns relevant context. Memory never initiates.

---

### 2. Orchestrator (`dr-fate-orchestrator`)

**Port:** `8080`
**Storage:** Turso (libSQL) or SQLite (`orchestrator.db`)
**Language:** TypeScript
**Runtime:** Node 22+

**What it does:**
- **Schedule:** Cron + event-driven triggers for workflows
- **Plan:** Build context from Memory, call LLM for decisions
- **Validate:** Safety gates (deterministic pre-checks)
- **Dispatch:** Send validated commands to Executor
- **Learn:** Measure outcomes, synthesize knowledge, feed back to Memory
- **Report:** Generate human-readable summaries

**Core API:**

```
POST /v1/workflows/{name}/trigger  ← Start a workflow
GET  /v1/workflows                 ← List available workflows
GET  /v1/runs/{id}                 ← Run status and results
POST /v1/runs/{id}/feedback        ← Human feedback on a run
GET  /v1/accounts                  ← Multi-tenant account list
GET  /v1/accounts/{id}/config      ← Per-account configuration
POST /v1/decisions                 ← Direct decision request (for testing)
GET  /v1/proc/running              ← Currently running workflows
GET  /v1/proc/schedule             ← Next scheduled runs
```

**Data model:**

```typescript
interface Run {
  id: string;
  account_id: string;
  workflow: string;           // e.g., "google-ads-daily-agent"
  status: "pending" | "planning" | "deciding" | "executing" | "learning" | "completed" | "failed";
  mode: "observe" | "hybrid" | "execute";
  started_at: Date;
  completed_at?: Date;
  context_tokens: number;
  decisions: Decision[];
  actions: Action[];
  outcomes: Outcome[];
  model: string;              // Which LLM was used
  cost_usd: number;
}

interface Decision {
  id: string;
  run_id: string;
  type: "NEGATE" | "BUDGET" | "PAUSE" | "RESUME" | "CREATE" | "NOOP";
  target: string;             // Campaign ID, keyword, etc.
  payload: Record<string, unknown>;
  confidence: "low" | "medium" | "high";
  reasoning: string;
  safety_checks: SafetyResult[];
}

interface Action {
  id: string;
  decision_id: string;
  status: "proposed" | "validated" | "executed" | "failed" | "rejected";
  executed_at?: Date;
  mutation_id?: string;       // Executor's reference
  error?: string;
}

interface Outcome {
  id: string;
  action_id: string;
  metric: string;             // "ctr", "cpl", "conversion_rate"
  before: number;
  after: number;
  delta_pct: number;
  measured_at: Date;
  horizon_days: number;
}
```

**Key insight:** The orchestrator is the brain but has no hands. It can read Memory and write decisions, but it cannot touch live APIs. That power belongs to the Executor. This separation means a compromised orchestrator can't damage campaigns — it can only propose bad ideas, which the Executor's safety gates catch.

---

### 3. Executor (`dr-fate-executor`)

**Port:** `9000` (REST API) + `stdio` (MCP transport)
**Storage:** SQLite (`executor.db`) — append-only audit log
**Language:** TypeScript
**Runtime:** Node 22+
**Protocol:** MCP (Model Context Protocol) + REST

**What it does:**
- **Validate:** Re-validate commands against live API state (TOCTOU protection)
- **Execute:** Call Google Ads API, Meta Marketing API, LinkedIn API
- **Audit:** Log every command + result immutably
- **Gate:** `EXECUTE=0` → dry-run only. `EXECUTE=1` → live mutations.
- **Retry:** Exponential backoff with jitter, respects rate limits
- **Report:** Return structured results to Orchestrator

**Core API (REST):**

```
POST /v1/validate          ← Validate a command without executing
POST /v1/execute           ← Execute a validated command
GET  /v1/audit             ← Query audit log
GET  /v1/tools             ← List available tools
GET  /v1/tools/{name}      ← Tool schema (for MCP)
GET  /v1/proc/queue        ← Pending commands
GET  /v1/proc/rate-limits  ← Current rate limit status
```

**MCP Tools exposed:**

```
gads_gaql_search
gads_list_campaigns
gads_add_negative_keyword
gads_update_budget
gads_pause_campaign
gads_resume_campaign
meta_list_campaigns
meta_update_campaign
meta_pause_adset
meta_create_campaign
linkedin_list_campaigns
linkedin_update_campaign
```

**Data model:**

```typescript
interface Command {
  id: string;
  source: string;             // "orchestrator", "cli", "mcp"
  tool: string;
  args: Record<string, unknown>;
  dry_run: boolean;
  validated_at?: Date;
  validation_result?: ValidationResult;
  executed_at?: Date;
  result?: Record<string, unknown>;
  error?: string;
  duration_ms: number;
}

interface ValidationResult {
  passed: boolean;
  gates: GateResult[];
}

interface GateResult {
  name: string;
  passed: boolean;
  reason?: string;
}
```

**Key insight:** The executor is the only pillar with live API credentials. It treats every command as potentially destructive. Even in `EXECUTE=1` mode, it validates twice: once at receipt, once immediately before mutation (TOCTOU). The audit log is append-only — no UPDATE, no DELETE.

---

## The Message Protocol

All inter-service messages follow this envelope:

```json
{
  "msg_id": "msg_123e4567-e89b-12d3-a456-426614174000",
  "timestamp": "2026-05-28T09:00:01Z",
  "source": "orchestrator",
  "target": "executor",
  "type": "COMMAND_REQUEST",
  "version": "1.0",
  "correlation_id": "run_abc123",
  "payload": { ... }
}
```

**Types:**
- `QUERY_REQUEST / QUERY_RESPONSE` — Memory queries
- `DECISION_PROPOSAL / DECISION_ACK` — Orchestrator → self
- `COMMAND_REQUEST / COMMAND_RESPONSE` — Orchestrator → Executor
- `AUDIT_LOG / AUDIT_ACK` — Executor → Memory (for observation)
- `SCHEDULE_TRIGGER / SCHEDULE_ACK` — Cron → Orchestrator
- `FEEDBACK_INGEST / FEEDBACK_ACK` — User → Orchestrator → Memory

---

## The Fate Stream

All three pillars write to a shared **Fate Stream** — an append-only log of every significant event. This is the system's `/var/log`:

```
# Local mode: SQLite table
# Distributed mode: Kafka / Redpanda / NATS

CREATE TABLE fate_stream (
  seq INTEGER PRIMARY KEY AUTOINCREMENT,
  timestamp TEXT NOT NULL,
  source TEXT NOT NULL,
  type TEXT NOT NULL,
  account_id TEXT,
  correlation_id TEXT,
  payload TEXT NOT NULL
);
```

**Why?** You can replay any day's decisions by `SELECT * FROM fate_stream WHERE date(timestamp) = '2026-05-28'`. You can debug by correlating across pillars using `correlation_id`. You can build dashboards by aggregating `type` counts.

---

## Multi-Tenancy

**Unix analogy:** Linux isolates processes via namespaces and cgroups. Dr. Fate isolates tenants via `account_id`.

Every table, every message, every config stanza carries `account_id`. There is no "default" account. There is no global state that isn't namespaced.

```yaml
# /etc/dr-fate/accounts/smartworks-india.yaml
id: smartworks-india
customer_id: "123-456-7890"
currency: INR
timezone: Asia/Kolkata
platforms:
  google-ads:
    customer_id: "123-456-7890"
    login_customer_id: "987-654-3210"
  meta-ads:
    account_id: "act_123456789"
    token_env: META_SMARTWORKS_TOKEN
workflows:
  - name: google-ads-daily-agent
    cron: "30 3,10 * * *"
    mode: observe
  - name: meta-ads-daily-agent
    cron: "30 3,10 * * *"
    mode: observe
```

Adding a new client = creating one YAML file. No code change. No database migration. The orchestrator discovers accounts by reading `/etc/dr-fate/accounts/*.yaml` at startup and on SIGHUP.

---

## Deployment Models

### Local Development (Single Process)

```bash
# All three pillars + SQLite backends
fate up --dev

# Or individually:
python -m dr_fate_memory --port 8000 --db memory.db
npx dr-fate-orchestrator --port 8080 --db orchestrator.db
npx dr-fate-executor --port 9000 --db executor.db
```

### Production (GitHub Actions)

```yaml
# .github/workflows/dr-fate-daily.yml
name: Dr. Fate Daily Run
on:
  schedule:
    - cron: "30 3,10 * * *"
jobs:
  run:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npx fate run --workflow=google-ads-daily-agent --account=smartworks-india
```

### Self-Hosted (Long-Running)

```yaml
# docker-compose.yml (future)
services:
  memory:
    image: dr-fate-memory:latest
    volumes:
      - ./data/memory:/data
  orchestrator:
    image: dr-fate-orchestrator:latest
    environment:
      - MEMORY_URL=http://memory:8000
      - EXECUTOR_URL=http://executor:9000
  executor:
    image: dr-fate-executor:latest
    environment:
      - EXECUTE=1
    volumes:
      - ./data/executor:/data
```

---

## Error Handling

**Unix philosophy:** Fail loudly. Exit non-zero. Print to stderr.

**Dr. Fate:**
- Every API returns HTTP status codes honestly. 500 = our fault. 400 = your fault. 422 = validation failure.
- Every error is logged to the Fate Stream with full context.
- Retries are explicit: `Retry-After` headers, exponential backoff, max attempts.
- Circuit breakers: After 5 consecutive failures to an API, stop trying for 5 minutes.
- No swallowed exceptions. If something breaks, the stream knows.

---

## Security Model

| Layer | Control |
|-------|---------|
| Executor gate | `EXECUTE` env var (0 = dry-run, 1 = live) |
| Account isolation | `account_id` on every request |
| API credentials | Environment variables only, never in code or config |
| Audit immutability | SQLite WAL mode, no UPDATE/DELETE on audit tables |
| LLM calls | Rate-limited, capped spend, no PII in prompts |

---

## Observability

Every pillar exposes `/proc`:

```bash
# System health
curl http://localhost:8080/proc/health

# What's running?
curl http://localhost:8080/proc/running

# Recent errors
curl http://localhost:8080/proc/errors?limit=10

# Rate limits
curl http://localhost:9000/proc/rate-limits

# Index stats
curl http://localhost:8000/proc/index/stats

# Stream tail
fate tail --follow
```

---

## Migration from Legacy

Existing repos (adpilot, centralintelligence, marketing-ops-mcp, gads-mcp) continue running. Dr. Fate is deployed alongside them. Migration is account-by-account, workflow-by-workflow:

1. Deploy Dr. Fate in `observe` mode for one account
2. Run both legacy and Dr. Fate in parallel
3. Compare outputs for 2 weeks
4. Switch account to Dr. Fate `hybrid` mode
5. After 1 month, switch to `execute` mode
6. Repeat for next account

No big-bang migration. No ripping out what works.
