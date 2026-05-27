# Dr. Fate Philosophy — The Unix Way Applied to Marketing Ops

> *"This is the Unix philosophy: Write programs that do one thing and do it well. Write programs to work together. Write programs to handle text streams, because that is a universal interface."*
> — Doug McIlroy

---

## 1. Everything Is a Stream

**Unix:** `/proc`, `/dev`, files, pipes — everything looks like a byte stream.

**Dr. Fate:** Everything is an **event stream**. A campaign impression, a lead form submission, a decision to pause a keyword, a learning synthesized from outcomes — all are events in a append-only log.

```
┌─────────────────────────────────────────────────────────┐
│                    The Event Stream                      │
│  (Append-only, immutable, ordered, replayable)          │
├─────────────────────────────────────────────────────────┤
│  2026-05-28T09:00:01Z  RUN_STARTED      {run_id: "r1"} │
│  2026-05-28T09:00:02Z  CONTEXT_BUILT    {tokens: 4200} │
│  2026-05-28T09:00:05Z  DECISION_MADE    {action: "NEGATE", term: "free office"} │
│  2026-05-28T09:00:06Z  SAFETY_PASSED    {gates: ["protected", "budget", "dedup"]} │
│  2026-05-28T09:00:07Z  COMMAND_EXECUTED {status: "SUCCESS", mutation_id: "m1"} │
│  2026-05-28T09:00:08Z  RUN_COMPLETED   {actions: 3, errors: 0} │
└─────────────────────────────────────────────────────────┘
```

**Implication:** You can `tail -f` the stream, `grep` for decisions, `wc -l` for throughput. Debugging is `cat stream | jq '.type == "ERROR"'`. The entire system's history is a text file you can read.

---

## 2. Do One Thing Well

**Unix:** `ls` lists. `grep` filters. `sort` sorts. `awk` processes. Each is perfect at its one job.

**Dr. Fate:**

| Tool | One Thing | Unix Analogy |
|------|-----------|--------------|
| `memory` | Store and retrieve knowledge with context | `grep` + `cat` + `find` for semantic space |
| `orchestrator` | Decide what to do and when | `cron` + `make` for reasoning |
| `executor` | Execute commands safely | `sudo` + `script` + `logger` for APIs |

**Anti-pattern:** A monolithic `adpilot` that fetches data, reasons, executes, emails, and learns all in one process. In Dr. Fate, these are separate binaries that communicate through well-defined streams.

---

## 3. Compose Through Pipes

**Unix:** `cat file | grep pattern | sort | uniq -c | sort -rn` — simple tools, powerful pipelines.

**Dr. Fate:**

```bash
# A complete autonomous cycle, expressed as a pipeline:
orchestrator plan --account=smartworks-india \
  | memory enrich --context=business-goals,competitors \
  | orchestrator decide \
  | executor validate --mode=strict \
  | executor execute --dry-run=false \
  | memory log --type=outcome \
  | orchestrator learn --horizon=7d
```

Each stage reads JSON from stdin, writes JSON to stdout. You can insert `| jq .` anywhere to inspect. You can replace `executor validate` with a stricter version without touching `orchestrator decide`.

---

## 4. Text Is the Universal Interface

**Unix:** All tools read and write text. No binary blobs, no proprietary formats.

**Dr. Fate:**
- **Config:** YAML/JSON files, checked into git
- **State:** SQLite databases (accessible via `sqlite3` CLI)
- **APIs:** JSON over HTTP or NDJSON over SSE
- **Logs:** Structured JSON, one line per event
- **Schemas:** JSON Schema + TypeScript types (generated from the same source)

**You can `curl` any pillar. You can `sqlite3` any database. You can `jq` any response.**

---

## 5. Filesystem as Database

**Unix:** `/proc/123/status`, `/sys/class/net/eth0/statistics`, `/var/log` — the filesystem IS the API.

**Dr. Fate:**

```
/memory/
  /index/          ← FTS5 + vector indexes (SQLite)
  /raw/            ← Original documents, chunked
  /synthesis/      ← LLM-distilled learnings
  /graph/          ← Entity relationships (who mentioned what, when)

/orchestrator/
  /state/
    /runs/         ← Run records (Turso/SQLite)
    /actions/      ← Action history
    /knowledge/    ← Synthesized rules (mirror of memory)
  /schedule/
    /crontab/      ← Time-based triggers (YAML)
    /triggers/     ← Event-based triggers (webhooks)

/executor/
  /audit/
    /commands/     ← Every command executed (SQLite)
    /results/      ← Every result received
  /queue/
    /pending/      ← Commands awaiting execution
    /dead-letter/  ← Failed commands for retry
```

