# Runbook: Parachain Collator Operations

## Overview

A parachain collator:
1. Runs an **embedded relay chain node** (syncs the relay chain)
2. Runs the **parachain node** (syncs its own chain)
3. **Produces parachain blocks** (collations) and submits them to relay chain validators

Both the relay chain and parachain components must be healthy.

## Symptoms

- Collator not producing blocks
- Parachain falling behind (not importing relay-chain-backed blocks)
- Collator sync issues (relay chain or parachain side)

## Quick Health Check

```bash
# Check both chain heights
# Relay chain height
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9933 | jq .

# Parachain height (usually on a different RPC port, e.g., 9944)
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9944 | jq .
```

**Prometheus:**
```
# Parachain best block
substrate_block_height{status="best", chain="<parachain>"}

# Relay chain best block (embedded)
substrate_block_height{status="best", chain="<relay>"}

# Peer count (both chains)
substrate_sub_libp2p_peers_count
```

**Loki:**
```logql
# Collation-related logs
{instance="<node>"} |~ "collat|Collat|parachain|aura"

# Relay chain sync on collator
{instance="<node>"} |~ "relay.*sync|relay.*import"
```

## Decision Tree

```
Collator issues
│
├─ Relay chain not synced on collator?
│  └─ The embedded relay chain node must be synced first
│     Check relay chain block height
│     Same debugging as [not-syncing](not-syncing.md)
│     Note: relay chain on collator syncs in "light" mode by default
│
├─ Parachain not syncing?
│  ├─ Check if parachain has peers
│  │  └─ Parachain needs its own peers (separate from relay chain peers)
│  │     Ensure parachain bootnodes are configured
│  │
│  └─ Check for "waiting for relay chain" in logs
│     └─ Parachain waits until relay chain is sufficiently synced
│
├─ Collator not producing blocks?
│  ├─ Is the collator registered on-chain?
│  │  └─ Not an SRE issue — check with parachain team
│  │
│  ├─ Is it the collator's turn? (AURA-based)
│  │  └─ Only the designated collator produces per slot
│  │     Check logs for "Starting collation" or "slot"
│  │
│  ├─ Collation produced but not included?
│  │  └─ Validators may not have included the collation
│  │     Check relay chain for parachain inclusion
│  │
│  └─ Collation timeout?
│     └─ Block production took too long
│        Check CPU/memory during collation window
│
└─ Collator producing but blocks not finalized?
   └─ Parachain finality depends on relay chain finality
      Check relay chain finalization status
      Go to [finalization-stall](finalization-stall.md)
```

## Resolution

### Relay chain not syncing on collator

The embedded relay chain client syncs independently:
1. Check relay chain peer count (may show as separate metric or in logs)
2. Ensure relay chain bootnodes are reachable
3. Check if `--relay-chain-rpc-urls` is used (external relay chain RPC instead of embedded node)
   - If using external RPC: verify the RPC endpoint is healthy

### Parachain peer discovery

Parachain nodes discover peers differently from relay chain:
- Via relay chain DHT (peers register their parachain-related addresses)
- Via explicit bootnodes (`--bootnodes` on the parachain CLI)

If 0 parachain peers:
1. Ensure other parachain nodes exist and are running
2. Add explicit bootnodes: `--bootnodes /dns/...`
3. Check firewall — parachain uses a separate P2P port

### Collation not being included

If the collator produces blocks but they're not included on the relay chain — this is a **chain-level issue**, not a node issue. See the parachain runbooks:
- [parachain/not-producing](../parachain/not-producing.md) — collation backing/inclusion failures
- [parachain/coretime](../parachain/coretime.md) — core assignment issues

## Key Differences from Relay Chain Nodes

| Aspect | Relay chain node | Parachain collator |
|---|---|---|
| Sync | Single chain | Two chains (relay + para) |
| Block production | BABE (6s slots) | AURA (12s typical) + relay chain scheduling |
| Finality | Own GRANDPA | Inherits from relay chain |
| Peers needed | Relay chain peers | Both relay chain + parachain peers |
| Memory | ~4-8 GB | ~4-8 GB (two chains) |
| Ports | 30333 (P2P), 9944 (RPC) | 30333 + 30334 (two P2P), 9944 + 9945 (two RPC) |

## Escalation

- Collect: both chain heights, both peer counts, collation logs, relay chain sync status
- Note: parachain ID, collator role (registered/active), coretime status
- Escalate to: `[TODO: parachain team / dev contact]`
