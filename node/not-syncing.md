# Runbook: Node Not Syncing

## Symptoms

- `substrate_block_height{status="best"}` is not increasing
- Logs show no new blocks being imported
- Node is stuck at a specific block number
- Node is syncing but falling further behind the network

## Quick Health Check

```logql
# Check current block height vs network
# In Grafana, compare:
substrate_block_height{status="best", instance="<node>"}
# against a known healthy node or public RPC

# Check if node is importing blocks at all
{instance="<node>"} |~ "Imported|imported"
```

```bash
# Via RPC (if accessible)
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9933 | jq .

# Expected: {"isSyncing": true/false, "peers": N, "shouldHavePeers": true}
```

## Decision Tree

```
Node not importing new blocks
│
├─ Has 0 peers?
│  └─ Go to [peer-connectivity](peer-connectivity.md)
│
├─ Has peers but not syncing?
│  ├─ Check logs for "State sync" or "Warp sync"
│  │  └─ Warp sync in progress — can be slow, may take hours
│  │     Check: {instance="<node>"} |~ "warp|state download"
│  │
│  ├─ Check logs for "Verification failed" or "bad block"
│  │  └─ Node may be on wrong fork or have corrupt DB
│  │     → Resolution: DB corruption
│  │
│  └─ Check logs for "Queue is full" or "too many"
│     └─ Node is overwhelmed importing blocks
│        → Resolution: Resource bottleneck (see 04-high-resource-usage)
│
├─ Syncing but falling behind?
│  ├─ Check CPU usage — is it at 100%?
│  │  └─ Block execution is CPU-bound
│  │     → Resolution: Resource bottleneck
│  │
│  ├─ Check disk I/O — is it saturated?
│  │  └─ RocksDB is I/O-bound
│  │     → Resolution: Upgrade to SSD/NVMe
│  │
│  └─ Check network bandwidth
│     └─ Insufficient bandwidth for block download
│        → Resolution: Network bottleneck
│
└─ Stuck after restart?
   ├─ Check logs for "Database" errors
   │  └─ → Resolution: DB corruption
   │
   └─ Check if node is replaying blocks (rebuilding state)
      └─ Normal after unclean shutdown. Wait for it to catch up.
         Check: {instance="<node>"} |~ "Preparing|replay"
```

## Resolution

### Warp sync taking too long

- Warp sync downloads finality proofs first, then state. The state download phase can take hours depending on chain size.
- **Check progress:** Look for `State sync` log messages showing download percentage.
- **Not a problem** unless it's been >24h with no progress. If stuck:
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

If none of the above resolves the issue:
- Collect logs from the stuck period: `{instance="<node>"} |= "ERROR" OR |= "WARN"`
- Note the exact block number where it's stuck
- Check if other nodes on the same network are also affected (network-wide issue vs single node)
- Escalate to: `[TODO: dev team contact / channel]`
