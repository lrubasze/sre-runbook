# Polkadot SRE Runbooks

Operational runbooks for SRE team managing Polkadot/Kusama relay chain and parachain infrastructure.

## Structure

```
runbooks/
├── README.md                        # This file — master index
├── node/                            # Node-level debugging
│   ├── overview.md                  # Node architecture primer (read first)
│   ├── not-syncing.md               # Node stuck or falling behind
│   ├── block-production.md          # Validator/collator not producing blocks
│   ├── peer-connectivity.md         # P2P networking issues
│   ├── high-resource-usage.md       # CPU, memory, disk issues
│   ├── finalization-stall.md        # Chain not finalizing
│   ├── collator-operations.md       # Parachain collator specifics
│   ├── crash-triage.md              # Node crash diagnosis (OOM/panic/segfault)
│   └── rpc-issues.md               # RPC unhealthy, slow calls, LB issues
├── parachain/                       # Parachain chain-level issues
│   ├── not-producing.md             # Blocks not included on relay chain
│   ├── xcm-delivery.md             # Cross-chain messages stuck
│   ├── coretime.md                  # Core assignment and scheduling
│   └── onboarding.md               # New parachain registration
├── release/                         # Deployments and upgrades
│   └── runtime-upgrade.md           # Pre/post runtime upgrade checklist
├── incident-response/               # Incident handling
│   └── (TODO)
└── monitoring/                      # Alerts and dashboards
    └── alert-reference.md           # Alert → runbook lookup (start here when paged)
```

## Runbook philosophy

Every runbook is structured for **incident use** — you should know what to do within 30 seconds of opening it, without scrolling past reference material you don't need yet.

### Structure (layered by urgency)

1. **Quick check** — first 30 seconds. Is it network-wide? Is the node synced? Basic status check. Just enough to decide where to go next.
2. **Triage** — numbered steps with bold headings. Walk through the most likely causes. Links to resolution sections and other runbooks where needed.
3. **Deep Investigation** — metrics checklists, Loki/Prometheus queries, detailed verification steps. Only needed when triage didn't resolve it.
4. **Resolution** — reference material for specific fixes (keystore repair, resource tuning, PVF bottleneck, etc.). Linked from triage, not inlined.
5. **Escalation** — what to collect before escalating, who to contact.

### Principles

- **Metrics over logs** — Prometheus metrics are always available regardless of log level configuration. Prefer them for Quick check and Triage. Loki queries go in Deep Investigation.
- **Verify both sides** — for parachain issues, always check the collator side (block built?) AND the relay chain side (candidate backed/included?). One side reporting success doesn't mean the other agrees.
- **Link, don't duplicate** — triage steps link to resolution sections and other runbooks rather than inlining explanations. One source of truth per topic.
- **No invented log messages** — every log string in these runbooks was verified against the polkadot-sdk source code. If a log message can't be confirmed, it doesn't go in.
- **Placeholders for thresholds** — values that need team agreement use `[TODO: ...]` markers rather than guessed numbers.

## How to use

1. **Got paged?** Start with [monitoring/alert-reference](monitoring/alert-reference.md) — maps alerts to runbooks
2. Find the relevant runbook in the table below
3. Follow **Quick check** (30 seconds)
4. Walk through **Triage** to narrow down the cause
5. If needed, go to **Deep Investigation** for detailed verification
6. Apply the linked **Resolution**
7. If unresolved, see **Escalation**

## Quick reference: Which runbook?

### Node issues

| Symptom | Runbook |
|---|---|
| Node stuck at block N, not importing new blocks | [node/not-syncing](node/not-syncing.md) |
| Validator missing slots / not authoring | [node/block-production](node/block-production.md) |
| 0 peers or dropping peers | [node/peer-connectivity](node/peer-connectivity.md) |
| OOM kills, high CPU, disk full | [node/high-resource-usage](node/high-resource-usage.md) |
| Best block advancing but finalized block stuck | [node/finalization-stall](node/finalization-stall.md) |
| Parachain collator not producing / not syncing | [node/collator-operations](node/collator-operations.md) |
| Node crashed — OOM, panic, segfault | [node/crash-triage](node/crash-triage.md) |
| RPC unhealthy, slow calls, WebSocket drops | [node/rpc-issues](node/rpc-issues.md) |

### Parachain issues

| Symptom | Runbook |
|---|---|
| Parachain blocks not being included on relay chain | [parachain/not-producing](parachain/not-producing.md) |
| Cross-chain transfer / XCM message stuck | [parachain/xcm-delivery](parachain/xcm-delivery.md) |
| No coretime, expired allocation, on-demand not working | [parachain/coretime](parachain/coretime.md) |
| New parachain stuck in onboarding | [parachain/onboarding](parachain/onboarding.md) |

### Release / deployment

| Symptom | Runbook |
|---|---|
| Runtime upgrade scheduled or just enacted | [release/runtime-upgrade](release/runtime-upgrade.md) |

## Observability

- **Grafana dashboards:** `[TODO: link to your Grafana instance]`
- **Prometheus metrics:** Substrate exposes metrics on `:9615/metrics` by default
- **Logs:** Available via Loki in Grafana; node logs use structured format with `target` field for filtering

### Key Prometheus metrics

| Metric | What it tells you |
|---|---|
| `substrate_block_height{status="best"}` | Current best (head) block |
| `substrate_block_height{status="finalized"}` | Current finalized block |
| `substrate_sub_libp2p_peers_count` | Number of connected peers |
| `substrate_tasks_polling_duration_bucket` | Task execution times (spot bottlenecks) |
| `substrate_state_db_cache_bytes` | State DB cache memory usage |
| `process_resident_memory_bytes` | Total RSS memory |

### Key Loki log filters

```logql
# Errors for a specific node
{instance="<node>"} |= "ERROR"

# Sync-related logs
{instance="<node>"} |~ "sync|import|block"

# Consensus/finality logs
{instance="<node>"} |~ "grandpa|finali"

# Networking
{instance="<node>"} |~ "libp2p|peer|network"
```
