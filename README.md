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
│   └── collator-operations.md       # Parachain collator specifics
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
    └── (TODO)
```

## How to use

1. Identify the **symptom** you're seeing
2. Find the relevant runbook in the table below
3. Follow the **Quick Health Check** to confirm the issue
4. Walk through the **Decision Tree** to narrow down the cause
5. Follow the **Resolution** steps
6. If unresolved, see **Escalation**

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
