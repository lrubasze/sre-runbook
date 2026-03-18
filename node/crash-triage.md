# Runbook: Node Crash Triage

> **Triggered by alert:** `VMNodeProcessCrashed`

## Symptoms

- Node process exited unexpectedly
- Systemd restarted the service (or it's in a crash loop)
- `dmesg` or `journalctl` shows OOM kill, segfault, or panic

## Quick Health Check

```bash
# Is the node running now?
systemctl status <service>

# How many recent restarts?
journalctl -u <service> --since "1 hour ago" | grep -c "Started\|Stopped"

# Check for OOM kill
dmesg | grep -i "oom\|killed process"
journalctl -k --since "1 hour ago" | grep -i oom

# Check for panic/segfault
journalctl -u <service> --since "1 hour ago" | grep -i "panic\|segfault\|SIGSEGV\|SIGABRT\|fatal"

# Check last exit status
systemctl show <service> --property=ExecMainStatus
```

## Decision Tree

```
Node crashed
│
├─ OOM killed? (dmesg shows "Out of memory: Killed process")
│  ├─ During initial sync?
│  │  └─ Sync is memory-intensive
│  │     Increase RAM or limit cache: --state-cache-size 1073741824
│  │     See [high-resource-usage](high-resource-usage.md)
│  │
│  ├─ During normal operation?
│  │  ├─ Finalization stalled? (journals accumulating)
│  │  │  └─ Memory is a symptom, fix finalization first
│  │  │     See [finalization-stall](finalization-stall.md)
│  │  │
│  │  ├─ After runtime upgrade?
│  │  │  └─ New runtime may need more memory → increase allocation
│  │  │
│  │  └─ Memory growing steadily over days? (leak)
│  │     └─ Collect RSS graph over 24-48h → escalate to dev team
│  │
│  └─ Crash loop (OOM → restart → OOM)?
│     └─ Reduce --state-cache-size, increase RAM, or restore from snapshot
│        See [high-resource-usage](high-resource-usage.md)
│
├─ Panic in Rust code? (logs show "thread panicked" or backtrace)
│  └─ This is a bug in the node software
│     1. Capture the full backtrace from logs
│     2. Note exact binary version: polkadot --version
│     3. Note the block height when it panicked
│     4. Check if known issue in release notes
│     5. Escalate with backtrace → dev team
│
├─ Segmentation fault? (SIGSEGV)
│  └─ Possible causes:
│     - Corrupt binary (re-download/rebuild)
│     - Hardware issue (check ECC memory errors, disk health)
│     - WASM execution issue (rare)
│     1. Verify binary checksum against release
│     2. Check hardware: `smartctl -a /dev/sda`, `mcelog`
│     3. If recurring: escalate to dev team
│
├─ DB corruption on restart?
│  └─ Crash during write can corrupt RocksDB
│     Logs: "database", "corruption", "invalid"
│     See [not-syncing](not-syncing.md) — DB corruption section
│
└─ No clear cause in logs?
   ├─ Check if process was killed externally
   │  └─ `journalctl -u <service> | grep -i "signal\|kill"`
   │     systemd OOMPolicy, cgroup limits, manual kill
   │
   └─ Check host-level issues
      └─ Disk full? Power event? Kernel issue?
         `dmesg | tail -50`
         `df -h`
```

## Resolution

### After OOM kill

1. **Immediate:** If the node restarted successfully, check it's syncing:
   ```bash
   curl -s -H "Content-Type: application/json" \
     -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
     http://<node>:9933 | jq .
   ```
2. **Prevent recurrence:**
   - Reduce `--state-cache-size` (e.g., to 1 GB or 512 MB)
   - Increase machine memory
   - Check and fix finalization stall if present
3. **If crash-looping:** Restore from snapshot to avoid repeated OOM during state rebuild

### After panic

1. The node usually restarts fine after a panic (transient state issue)
2. If it panics on the same block repeatedly:
   - Try skipping that block by syncing from a snapshot past that height
   - Escalate with the full backtrace
3. Capture: backtrace, binary version, block height, chain

### After segfault

1. Re-download or rebuild the binary
2. Verify checksum: `sha256sum polkadot`
3. If it recurs with a fresh binary: suspect hardware — check ECC errors, run memtest
4. Escalate with `coredump` if available (`coredumpctl list`)

### After DB corruption

See [not-syncing.md — DB corruption](not-syncing.md#db-corruption).

Quick path:
1. Try restarting — RocksDB sometimes self-recovers
2. If not: `polkadot purge-chain --chain <chain>` + re-sync or restore from snapshot

## Escalation

- Collect: crash type (OOM/panic/segfault), backtrace if available, binary version, block height, host metrics at crash time
- For panics: full backtrace is critical for dev team
- For recurring crashes: include frequency and pattern (same block? same time of day? correlated with load?)
- Escalate to: `[TODO: dev team contact / channel]`
