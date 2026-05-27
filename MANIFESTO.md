# The Dr. Fate Linux Manifesto

> *"Unix is simple. It just takes a genius to understand its simplicity."*
> — Dennis Ritchie

## Why Linux?

Not Linux the kernel. Linux the **philosophy**:
- Small programs that do one thing well
- Text streams as the universal interface
- Filesystems as the source of truth
- Configuration in plain text
- Composition through pipes
- Fail loudly, recover gracefully

We didn't copy Linux. We **translated** it.

---

## The Translation Table

| Unix / Linux | Dr. Fate | Why It Matters |
|-------------|----------|----------------|
| `cat file` | `memory.query(id)` | Read knowledge. Simple. Direct. |
| `grep pattern` | `memory.query(q, mode=fts5)` | Find what you need. Fast. Precise. |
| `sort | uniq -c` | `memory.synthesize(results)` | Distill patterns from noise. |
| `cron` | `orchestrator.schedule()` | Run things on time. Reliably. |
| `make` | `orchestrator.workflow()` | DAG of dependent steps. Declarative. |
| `sudo` | `executor.execute(EXECUTE=1)` | Elevated privileges. Explicit. Audited. |
| `script` | `executor.audit.log` | Record everything. Immutable. |
| `/var/log` | `fate_stream` | Append-only event log. Replayable. |
| `/proc` | `/proc` endpoints | Live introspection. No magic. |
| `/etc` | `/etc/dr-fate/*.yaml` | Config in text. Version-controlled. |
| `sh` | `fate` CLI | The glue. Thin. Composable. |
| `ps` | `fate status` | What's running? Right now. |
| `tail -f` | `fate tail --follow` | Watch events stream by. |
| `|` (pipe) | `fate plan \| fate decide \| fate execute` | Compose workflows from simple steps. |
| `&` (background) | `orchestrator.trigger(async=true)` | Fire and forget. Check later. |
| `>` (redirect) | `fate run > report.md` | Output goes wherever you want. |
| `2>` (stderr) | `fate_stream type=ERROR` | Errors are a separate stream. |
| `chmod` | Safety gates | Permissions enforced at runtime. |
| `chroot` | `account_id` isolation | Tenants can't see each other. |

---

## The Three Binaries

Like `ls`, `grep`, and `awk` — three tools, infinite combinations:

```bash
# Memory: the grep of knowledge
fate memory query "Bangalore negative keywords" --top-k 10

# Orchestrator: the make of decisions
fate orchestrator trigger --workflow=google-ads-daily --account=sw-india

# Executor: the sudo of APIs
fate executor execute --tool=gads_add_negative --arg term="free office"
```

**Piped together:**
```bash
fate memory query "high spend zero conversion" \
  | fate orchestrator decide --account=sw-india \
  | fate executor validate \
  | fate executor execute
```

---

## What We Rejected

| Bad Idea | Why | Linux Equivalent |
|----------|-----|-----------------|
| Monolithic binary | Can't test, can't replace, can't reason about | `emacs` vs `ed` + `grep` + `awk` |
| Binary protocols | Can't debug with `curl`, can't read with `cat` | Protobuf vs JSON |
| Hidden state | Can't `ls` it, can't `grep` it | Registry vs `/etc` |
| Magic frameworks | Too much abstraction, not enough control | SystemD vs `cron` + `init` |
| ORMs | SQL is the API. Don't hide it. | ActiveRecord vs `sqlite3` CLI |
| Web dashboards | `curl` is the UI. Build dashboards on top. | cPanel vs `apachectl` |

---

## The Filesystem Is the API

```
/etc/dr-fate/              ← Configuration (YAML)
  accounts/
    smartworks-india.yaml
    workstudio-singapore.yaml
  workflows/
    google-ads-daily.yaml
    meta-ads-daily.yaml

/var/dr-fate/              ← Runtime state (SQLite)
  memory/
    knowledge.db
    embeddings.db
  orchestrator/
    state.db
    schedule.db
  executor/
    audit.db
    dead-letter.db

/proc/dr-fate/             ← Live introspection (HTTP)
  memory/stats
  orchestrator/running
  executor/rate-limits
```

You can `cat`, `ls`, `grep`, `sqlite3` every piece of state. No hidden databases. No proprietary formats.

---

## Failure Is Data

In Linux, a failed command returns non-zero and prints to stderr. In Dr. Fate:

```json
{
  "type": "COMMAND_FAILED",
  "timestamp": "2026-05-28T09:00:07Z",
  "source": "executor",
  "error": {
    "code": "RATE_LIMITED",
    "message": "Google Ads API rate limit exceeded",
    "retryable": true,
    "retry_after": 60
  },
  "payload": { ... }
}
```

**Every failure is an event in the stream.** You can `grep RATE_LIMITED | wc -l` to count them. You can `jq '.error.retryable'` to filter retryable ones. You don't need a special dashboard. You need `jq`.

---

## The Shell

`fate` is `/bin/sh` for marketing ops:

```bash
# Interactive
$ fate
fate> memory query "MQL definition"
fate> orchestrator status --account=smartworks-india
fate> executor audit --since=yesterday

# Scripting
$ cat daily.sh
#!/usr/bin/env fate
set -e
memory query "context for $ACCOUNT" \
  | orchestrator decide --account=$ACCOUNT \
  | executor validate \
  | executor execute
orchestrator report --account=$ACCOUNT

# One-liners
$ fate memory query "Bangalore" | jq '.results[].score'
$ fate orchestrator runs --account=sw-india | jq '.[].status'
$ fate executor audit --tool=gads_add_negative | jq '.[].duration_ms'
```

---

## Portability

A Linux script written in 1995 still runs today. Dr. Fate aims for the same:

- **Language:** Python 3.12+ and Node 22+ — both LTS, both ubiquitous
- **Storage:** SQLite — the most deployed database in the world
- **Protocol:** HTTP + JSON — understood by every language, every tool
- **Config:** YAML — human-readable, machine-parseable, git-friendly

In 2036, you'll still be able to `sqlite3 memory.db "SELECT * FROM memories"`.

---

## What This Is Not

- **Not microservices.** Three binaries on one machine is fine. Split when you need to.
- **Not Kubernetes.** `systemd` or `docker-compose` is enough. K8s is a last resort.
- **Not event-driven.** Events are logged, not queued. Polling is fine. Simplicity > scale.
- **Not serverless.** Long-running processes with local state. Functions are for stateless work.
- **Not SaaS.** Self-hosted. Your data, your machine, your control.

---

## The Ten Commandments

1. **Thou shalt do one thing well.** Memory stores. Orchestrator decides. Executor acts.
2. **Thou shalt speak JSON.** Text is the universal interface.
3. **Thou shalt store in SQLite.** Files are the universal database.
4. **Thou shalt config in YAML.** Plain text, version-controlled, human-readable.
5. **Thou shalt log to streams.** Every event is a line of JSON. Append-only.
6. **Thou shalt expose /proc.** Live introspection. No black boxes.
7. **Thou shalt fail loudly.** Non-zero exits. Structured errors. No swallowed exceptions.
8. **Thou shalt compose through pipes.** Small steps, chained together.
9. **Thou shalt isolate by account.** `account_id` is the chroot.
10. **Thou shalt not optimize prematurely.** Working > perfect. Measure, then optimize.

---

## The Goal

A system so simple that a new developer can understand it in a day. So transparent that a manager can audit it with `grep`. So reliable that it runs for years without intervention.

**This is the Unix way. This is Dr. Fate.**
