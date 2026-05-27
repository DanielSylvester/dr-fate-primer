# Project Dr. Fate

> *"The three pillars of autonomous marketing operations:*
> *Memory knows everything. The Orchestrator decides everything. The Executor does everything.*
> *Together, they are greater than the sum of their parts."*

## What Is This?

**Project Dr. Fate** is a from-scratch rebuild of autonomous marketing operations, inspired by the Unix philosophy: small, focused tools that compose into something powerful. Three pillars, one mission, zero bloat.

| Pillar | Responsibility | Analogy |
|--------|---------------|---------|
| **Memory** (`dr-fate-memory`) | Knows everything. Stores, retrieves, synthesizes. | The brain's hippocampus + cortex |
| **Orchestrator** (`dr-fate-orchestrator`) | Decides everything. Schedules, reasons, learns. | The prefrontal cortex |
| **Executor** (`dr-fate-executor`) | Does everything. Validates, executes, audits. | The motor cortex |

## Why Rebuild?

The existing stack (adpilot, centralintelligence, marketing-ops-mcp, gads-mcp) works but grew organically. Responsibilities blurred. State scattered across Sheets, Turso, and code. Configuration leaked into implementation. Testing is minimal. Scaling to 40+ companies requires a cleaner foundation.

**Dr. Fate keeps what works, discards what doesn't, and builds with intention.**

## Core Principles (The Unix Way)

1. **Everything is a message** — All state changes flow through a unified message log. No hidden side effects.
2. **Do one thing well** — Memory stores. Orchestrator decides. Executor acts. Never two responsibilities in one binary.
3. **Compose through pipes** — Complex workflows are chains of API calls, not monolithic scripts.
4. **Text is the universal interface** — JSON everywhere. No binary protocols. Human-readable, machine-parseable.
5. **Configuration over code** — Behavior lives in config files. Code is the engine, not the policy.
6. **Fail loudly, recover gracefully** — Every error is logged. Every retry is visible. No silent failures.
7. **Worse is better** — Simple solutions that work beat perfect abstractions that ship never.

## Quick Start

```bash
# Clone all four repos
git clone https://github.com/DanielSylvester/dr-fate-primer.git
git clone https://github.com/DanielSylvester/dr-fate-memory.git
git clone https://github.com/DanielSylvester/dr-fate-orchestrator.git
git clone https://github.com/DanielSylvester/dr-fate-executor.git

# Read the architecture
open dr-fate-primer/ARCHITECTURE.md
open dr-fate-primer/CONTRACTS.md
open dr-fate-primer/ROADMAP.md
```

## Repositories

| Repo | Language | Runtime | Purpose |
|------|----------|---------|---------|
| `dr-fate-primer` | Markdown | — | Architecture, contracts, roadmap |
| `dr-fate-memory` | Python 3.12+ | FastAPI + SQLite/FTS5 + ONNX | Knowledge, RAG, synthesis |
| `dr-fate-orchestrator` | TypeScript | Node 22+ + Turso | Scheduling, reasoning, learning |
| `dr-fate-executor` | TypeScript | Node 22+ (MCP) | Tool execution, safety, audit |

## Status

🚧 **Phase 0: Foundation** — Architecture defined, contracts written, skeleton repos created.

See [ROADMAP.md](./ROADMAP.md) for the full phased plan.

---

*Built for Dan Antony's marketing automation empire. Multi-tenant by design. Client-agnostic by default.*
