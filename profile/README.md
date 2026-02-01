# rstmdb

**A deterministic, versioned state machine database with WAL-backed durability**

---

## The Problem

Modern systems cannot replay or audit semantic state evolution.

When a payment fails, an order gets stuck, or a workflow breaks — teams dig through scattered logs, reconstruct timelines manually, and hope they find the root cause. State changes are buried in application code, spread across services, and lost to time.

**The consequences:**

- Auditors can't verify what happened and when
- Compliance teams can't prove process adherence
- Engineers can't reproduce bugs from production
- Recovery from failures requires guesswork

---

## What is rstmdb?

rstmdb is a **deterministic, versioned state machine database** with a Write-Ahead Log.

You define state machines declaratively. rstmdb persists every transition, enforces guard conditions, and maintains a complete, replayable history of how each entity evolved.

```json
{
  "states": ["pending", "approved", "rejected"],
  "initial": "pending",
  "transitions": [
    {
      "from": "pending",
      "event": "APPROVE",
      "to": "approved",
      "guard": "ctx.score > 80"
    },
    { "from": "pending", "event": "REJECT", "to": "rejected" }
  ]
}
```

Every state change is **durable**, **versioned**, and **replayable**.

---

## Why It Matters

### Audits

Every state transition is recorded with full context. Answer "what happened to entity X?" instantly, with cryptographic proof.

### Compliance

Enforce business rules at the database level. Guard conditions ensure transitions only happen when allowed. The audit trail is built-in.

### Reproducibility

Replay any entity's history from the WAL. Reproduce bugs, verify behavior, test migrations — all from production data.

### Infrastructure Resilience

Crash recovery is automatic. The WAL ensures no acknowledged write is ever lost. Deterministic replay rebuilds state exactly.

---

## How It Works

### State Machines

Define allowed states and transitions using a simple JSON DSL. Add guard expressions to enforce business rules. Version your definitions for safe evolution.

### Write-Ahead Log (WAL)

Every mutation is written to a durable, append-only log before acknowledgment. Segment-based storage with CRC32C checksums ensures integrity.

### Replay

On startup, rstmdb replays the WAL to reconstruct state. Any point in history can be materialized. Event sourcing is native.

### Versioning

Machine definitions are versioned. Instances are bound to specific versions. Evolve your state machines without breaking existing workflows.

---

## Quick Start

```bash
# Docker
docker run -p 7401:7401 rstmdb/rstmdb:latest

# From source
cargo install rstmdb
rstmdb --config config.yaml
```

---

## Repositories

| Repository                                 | Description                                      |
| ------------------------------------------ | ------------------------------------------------ |
| [rstmdb](https://github.com/rstmdb/rstmdb) | Core server, Rust client, CLI, and documentation |

---

## License & Governance

### BSL 1.1 (Business Source License)

rstmdb uses an **open-core** model with the Business Source License 1.1.

**Free for:**

- Development and testing
- Non-production environments
- Evaluation and research
- Production on a single node

**Commercial license required for:**

- Multi-node production deployments
- Managed service offerings (SaaS/DBaaS)

### Future: Apache 2.0

Each release converts to **Apache 2.0** after **4 years**, becoming fully open source.

We believe in building sustainable open source. BSL lets us invest in the project while keeping the code available and eventually free for everyone.

---

## Links

- [Documentation](https://github.com/rstmdb/rstmdb/tree/main/docs)
- [Architecture](https://github.com/rstmdb/rstmdb/blob/main/docs/architecture.md)
- [Roadmap](https://github.com/rstmdb/rstmdb/blob/main/ROADMAP.md)
- [Contributing](https://github.com/rstmdb/rstmdb/blob/main/CONTRIBUTING.md)

---

<p align="center">
  <i>Deterministic state. Durable history. Auditable by design.</i>
</p>
