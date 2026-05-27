# Dr. Fate API Contracts

> *"Programs must communicate through well-defined interfaces. The interface is the contract. The contract is law."*

## Versioning

All APIs are versioned via URL path: `/v1/...`. Breaking changes bump to `/v2/...`. All pillars must support at least the previous major version for 90 days.

---

## 1. Memory API (`dr-fate-memory`)

**Base URL:** `http://memory:8000/v1`

### `POST /ingest`

Store a document, decision, outcome, or any text into memory.

**Request:**
```json
{
  "source": "orchestrator",
  "type": "decision",
  "content": "Add negative keyword 'free office' to Bangalore Brand campaign. Reasoning: 234 clicks, 0 conversions, ₹12,400 spent.",
  "metadata": {
    "account_id": "smartworks-india",
    "campaign_id": "123456789",
    "run_id": "run_abc123"
  },
  "tags": ["smartworks-india", "google-ads", "negative-keyword", "bangalore"],
  "expires_at": "2026-12-31T00:00:00Z"
}
```

**Response:**
```json
{
  "memory_id": "mem_123e4567",
  "status": "stored",
  "chunks": 3,
  "indexed_at": "2026-05-28T09:00:01Z"
}
```

---

### `POST /query`

Hybrid RAG search: FTS5 + vector + optional graph boost.

**Request:**
```json
{
  "query": "What negative keywords worked best in Bangalore?",
  "filters": {
    "account_id": "smartworks-india",
    "tags": ["negative-keyword", "bangalore"],
    "type": "outcome"
  },
  "mode": "hybrid",
  "top_k": 10,
  "min_score": 0.6
}
```

**Response:**
```json
{
  "results": [
    {
      "memory_id": "mem_abc",
      "content": "Negating 'free office' in Bangalore Brand reduced CPL by 34%...",
      "score": 0.92,
      "sources": ["fts5", "embed"],
      "metadata": { ... }
    }
  ],
  "total_found": 47,
  "query_time_ms": 45
}
```

**Modes:**
- `fts5` — Keyword search only (fastest, ~10ms)
- `embed` — Semantic search only (moderate, ~50ms)
- `hybrid` — Both, re-ranked (best quality, ~60ms)
- `graph` — Relationship traversal (for connected entities)

---

### `POST /synthesize`

Distill patterns from query results into a concise learning.

**Request:**
```json
{
  "query": "What negative keywords worked best in Bangalore?",
  "results_limit": 20,
  "max_length": 200,
  "style": "bullet_points"
}
```

**Response:**
```json
{
  "synthesis": "• 'Free office' → -34% CPL (3 campaigns, 14 days)\n• 'Cheap coworking' → -28% CPL (2 campaigns, 10 days)\n• '1 day pass' → -19% CPL but also reduced volume 12%",
  "confidence": 0.87,
  "sources": ["mem_abc", "mem_def", "mem_ghi"],
  "model": "gpt-4.1-nano"
}
```

---

### `POST /observe`

Stream event ingestion for real-time memory updates.

**Request:**
```json
{
  "events": [
    {
      "type": "CAMPAIGN_PAUSED",
      "account_id": "smartworks-india",
      "payload": { "campaign_id": "123", "reason": "overspend" }
    }
  ]
}
```

**Response:**
```json
{
  "ingested": 1,
  "errors": 0
}
```

---

### `GET /proc/stats`

Live introspection.

**Response:**
```json
{
  "total_memories": 15234,
  "total_chunks": 45201,
  "index_size_mb": 128,
  "last_ingestion": "2026-05-28T09:00:01Z",
  "queries_last_hour": 342
}
```

---

## 2. Orchestrator API (`dr-fate-orchestrator`)

**Base URL:** `http://orchestrator:8080/v1`

### `POST /workflows/{name}/trigger`

Trigger a workflow for an account.

**Request:**
```json
{
  "account_id": "smartworks-india",
  "mode": "observe",
  "context_override": {}
}
```

**Response:**
```json
{
  "run_id": "run_abc123",
  "status": "pending",
  "queued_at": "2026-05-28T09:00:00Z",
  "estimated_start": "2026-05-28T09:00:05Z"
}
```

---

### `GET /runs/{id}`

Get full run state.

