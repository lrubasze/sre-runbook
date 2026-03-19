# Runbook: High Resource Usage

> **Triggered by alerts:** `VMNodeProcessCrashed` (when OOM), `SlowRPCCallTime` (when resource-bound)
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. What resource is the problem?**
```bash
# OOM killed?
dmesg | grep -i "oom\|killed process"
journalctl -k | grep -i oom

# Disk full?
df -h /path/to/chain/data

# Memory pressure?
free -h

# CPU?
uptime
```

**2. When did it start?**
- During initial sync? → expected, sync is resource-intensive
- During normal operation? → continue to triage
- After runtime upgrade? → new runtime may have higher requirements

## Triage

### OOM killed

1. **During initial sync?**
   - Sync is memory-intensive (state download, trie building)
   - See [Memory during sync](#memory-during-sync)

2. **During normal operation?**
   - Check `--state-cache-size` — default may be too large for available RAM
   - Check if finalization is stalled → journals accumulate in memory. See [finalization-stall](finalization-stall.md)
   - RSS growing unbounded over days? → possible memory leak, escalate with memory profile

3. **After runtime upgrade?**
   - New runtime may need more memory → increase allocation

4. **Crash loop (OOM → restart → OOM)?**
   - Reduce `--state-cache-size`, increase RAM, or restore from snapshot
   - See [Memory during normal operation](#memory-during-normal-operation)

### High CPU

1. **During sync?** → expected, block execution is CPU-intensive
2. **During normal operation?** → check if re-executing old blocks (state rebuild) or RocksDB compaction spike
3. **100% sustained on all cores?** → possible infinite loop or consensus issue → [Escalation](#escalation)

### Disk full

1. **Pruning mode?** → archive nodes grow ~100+ GB/year, pruned are much smaller
2. **RocksDB compaction backlog?** → needs 20%+ free headroom for compaction
3. **Log files filling disk?** → check log rotation config
4. See [Disk full resolution](#disk-full)

## Deep Investigation

### Useful Prometheus queries

```promql
# Memory
process_resident_memory_bytes{instance="<node>"}

# CPU
rate(process_cpu_seconds_total{instance="<node>"}[5m])

# Open file descriptors
process_open_fds{instance="<node>"}
```

### Host-level checks

```bash
# Process memory details
cat /proc/<pid>/status | grep -i vm

# Disk breakdown
du -sh /path/to/chain/data/*
# Typical: db/ (biggest), keystore/ (small), network/ (small)
```

## Resolution

### Memory during sync

Warp sync and state rebuilding are the most memory-intensive operations:
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
2. Check finalization status — if finalized block is far behind best, state journals accumulate in RAM
3. Check for known memory issues in release notes
4. Restart the node (resets caches, reclaims leaked memory)

### Disk full

```bash
du -sh /path/to/chain/data/*
```

Options:
1. **Switch to pruned mode** if currently archive (requires re-sync)
2. **Increase disk size**
3. **Check log rotation** — ensure logs aren't accumulating
4. **RocksDB compaction:** if temporarily full during compaction, add 20%+ disk headroom

### RocksDB compaction spikes

RocksDB periodically compacts data, causing temporary CPU + I/O spikes.
- This is normal behavior
- If it causes issues: check storage I/O capacity
- `--database paritydb` is an alternative backend (less compaction overhead, but less battle-tested)

## Escalation

- Collect: `dmesg` output (OOM), memory graph over time, disk usage breakdown
- Note: exact binary version, CLI flags, pruning mode, chain
- If suspected memory leak: RSS graph over 24-48h showing steady growth
- Escalate to: `[TODO: dev team contact / channel]`
