<div align="center">

# rstmdb

**The database for things that move through states.**

Orders, payments, approvals, onboarding flows — define the lifecycle once,<br>
and rstmdb guarantees every entity follows it, durably, with a full replayable history.

[![License: BSL 1.1](https://img.shields.io/badge/license-BSL%201.1-orange)](LICENSE)
[![Built with Rust](https://img.shields.io/badge/built%20with-Rust-b7410e?logo=rust)](https://www.rust-lang.org)
[![Docs](https://img.shields.io/badge/docs-docs.rstmdb.com-blue)](https://docs.rstmdb.com)
[![Grafana Dashboard](https://img.shields.io/badge/Grafana-dashboard%20%2324778-F46800?logo=grafana)](https://grafana.com/grafana/dashboards/24778)

[Quick start](#-try-it-in-60-seconds) · [Docs](https://docs.rstmdb.com) · [Clients](#-client-libraries) · [Roadmap](ROADMAP.md) · [Website](https://rstmdb.com)

</div>

---

## Why rstmdb?

Most teams store workflow state as a `status` column and hope the application code moves it correctly. Then an order gets stuck in `paid` but never `shipped`, a payment is approved twice, and someone spends a day grepping logs to work out what happened.

rstmdb makes the lifecycle a first-class thing in the database:

- **Invalid transitions are impossible.** You declare which events move an entity from which state to which. Anything else is rejected — by the database, not by a code review.
- **Business rules live next to the data.** Guards like `ctx.amount <= 1000` decide whether a transition is allowed.
- **Nothing acknowledged is ever lost.** Every change hits a write-ahead log before you get an OK, and the server recovers automatically after a crash.
- **You can always answer "what happened?".** Each instance has a complete, ordered history you can replay, audit, or stream to other services.

```text
          CONFIRM            SHIP              DELIVER
pending ──────────▶ confirmed ──────────▶ shipped ──────────▶ delivered
   │
   └── CANCEL ──▶ cancelled          (anything else → rejected)
```

---

## ⚡ Try it in 60 seconds

You only need Docker. The image ships with both the server and the `rstmdb-cli` tool.

**1. Start the server**

```bash
docker run -d --name rstmdb -p 7401:7401 rstmdb/rstmdb:latest
```

**2. Define a state machine**

```bash
docker exec rstmdb rstmdb-cli put-machine -n order -v 1 '{
  "states": ["created", "paid", "shipped"],
  "initial": "created",
  "transitions": [
    {"from": "created", "event": "PAY",  "to": "paid"},
    {"from": "paid",    "event": "SHIP", "to": "shipped", "guard": "ctx.ready"}
  ]
}'
```

**3. Create an instance and move it forward**

```bash
docker exec rstmdb rstmdb-cli create-instance -m order -V 1 -i order-001 -c '{"ready": true}'
docker exec rstmdb rstmdb-cli apply-event -i order-001 -e PAY -p '{"amount": 99.99}'
docker exec rstmdb rstmdb-cli get-instance order-001
```

**4. Try to break it**

```bash
# "created" has no SHIP transition — rstmdb refuses it
docker exec rstmdb rstmdb-cli create-instance -m order -V 1 -i order-002
docker exec rstmdb rstmdb-cli apply-event -i order-002 -e SHIP
```

That refusal is the whole point: your data can only move the way you said it could.

> **Next:** connect from your app with one of the [client libraries](#-client-libraries), or read the [full docs](https://docs.rstmdb.com).

<details>
<summary><b>Other ways to install</b></summary>

```bash
# Docker Compose (server + metrics on :9090)
docker compose up -d

# From crates.io
cargo install rstmdb

# From source
git clone https://github.com/rstmdb/rstmdb && cd rstmdb
cargo build --release
./target/release/rstmdb                                  # server
RSTMDB_CONFIG=config.yaml ./target/release/rstmdb        # with a config file
```

</details>

---

## What you get

| | Feature | What it means for you |
|---|---|---|
| 🧭 | **State machine engine** | States, transitions and guard expressions in a small JSON DSL |
| 🏷️ | **Versioned definitions** | Change your workflow without breaking instances already running on the old version |
| 💾 | **Write-ahead log** | Every mutation is persisted (with CRC32C checksums) before it's acknowledged |
| 🗜️ | **Snapshots & compaction** | WAL stays small automatically; restarts stay fast |
| 🔁 | **Idempotency keys** | Retries are safe — the same event won't be applied twice |
| 📦 | **Batch operations** | Several operations applied atomically |
| 📡 | **Event streaming** | Subscribe to changes on one instance or all of them, with filters |
| 🪞 | **Replication** | Stream the WAL to read-only replicas (async or sync) |
| 🧳 | **Backup & restore** | Offline backups plus hot backup on a running server |
| 🔒 | **TLS, mTLS & tokens** | Encrypted transport and hashed-token authentication |
| 📈 | **Prometheus & Grafana** | Metrics, health endpoints and a ready-made dashboard |
| ⚙️ | **Fast binary protocol** | RCP over TCP, built in Rust |

---

## Where it fits

**A great fit when** your entities have a clear lifecycle and correctness matters:

- 🛒 **Order & fulfilment flows** — created → paid → shipped → delivered
- 💳 **Payments & refunds** — no double approvals, a full trail for every transaction
- ✅ **Approvals & KYC** — guard rules like `ctx.amount <= 1000` enforced at the data layer
- 🔔 **Event-driven services** — stream transitions to notifications, audit logs or read models (CQRS)

**Probably not the right tool** for ad-hoc analytics, full-text search, or as a general-purpose document store. Use rstmdb for the lifecycle, and your usual database for everything else.

---

## 🧩 Client libraries

| Language | Install | Repository |
|---|---|---|
| Rust | `cargo add rstmdb-client` | [rstmdb-client](rstmdb-client) |
| Python | `pip install rstmdb` | [rstmdb-py](https://github.com/rstmdb/rstmdb-py) |
| Go | `go get github.com/rstmdb/rstmdb-go` | [rstmdb-go](https://github.com/rstmdb/rstmdb-go) |
| JavaScript / TypeScript | `npm install @rstmdb/client` | [rstmdb-js](https://github.com/rstmdb/rstmdb-js) |
| C# / .NET | `dotnet add package Rstmdb.Client` | [rstmdb-dotnet](https://github.com/rstmdb/rstmdb-dotnet) |
| Java | `com.rstmdb:rstmdb-client` | [rstmdb-java](https://github.com/rstmdb/rstmdb-java) |

---

## 📖 Reference

Everything below is the detailed technical documentation. Click a section to expand it — or browse it all at **[docs.rstmdb.com](https://docs.rstmdb.com)**.

<details>
<summary><b>State machine definition & guard expressions</b></summary>

```json
{
  "states": ["pending", "approved", "rejected"],
  "initial": "pending",
  "transitions": [
    {
      "from": "pending",
      "event": "APPROVE",
      "to": "approved",
      "guard": "ctx.amount <= 1000"
    },
    {
      "from": "pending",
      "event": "REJECT",
      "to": "rejected"
    }
  ]
}
```

Guards are boolean expressions evaluated against the instance context:

```
ctx.amount <= 1000
ctx.approved && ctx.manager_id
ctx.items > 0
!ctx.cancelled
```

</details>

<details>
<summary><b>CLI usage</b></summary>

```bash
# Ping server
rstmdb-cli ping

# Define a state machine
rstmdb-cli put-machine -n order -v 1 '{
  "states": ["created", "paid", "shipped"],
  "initial": "created",
  "transitions": [
    {"from": "created", "event": "PAY", "to": "paid"},
    {"from": "paid", "event": "SHIP", "to": "shipped", "guard": "ctx.ready"}
  ]
}'

# Create instance
rstmdb-cli create-instance -m order -V 1 -i order-001 -c '{"ready": false}'

# Apply event
rstmdb-cli apply-event -i order-001 -e PAY -p '{"amount": 99.99}'

# Get instance
rstmdb-cli get-instance order-001

# Compact WAL
rstmdb-cli compact --force
```

</details>

<details>
<summary><b>Event streaming (WATCH_INSTANCE / WATCH_ALL)</b></summary>

### Watch a specific instance

```bash
rstmdb-cli watch-instance order-001

# Output when state changes:
# {"type":"event","subscription_id":"sub-xxx","instance_id":"order-001",
#  "machine":"order","version":1,"from_state":"created","to_state":"paid",
#  "event":"PAY","wal_offset":42,"payload":{...},"ctx":{...}}
```

### Watch all events

```bash
rstmdb-cli watch-all                                   # everything
rstmdb-cli watch-all --machines order                  # by machine
rstmdb-cli watch-all --to-states shipped               # by target state
rstmdb-cli watch-all --events PAY,SHIP                 # by event
rstmdb-cli watch-all --machines order --to-states shipped
rstmdb-cli watch-all --no-ctx                          # smaller payloads
```

### Recipes

```bash
# Notification service: react to shipped orders
rstmdb-cli watch-all --machines order --to-states shipped

# Audit log: append every event to a file
rstmdb-cli watch-all | jq -c >> audit.jsonl

# CQRS projection: build a read model (track the last processed wal_offset)
rstmdb-cli watch-all --machines order | ./projection-builder
```

### Unsubscribe

```bash
rstmdb-cli unwatch sub-xxxxxxxx
```

</details>

<details>
<summary><b>Configuration file</b></summary>

```yaml
# config.yaml
network:
  bind_addr: "127.0.0.1:7401"
  max_connections: 1000

auth:
  required: false
  # token_hashes:
  #   - "<sha256-hash>"  # Generate with: rstmdb-cli hash-token <token>

storage:
  data_dir: "./data"
  wal_segment_size_mb: 64
  fsync_policy: every_write   # every_write | every_n | every_ms | never

compaction:
  enabled: true
  events_threshold: 10000
  size_threshold_mb: 100
```

More examples in the repo: [`config.example.yaml`](config.example.yaml), [`config.production.yaml`](config.production.yaml).

</details>

<details>
<summary><b>Authentication</b></summary>

When `auth.required: true`, clients must authenticate before executing commands.

```bash
# Generate a token hash for config
rstmdb-cli hash-token "my-secret-token"

# Use token with CLI
rstmdb-cli --token "my-secret-token" list-machines

# Or via environment variable
export RSTMDB_TOKEN="my-secret-token"
rstmdb-cli list-machines
```

Token hashes can be stored in config or loaded from a secrets file:

```yaml
auth:
  required: true
  secrets_file: "/etc/rstmdb/tokens.secret"  # One hash per line
```

</details>

<details>
<summary><b>TLS & mTLS</b></summary>

```yaml
# config.yaml
tls:
  enabled: true
  cert_path: "/path/to/server-cert.pem"
  key_path: "/path/to/server-key.pem"
  # For mutual TLS (mTLS):
  require_client_cert: true
  client_ca_path: "/path/to/client-ca.pem"
```

Generate development certificates:

```bash
./scripts/generate-dev-certs.sh ./dev-certs
```

CLI usage with TLS:

```bash
# Standard TLS (verify server certificate)
rstmdb-cli --tls --ca-cert ./dev-certs/ca-cert.pem -t my-token ping

# Insecure mode (skip verification - dev only)
rstmdb-cli --tls --insecure -t my-token ping

# Mutual TLS (client certificate required)
rstmdb-cli --tls \
    --ca-cert ./dev-certs/ca-cert.pem \
    --client-cert ./dev-certs/client-cert.pem \
    --client-key ./dev-certs/client-key.pem \
    -t my-token ping
```

Environment variables:

```bash
export RSTMDB_TLS=true
export RSTMDB_CA_CERT=/path/to/ca-cert.pem
export RSTMDB_CLIENT_CERT=/path/to/client-cert.pem
export RSTMDB_CLIENT_KEY=/path/to/client-key.pem
```

</details>

<details>
<summary><b>Environment variables</b></summary>

| Variable                   | Default          | Description                    |
| -------------------------- | ---------------- | ------------------------------ |
| `RSTMDB_CONFIG`            | -                | Path to YAML config file       |
| `RSTMDB_BIND`              | `127.0.0.1:7401` | Server bind address            |
| `RSTMDB_DATA`              | `./data`         | Data directory                 |
| `RSTMDB_AUTH_REQUIRED`     | `false`          | Require authentication         |
| `RSTMDB_AUTH_TOKEN_HASH`   | -                | Single token hash              |
| `RSTMDB_AUTH_SECRETS_FILE` | -                | Path to secrets file           |
| `RSTMDB_COMPACT_EVENTS`    | `10000`          | Auto-compact after N events    |
| `RSTMDB_COMPACT_SIZE_MB`   | `100`            | Auto-compact when WAL > N MB   |
| `RSTMDB_TOKEN`             | -                | CLI: Auth token for commands   |

Environment variables override config file values.

</details>

<details>
<summary><b>Replication</b></summary>

The primary streams WAL entries to read-only replicas, in `async` or `sync` mode. Replicas serve reads, reject writes, and catch up by offset.

```yaml
# primary
replication:
  role: primary
  mode: async
  auth_token_hashes:
    - "<sha256-hash>"   # rstmdb-cli hash-token <token>
```

```yaml
# replica
replication:
  role: replica
  upstream: "primary-host:7401"
  auth_token: "<token>"
  max_lag_entries: 10000
  max_lag_seconds: 30
```

Full examples: [`config.primary.yaml`](config.primary.yaml), [`config.replica1.yaml`](config.replica1.yaml), [`docker-compose.replication.yaml`](docker-compose.replication.yaml).

</details>

<details>
<summary><b>Backup & restore</b></summary>

- **Offline:** `rstmdb backup`, `rstmdb restore` and `rstmdb verify` with the `.rstmbak` format (stop the server first).
- **Hot backup:** `BACKUP_BEGIN`, `BACKUP_CHUNK` and `BACKUP_END` protocol operations on a running server.

Run `rstmdb backup --help` for options.

</details>

<details>
<summary><b>Monitoring</b></summary>

rstmdb exposes Prometheus metrics on port `9090` by default, plus `/health` and `/healthz` endpoints.

```yaml
scrape_configs:
  - job_name: 'rstmdb'
    static_configs:
      - targets: ['localhost:9090']
```

Import the official Grafana dashboard: **[rstmdb Dashboard #24778](https://grafana.com/grafana/dashboards/24778)**. It includes:

- Request rate and latency (p50/p90/p99)
- Error rates by code
- Active connections and subscriptions
- WAL size, throughput, and I/O operations
- Instance and machine counts
- System resources (CPU, memory, file descriptors)

</details>

<details>
<summary><b>Architecture & data layout</b></summary>

```
┌─────────────────────────────────────────────┐
│                  Client                     │
└─────────────────┬───────────────────────────┘
                  │ RCP Protocol (TCP)
┌─────────────────▼───────────────────────────┐
│                 Server                      │
│  ┌───────────────────────────────────────┐  │
│  │           Command Handler             │  │
│  └─────────────────┬─────────────────────┘  │
│                    │                        │
│  ┌─────────────────▼─────────────────────┐  │
│  │         State Machine Engine          │  │
│  │  • Definitions  • Instances           │  │
│  │  • Guards       • Transitions         │  │
│  └─────────────────┬─────────────────────┘  │
│                    │                        │
│  ┌────────────┬────┴─────┬──────────────┐   │
│  │    WAL     │ Snapshot │  Compaction  │   │
│  │            │  Store   │   Manager    │   │
│  └────────────┴──────────┴──────────────┘   │
└─────────────────────────────────────────────┘
```

```
data/
├── wal/
│   ├── 0000000000000001.wal
│   ├── 0000000000000002.wal
│   └── ...
└── snapshots/
    ├── index.json
    ├── snap-xxx.snap
    └── ...
```

Deep dives: [`specs/architecture.md`](specs/architecture.md), [`specs/protocol.md`](specs/protocol.md).

</details>

---

## 🛣️ Roadmap

Shipped in **v0.3.0**: core engine, WAL & snapshots, replication, backup/restore, streaming, TLS, and clients for six languages.

Next up: external failover orchestration, then Raft consensus with automatic leader election. See [ROADMAP.md](ROADMAP.md).

---

## 🤝 Contributing

Issues, ideas and PRs are welcome — see [CONTRIBUTING.md](CONTRIBUTING.md).

```bash
cargo test --workspace        # run tests
RUST_LOG=debug cargo run      # run with logging
cargo fmt && cargo clippy     # format & lint
./.githooks/setup.sh          # pre-commit hooks (fmt, clippy, tests)
```

---

## 📄 License

rstmdb is licensed under the **[Business Source License 1.1](LICENSE)**. In plain words:

| ✅ Free | 💼 Needs a commercial license |
|---|---|
| Development and testing | Production on more than one node |
| Non-production environments | Offering rstmdb as a managed service (SaaS/DBaaS) |
| Evaluation and research | |
| Production on a **single node** | |

Every release converts to **Apache 2.0** four years after it's published. For commercial licensing, [contact us via rstmdb.com](https://rstmdb.com/#contact).

---

<div align="center">

**If rstmdb saves you from one more stuck `status` column, consider giving it a ⭐ — it really helps.**

</div>