**Response:**
```json
{
  "id": "run_abc123",
  "account_id": "smartworks-india",
  "workflow": "google-ads-daily-agent",
  "status": "completed",
  "mode": "observe",
  "started_at": "2026-05-28T09:00:01Z",
  "completed_at": "2026-05-28T09:02:34Z",
  "context_tokens": 4200,
  "model": "gpt-4.1-nano",
  "cost_usd": 0.003,
  "decisions": [
    {
      "id": "dec_001",
      "type": "NEGATE",
      "target": "campaign_123",
      "payload": { "keyword": "free office", "match_type": "PHRASE" },
      "confidence": "high",
      "reasoning": "234 clicks, 0 conversions, ₹12,400 spent over 14 days.",
      "safety_checks": [
        { "name": "protected_terms", "passed": true },
        { "name": "budget_band", "passed": true },
        { "name": "dedup_30d", "passed": true }
      ]
    }
  ],
  "actions": [
    {
      "id": "act_001",
      "decision_id": "dec_001",
      "status": "proposed",
      "proposed_at": "2026-05-28T09:01:15Z"
    }
  ],
  "outcomes": [],
  "error": null
}
```

---

### `POST /runs/{id}/feedback`

Inject human feedback on a run.

**Request:**
```json
{
  "feedback": "Don't negate 'free office' — it's a branded term for a competitor campaign.",
  "type": "correction",
  "author": "dan@dan.marketing"
}
```

**Response:**
```json
{
  "feedback_id": "fb_001",
  "status": "stored",
  "memory_id": "mem_feedback_001"
}
```

---

### `GET /accounts`

List all configured accounts.

**Response:**
```json
{
  "accounts": [
    {
      "id": "smartworks-india",
      "name": "Smartworks India",
      "currency": "INR",
      "timezone": "Asia/Kolkata",
      "mode": "observe",
      "workflows": ["google-ads-daily-agent", "meta-ads-daily-agent"]
    },
    {
      "id": "workstudio-singapore",
      "name": "Workstudio Singapore",
      "currency": "SGD",
      "timezone": "Asia/Singapore",
      "mode": "observe",
      "workflows": ["meta-ads-daily-agent"]
    }
  ]
}
```

---

### `GET /proc/running`

Currently running workflows.

**Response:**
```json
{
  "running": [
    {
      "run_id": "run_xyz789",
      "workflow": "google-ads-daily-agent",
      "account_id": "smartworks-india",
      "started_at": "2026-05-28T10:30:01Z",
      "phase": "executing",
      "progress_pct": 67
    }
  ]
}
```

---

### `GET /proc/schedule`

Next scheduled runs.

**Response:**
```json
{
  "next_runs": [
    {
      "workflow": "google-ads-daily-agent",
      "account_id": "smartworks-india",
      "scheduled_at": "2026-05-28T10:30:00Z",
      "cron": "30 3,10 * * *"
    }
  ]
}
```

---

## 3. Executor API (`dr-fate-executor`)

**Base URL:** `http://executor:9000/v1`

### `POST /validate`

Validate a command without executing.

**Request:**
```json
{
  "account_id": "smartworks-india",
  "tool": "gads_add_negative_keyword",
  "args": {
    "campaign_id": "123456789",
    "keyword": "free office",
    "match_type": "PHRASE"
  }
}
```

**Response:**
```json
{
  "valid": true,
  "dry_run": true,
  "gates": [
    { "name": "schema", "passed": true },
    { "name": "permissions", "passed": true },
    { "name": "rate_limit", "passed": true },
    { "name": "budget_safety", "passed": true, "note": "Within 20% band" }
  ],
  "estimated_impact": {
    "clicks_change_pct": -5.2,
    "cost_change_inr": -12400
  }
}
```

---

### `POST /execute`

Execute a validated command.

**Request:**
```json
{
  "account_id": "smartworks-india",
  "tool": "gads_add_negative_keyword",
  "args": {
    "campaign_id": "123456789",
    "keyword": "free office",
    "match_type": "PHRASE"
  },
  "dry_run": false,
  "correlation_id": "run_abc123"
}
```

**Response (success):**
```json
{
  "command_id": "cmd_001",
  "status": "success",
  "executed_at": "2026-05-28T09:00:07Z",
  "duration_ms": 245,
  "result": {
    "mutation_id": "mut_abc",
    "campaign_id": "123456789",
    "keyword": "free office",
    "match_type": "PHRASE"
  },
  "audit_url": "http://executor:9000/v1/audit/cmd_001"
}
```

**Response (failure):**
```json
{
  "command_id": "cmd_001",
  "status": "failed",
  "executed_at": "2026-05-28T09:00:07Z",
  "duration_ms": 120,
  "error": {
    "code": "RATE_LIMITED",
    "message": "Google Ads API rate limit exceeded. Retry after 60s.",
    "retryable": true,
    "retry_after": 60
  }
}
```