Every state change is a file write. Every query is a file read. You can `ls`, `cat`, `grep` the entire system's state.

---

## 6. Configuration Over Code

**Unix:** `/etc` holds configuration. `/bin` holds code. You change behavior by editing files, not recompiling.

**Dr. Fate:**

```yaml
# /etc/dr-fate/orchestrator.yaml
accounts:
  smartworks-india:
    customer_id: "123-456-7890"
    currency: INR
    timezone: Asia/Kolkata
    mode: observe
    platforms: [google-ads, meta-ads]
    safety:
      budget_band: 0.20
      protected_terms: [managed office, smartworks]
      competitor_terms: [awfis, wework, regus]
    scheduling:
      - cron: "30 3,10 * * *"
        workflow: google-ads-agent
      - cron: "0 9 * * 1"
        workflow: weekly-report
```

Adding a new client = adding a YAML stanza. No code change. No deployment. The orchestrator hot-reloads config.

---

## 7. Worse Is Better

**Unix:** `ed` is worse than a word processor. `sh` is worse than a systems language. But they compose into something greater.

**Dr. Fate:**
- Memory uses SQLite instead of a vector database. Worse at scale. Better at debugging, portability, and "just works."
- Orchestrator uses file polling instead of a message queue. Worse at real-time. Better at observability and recovery.
- Executor uses SQLite for audit instead of a centralized logging system. Worse at aggregation. Better at local debugging and offline replay.

**The right worse solution beats the wrong better solution.**

---

## 8. Permission Model

**Unix:** `rwx` for user, group, other. `sudo` for elevation. Clear boundaries.

**Dr. Fate:**

| Role | Can Read | Can Write | Can Execute |
|------|----------|-----------|-------------|
| Memory | Everything | Knowledge, context | Synthesis |
| Orchestrator | Memory, Executor audit | Decisions, schedules | Planning, learning |
| Executor | Commands | Audit log | API mutations (gated) |

The Executor is the only pillar that can touch live APIs. It requires explicit `EXECUTE=1` (like `sudo`). The Orchestrator proposes but cannot act directly. Memory observes but never initiates.

---

## 9. The Shell — Fate CLI

**Unix:** `sh` is the glue. It doesn't do much itself, but it wires everything together.

**Dr. Fate:** `fate` is the glue.

```bash
# Interactive mode
$ fate
fate> memory query "What did we learn about Bangalore Brand campaign?"
fate> orchestrator status --account=smartworks-india
fate> executor audit --tool=gads_add_negative --since=yesterday

# Scripting mode
$ fate run --workflow=daily-cycle --account=smartworks-india --mode=observe

# Pipeline mode
$ fate plan smartworks-india | fate enrich | fate decide | fate validate | fate execute
```

The CLI is a thin wrapper around HTTP APIs. It doesn't contain business logic. It's the `/bin/sh` of the system.

---

## 10. Live Introspection (/proc)

**Unix:** `ps`, `top`, `lsof`, `strace` — live system introspection without restarting anything.

**Dr. Fate:** Every pillar exposes a `/proc` endpoint:

```bash
# What's the orchestrator doing right now?
curl http://orchestrator:8080/proc/running

# What commands has the executor executed in the last hour?
curl http://executor:8080/proc/audit?since=1h

# What's in memory's working set?
curl http://memory:8080/proc/index/stats

# Live stream of all events
fate tail --follow
```

---

## Summary

| Unix Concept | Dr. Fate Equivalent |
|-------------|---------------------|
| Everything is a file | Everything is a stream/event |
| Small tools, one job | Three pillars, clear boundaries |
| Pipes compose | JSON pipelines compose |
| Text is universal | JSON/NDJSON is universal |
| Filesystem is API | SQLite + HTTP is API |
| /etc configures | YAML configures |
| /proc introspects | `/proc` endpoints introspect |
| `sh` glues | `fate` glues |
| `sudo` elevates | `EXECUTE=1` elevates |
| `man` documents | `primer` documents |

**This is not a metaphor. This is the architecture.**
