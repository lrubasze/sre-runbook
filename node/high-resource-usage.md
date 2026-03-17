# Runbook: High Resource Usage

## Symptoms

- Node killed by OOM killer
- High CPU usage (>90% sustained)
- Disk full or filling fast
- Node becoming unresponsive

## Quick Health Check

**Prometheus / Grafana:**
```
# Memory
process_resident_memory_bytes{instance="<node>"}

# CPU
rate(process_cpu_seconds_total{instance="<node>"}[5m])

# Open file descriptors
process_open_fds{instance="<node>"}
```

**Loki:**
```logql
# Check for OOM or resource warnings
{instance="<node>"} |~ "OOM|memory|out of memory"
```

**On the host:**
```bash
# Check if OOM killer struck
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom

# Disk usage
df -h /path/to/chain/data

# Memory
free -h

# Process memory details
cat /proc/<pid>/status | grep -i vm
```

## Decision Tree

```
High resource usage
│
├─ OOM killed?
│  ├─ During initial sync (warp/full)?
│  │  └─ Sync is memory-intensive (state download, trie building)
│  │     → Resolution: Memory during sync
│  │
│  ├─ During normal operation?
│  │  ├─ Check state DB cache size (--state-cache-size)
│  │  │  └─ Default may be too large for available RAM
│  │  │
│  │  ├─ Check if finalization is stalled
│  │  │  └─ Stalled finality = journals/state accumulates in memory
│  │  │     Go to [finalization-stall](finalization-stall.md)
│  │  │
│  │  └─ Memory leak? (RSS growing unbounded over days)
│  │     → Resolution: Escalate with memory profile
│  │
│  └─ After runtime upgrade?
│     └─ New runtime may have higher memory requirements
│        → Resolution: Increase memory allocation
│
├─ High CPU?
│  ├─ During sync?
│  │  └─ Expected — block execution is CPU-intensive
│  │
│  ├─ During normal operation?
│  │  └─ Check if node is re-executing old blocks (state rebuild)
│  │     Or: RocksDB compaction spike
│  │
│  └─ 100% sustained on all cores?
│     └─ Possible infinite loop or consensus issue → escalate
│
└─ Disk full?
   ├─ Pruning mode?
   │  └─ Archive nodes grow ~100+ GB/year
   │     Pruned nodes are much smaller
   │
   ├─ RocksDB compaction backlog?
   │  └─ Disk needs headroom for compaction (20%+ free recommended)
   │
   └─ Log files filling disk?
      └─ Check log rotation config
```

## Resolution

### Memory during sync

Warp sync and state rebuilding are the most memory-intensive operations. Options:
1. **Increase node memory** to 16+ GB during sync
2. **Limit state cache:** `--state-cache-size 1073741824` (1 GB, default may be higher)
3. If using `--state-pruning archive`: requires significantly more memory
4. Wait — memory usage typically drops after sync completes

### Memory during normal operation

Expected baseline memory (rough guide):
- **Pruned relay chain node:** ~4-8 GB
- **Archive relay chain node:** ~8-16 GB
- **Parachain collator:** ~4-8 GB (runs two chains)

If memory exceeds these significantly:
1. Check `--state-cache-size` setting
2. Check finalization status — if finalized block is far behind best block, state journals accumulate in RAM
3. Check for known memory issues in release notes of the running version
4. Restart the node (resets caches, reclaims leaked memory)

### Disk full

```bash
# Check what's using space
du -sh /path/to/chain/data/*

# Typical breakdown:
# db/         - RocksDB (biggest)
# keystore/   - small
# network/    - small
```

Options:
1. **Switch to pruned mode** if currently archive (requires re-sync)
2. **Increase disk size**
3. **Check log rotation** — ensure logs aren't accumulating
4. **RocksDB compaction:** if temporarily full during compaction, add 20%+ disk headroom

### RocksDB compaction spikes

RocksDB periodically compacts data, causing temporary CPU + I/O spikes.
- This is normal behavior
- If it causes issues: consider NVMe storage for better I/O
- `--database paritydb` is an alternative backend (less compaction overhead, but less battle-tested)

## Escalation

- Collect: `dmesg` output (OOM), memory graph over time, disk usage breakdown
- Note: exact binary version, CLI flags, pruning mode, chain
- If suspected memory leak: RSS graph over 24-48h showing steady growth
- Escalate to: `[TODO: dev team contact / channel]`