---

### `GET /audit`

Query the audit log.

**Query params:**
- `limit` — max results (default 50, max 1000)
- `offset` — pagination offset
- `tool` — filter by tool name
- `account_id` — filter by account
- `since` — ISO datetime
- `status` — `success`, `failed`, `pending`
- `mutations_only` — only live mutations (not dry-runs)

**Response:**
```json
{
  "commands": [
    {
      "id": "cmd_001",
      "tool": "gads_add_negative_keyword",
      "account_id": "smartworks-india",
      "status": "success",
      "dry_run": false,
      "executed_at": "2026-05-28T09:00:07Z",
      "duration_ms": 245,
      "correlation_id": "run_abc123"
    }
  ],
  "total": 15234,
  "limit": 50,
  "offset": 0
}
```

---

### `GET /tools`

List all available tools with schemas.

**Response:**
```json
{
  "tools": [
    {
      "name": "gads_add_negative_keyword",
      "description": "Add a negative keyword to a campaign",
      "platform": "google-ads",
      "is_mutation": true,
      "schema": {
        "type": "object",
        "required": ["campaign_id", "keyword", "match_type"],
        "properties": { ... }
      }
    }
  ]
}
```

---

### `GET /proc/rate-limits`

Current rate limit status per platform.

**Response:**
```json
{
  "google-ads": {
    "requests_remaining": 845,
    "resets_at": "2026-05-28T10:00:00Z",
    "status": "healthy"
  },
  "meta-ads": {
    "requests_remaining": 198,
    "resets_at": "2026-05-28T10:00:00Z",
    "status": "healthy"
  }
}
```

---

## 4. Inter-Service Communication Flows

### Flow A: Daily Autonomous Run

```
Cron ──► Orchestrator.trigger()
           │
           ├──► Memory.query("What do we know?")
           │      └──► [context chunks]
           │
           ├──► LLM.decide(context)
           │      └──► [decisions]
           │
           ├──► Orchestrator.validate(decisions)
           │      └──► [safe decisions]
           │
           ├──► Executor.validate(cmd)
           │      └──► {valid: true}
           │
           ├──► Executor.execute(cmd)
           │      └──► {status: "success", mutation_id: "mut_123"}
           │
           ├──► Memory.ingest(outcome)
           │      └──► {memory_id: "mem_456"}
           │
           └──► Orchestrator.report()
                  └──► Email sent
```

### Flow B: Human Feedback

```
User ──► Orchestrator.feedback(run_id, "Don't do this")
           │
           ├──► Memory.ingest(feedback)
           │      └──► Tagged for next run's context
           │
           └──► {status: "stored"}
```

### Flow C: Outcome Learning (7 days later)

```
Orchestrator ──► Measure outcomes for run_abc123
                   │
                   ├──► Query before/after metrics
                   │      └──► [metrics]
                   │
                   ├──► Memory.synthesize(results)
                   │      └──► "Negating X reduced CPL by Y%"
                   │
                   ├──► Memory.ingest(synthesis)
                   │      └──► {memory_id: "mem_789"}
                   │
                   └──► Update knowledge base
```

---

## 5. Error Codes

All services use these standard error codes:

| Code | HTTP | Meaning | Retryable |
|------|------|---------|-----------|
| `VALIDATION_ERROR` | 400 | Bad request format | No |
| `UNAUTHORIZED` | 401 | Missing/invalid credentials | No |
| `FORBIDDEN` | 403 | Insufficient permissions | No |
| `NOT_FOUND` | 404 | Resource doesn't exist | No |
| `RATE_LIMITED` | 429 | Too many requests | Yes |
| `TIMEOUT` | 504 | Upstream API timeout | Yes |
| `UPSTREAM_ERROR` | 502 | Google/Meta/LinkedIn API error | Yes |
| `SAFETY_BLOCKED` | 422 | Command failed safety gate | No |
| `CIRCUIT_OPEN` | 503 | Circuit breaker triggered | Yes (after cooldown) |
| `INTERNAL_ERROR` | 500 | Unexpected server error | Yes |

---

## 6. Schema Definitions

All schemas are defined in JSON Schema and generated TypeScript types live in `schemas/`:

```
schemas/
├── message.json          # The envelope
├── memory.json           # Memory data model
├── run.json              # Run data model
├── decision.json         # Decision data model
├── command.json          # Command data model
├── account.json          # Account config schema
└── generated/
    ├── types.ts          # TypeScript types
    └── types.py          # Python dataclasses
```

**Single source of truth:** `schemas/*.json`. All code generates types from these.
