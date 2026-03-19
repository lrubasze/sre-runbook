# Runbook: Parachain Collator Operations

> **Triggered by alert:** `CollatorNotProducingBlocks`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

A collator runs two chains: an embedded relay chain node + the parachain node. Both must be healthy.

**1. Are both chains synced?**
```bash
# Parachain health (default port 9944)
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9944 | jq .

# Relay chain health (default port 9945)
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9945 | jq .
```

**2. Does the collator have peers on both chains?**
- `substrate_sub_libp2p_peers_count` — check for both chain labels

**3. Is the collator producing?**
- `substrate_block_height{status="best", chain="<parachain>"}` — advancing?

## Triage

1. **Relay chain not synced?**
   - Embedded relay chain must sync first
   - Same debugging as [not-syncing](not-syncing.md)
   - Note: relay chain on collator syncs in "light" mode by default

2. **Parachain not syncing?**
   - Check if parachain has peers (separate from relay chain peers)
   - Parachain peers discovered via relay chain DHT or explicit `--bootnodes`
   - Logs: `"waiting for relay chain"` → parachain waits for relay chain to sync
   - See [Parachain peer discovery](#parachain-peer-discovery)

3. **Collator not producing blocks?**
   - Is the collator registered on-chain? → not SRE, check with parachain team
   - Is it the collator's turn? (AURA) → only the designated collator produces per slot
   - Check for block production issues → see [block-production](block-production.md) (collator triage section)

4. **Collation produced but not included?**
   - This is a chain-level issue, not a node issue
   - See [parachain/not-producing](../parachain/not-producing.md)
   - See [parachain/coretime](../parachain/coretime.md) for core assignment

5. **Blocks not finalized?**
   - Parachain finality depends on relay chain finality
   - See [finalization-stall](finalization-stall.md)

## Deep Investigation

### Useful Loki queries

```logql
# Collation-related logs
{instance="<node>"} |~ "collat|Collat|parachain|aura"

# Relay chain sync on collator
{instance="<node>"} |~ "relay.*sync|relay.*import"
```

### Useful Prometheus queries

```promql
# Parachain best block
substrate_block_height{status="best", chain="<parachain>"}

# Relay chain best block (embedded)
substrate_block_height{status="best", chain="<relay>"}

# Peer count (both chains)
substrate_sub_libp2p_peers_count
```

### Key differences from relay chain nodes

| Aspect | Relay chain node | Parachain collator |
|---|---|---|
| Sync | Single chain | Two chains (relay + para) |
| Block production | BABE (6s slots) | AURA (12s typical) + relay chain scheduling |
| Finality | Own GRANDPA | Inherits from relay chain |
| Peers needed | Relay chain peers | Both relay chain + parachain peers |
| Memory | ~4-8 GB | ~4-8 GB (two chains) |
| Ports | 30333 (P2P), 9944 (RPC) | 30333 + 30334 (two P2P), 9944 + 9945 (two RPC) |

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

If the collator produces blocks but they're not included on the relay chain — this is a **chain-level issue**, not a node issue. See:
- [parachain/not-producing](../parachain/not-producing.md) — collation backing/inclusion failures
- [parachain/coretime](../parachain/coretime.md) — core assignment issues

## Escalation

- Collect: both chain heights, both peer counts, collation logs, relay chain sync status
- Note: parachain ID, collator role (registered/active), coretime status
- Escalate to: `[TODO: parachain team / dev contact]`
