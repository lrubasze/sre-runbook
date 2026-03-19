# Runbook: Node Not Syncing

> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. Does the node have peers?**
- `substrate_sub_libp2p_peers_count` > 0?
- 0 peers → see [peer-connectivity](peer-connectivity.md)

**2. Is the node importing blocks at all?**
```bash
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9944 | jq .
# Expected: {"isSyncing": true/false, "peers": N, "shouldHavePeers": true}
```

**3. How far behind is it?**
- Compare `substrate_block_height{status="best"}` against a known healthy node or public RPC

## Triage

1. **Has peers but not syncing?**
   - Check logs for `"State sync"` or `"Warp sync"` → warp sync in progress, can take hours. See [Warp sync taking too long](#warp-sync-taking-too-long)
   - Check logs for `"Verification failed"` or `"bad block"` → corrupt DB or wrong fork. See [DB corruption](#db-corruption)
   - Check logs for `"Queue is full"` → node overwhelmed importing blocks. See [high-resource-usage](high-resource-usage.md)

2. **Syncing but falling behind?**
   - Check CPU — at 100%? Block execution is CPU-bound → see [high-resource-usage](high-resource-usage.md)
   - Check disk I/O — saturated? RocksDB is I/O-bound → upgrade to SSD/NVMe
   - Check network bandwidth — insufficient for block download?

3. **Stuck after restart?**
   - Check logs for `"Database"` errors → [DB corruption](#db-corruption)
   - Check if node is replaying blocks (rebuilding state) → normal after unclean shutdown, wait for it to catch up

## Deep Investigation

### Useful Loki queries

```logql
# Check if node is importing blocks at all
{instance="<node>"} |~ "Imported|imported"

# Warp sync progress
{instance="<node>"} |~ "warp|state download"

# Block replay after unclean shutdown
{instance="<node>"} |~ "Preparing|replay"
```

### Useful Prometheus queries

```promql
substrate_block_height{status="best", instance="<node>"}
substrate_block_height{status="finalized", instance="<node>"}
```

## Resolution

### Warp sync taking too long

- Warp sync downloads finality proofs first, then state. State download can take hours depending on chain size.
- **Check progress:** look for `State sync` log messages showing download percentage
- **Not a problem** unless >24h with no progress. If stuck:
  - Verify peers are reachable (check peer count)
  - Consider restarting the node
  - Check if disk space is sufficient

### DB corruption

Signs: repeated errors mentioning "database", "corruption", "invalid block", or node crashes on specific block.

1. **Stop the node**
2. **Check disk health** — corruption often caused by disk issues or unclean shutdowns
3. **Try restarting** — RocksDB may self-recover on restart
4. If restart doesn't help:
   - **Purge chain data:** `polkadot purge-chain --chain <chain>` and re-sync
   - Or **restore from snapshot** if available (faster than full re-sync)
5. **Investigate root cause** — check for OOM kills, disk failures, or power issues that caused unclean shutdown

### Resource bottleneck

See [high-resource-usage](high-resource-usage.md) for detailed steps.

Quick checks:
- CPU: `top` or Grafana `process_cpu_seconds_total`
- Memory: `free -h` or Grafana `process_resident_memory_bytes`
- Disk I/O: `iostat -x 1` or Grafana disk metrics
- Disk space: `df -h` (RocksDB needs headroom for compaction)

## Escalation

- Collect: logs from the stuck period, exact block number where stuck, peer count
- Check: are other nodes on the same network also affected? (network-wide vs single node)
- Escalate to: `[TODO: dev team contact / channel]`
