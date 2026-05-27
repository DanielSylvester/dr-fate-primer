# Dr. Fate Shared Schemas

Single source of truth for all data models across the three pillars.

## Usage

### TypeScript
```bash
npx json-schema-to-typescript schemas/message.json > types/message.ts
```

### Python
```bash
datamodel-codegen --input schemas/message.json --output models/message.py
```

## Schema List

| Schema | Description |
|--------|-------------|
| `message.json` | Inter-service message envelope |
| `memory.json` | Memory document model |
| `run.json` | Orchestrator run model |
| `decision.json` | Decision model |
| `action.json` | Action model |
| `outcome.json` | Outcome model |
| `command.json` | Executor command model |
| `account.json` | Account configuration |
| `safety.json` | Safety gate results |
