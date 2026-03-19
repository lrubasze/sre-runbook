# Runbook: Node Crash Triage

> **Triggered by alert:** `VMNodeProcessCrashed`

## Quick check

**1. Is the node running now?**
```bash
systemctl status <service>
```

**2. What killed it?**
```bash
# OOM?
dmesg | grep -i "oom\|killed process"

# Panic or segfault?
journalctl -u <service> --since "1 hour ago" | grep -i "panic\|segfault\|SIGSEGV\|SIGABRT\|fatal"

# Exit status
systemctl show <service> --property=ExecMainStatus
```

**3. Is it crash-looping?**
```bash
journalctl -u <service> --since "1 hour ago" | grep -c "Started\|Stopped"
```

## Triage

1. **OOM killed?** (`dmesg` shows "Out of memory: Killed process")
   - During initial sync? → sync is memory-intensive. Increase RAM or limit cache: `--state-cache-size 1073741824`. See [high-resource-usage](high-resource-usage.md)
   - Finalization stalled? → journals accumulate in memory. Fix finalization first. See [finalization-stall](finalization-stall.md)
   - After runtime upgrade? → new runtime may need more memory, increase allocation
   - Memory growing over days? (leak) → collect RSS graph over 24-48h, [Escalation](#escalation)
   - Crash loop? → reduce `--state-cache-size`, increase RAM, or restore from snapshot
   - See [After OOM kill](#after-oom-kill)

2. **Panic in Rust code?** (logs show "thread panicked" or backtrace)
   - This is a bug in the node software
   - Capture the full backtrace, binary version (`polkadot --version`), block height
   - Check if known issue in release notes
   - See [After panic](#after-panic)

3. **Segmentation fault?** (SIGSEGV)
   - Possible: corrupt binary, hardware issue, WASM execution issue (rare)
   - See [After segfault](#after-segfault)

4. **DB corruption on restart?**
   - Crash during write can corrupt RocksDB
   - Logs: `"database"`, `"corruption"`, `"invalid"`
   - See [not-syncing — DB corruption](not-syncing.md#db-corruption)

5. **No clear cause in logs?**
   - Check if process was killed externally: `journalctl -u <service> | grep -i "signal\|kill"` (systemd OOMPolicy, cgroup limits, manual kill)
   - Check host-level issues: disk full? power event? kernel issue? `dmesg | tail -50`, `df -h`

## Resolution

### After OOM kill

1. **Immediate:** if the node restarted successfully, check it's syncing:
   ```bash
   curl -s -H "Content-Type: application/json" \
     -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
     http://<node>:9944 | jq .
   ```
2. **Prevent recurrence:**
   - Reduce `--state-cache-size` (e.g., to 1 GB or 512 MB)
   - Increase machine memory
   - Check and fix finalization stall if present
3. **If crash-looping:** restore from snapshot to avoid repeated OOM during state rebuild

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
