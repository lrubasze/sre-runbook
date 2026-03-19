# Runbook: RPC Issues

> **Triggered by alerts:** `UnhealthyRPCEndpoint`, `SlowRPCCallTime`, `UnhealthyRPCCloudflareLBPool`

## Quick check

**1. Is the node running?**
```bash
systemctl status <service>
ss -tlnp | grep 9944
```
- Not running → see [crash-triage](crash-triage.md)

**2. Does it respond?**
```bash
# Basic health
curl -s -o /dev/null -w "%{http_code}" http://<node>:9944/health

# Timed RPC call
time curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9944
```

**3. Is the host overloaded?**
```bash
uptime       # load average
free -h      # swap usage
df -h        # disk space
```

## Triage

1. **Port not responding?**
   - RPC disabled? Check CLI flags for `--rpc-port`, `--unsafe-rpc-external` → ensure RPC is enabled and bound to correct interface
   - Too many connections? `ss -tn | grep :9944 | wc -l` → see [Connection limits](#connection-limits)
   - Process hung? Check CPU, check if node is syncing (block import prioritized over RPC). Last resort: restart

2. **RPC responding but slow (>5s)?**
   - All calls slow?
     - CPU overloaded → node doing heavy work (sync, compaction). See [high-resource-usage](high-resource-usage.md)
     - Swapping? `free -h` shows high swap → not enough RAM. Reduce `--state-cache-size` or increase memory. See [Swap/memory](#swapmemory-causing-slowness)
   - Only state queries slow? → large state reads on archive nodes can be inherently slow (`state_getStorage`, `state_queryStorageAt`). Not necessarily a problem.
   - Only subscriptions timing out? → WebSocket backpressure, too many subscribers. Consider adding RPC nodes.

3. **Intermittent failures?**
   - Correlated with block production times? → block execution competing for CPU. Consider dedicated RPC nodes (non-validator). See [Node too busy](#node-too-busy-for-rpc)
   - Correlated with RocksDB compaction? → temporary I/O spikes. See [high-resource-usage](high-resource-usage.md)

4. **Cloudflare LB specific?**
   - All backends failing health checks? → check each backend directly
   - Health check config mismatch? → ensure CF health check path/port matches node config
   - Backends healthy but LB unhealthy? → Cloudflare-side issue, check status page
   - See [Cloudflare LB resolution](#cloudflare-lb-pool-unhealthy)

## Deep Investigation

### Useful Prometheus queries

```promql
# Node up?
up{instance="<node>"}

# CPU
rate(process_cpu_seconds_total{instance="<node>"}[5m])

# Memory
process_resident_memory_bytes{instance="<node>"}

# File descriptors
process_open_fds{instance="<node>"}
```

### Host-level checks

```bash
# Swap activity
vmstat 1 5   # check si/so columns

# Connection count to RPC port
ss -tn | grep :9944 | wc -l

# File descriptor usage
ls /proc/<node-pid>/fd | wc -l
ulimit -n
```

## Resolution

### Connection limits

```bash
ss -tn | grep :9944 | wc -l
```

If connection count is very high:
1. Check for connection leaks on the client side (clients not closing connections)
2. Increase `--rpc-max-connections` if set too low
3. Add more RPC nodes behind the load balancer
4. Consider rate limiting at the LB level

### Node too busy for RPC

If the node is a validator and RPC is slow due to block production load:
1. **Best practice:** run dedicated RPC nodes (non-validator) for serving RPC traffic
2. Validators should not serve public RPC — it competes with consensus duties
3. If validator must serve RPC: limit exposure to internal/monitoring use only

### Swap/memory causing slowness

```bash
free -h
swapon --show
```

If heavily swapping:
1. Reduce state cache: `--state-cache-size 536870912` (512 MB)
2. Or increase RAM

### Cloudflare LB pool unhealthy

1. Identify which pool members are failing health checks
2. Test each backend directly:
   ```bash
   for node in node1 node2 node3; do
     echo -n "$node: "
     curl -s -o /dev/null -w "%{http_code} %{time_total}s" http://$node:9944/health
     echo
   done
   ```
3. Backends healthy but CF marks them down? → check health check config (path, port, expected response code, timeout)
4. Subset of backends down? → fix individual nodes per other runbooks

## Escalation

- Collect: RPC response times, connection count, CPU/memory, which RPC methods are slow
- Note: node role (validator/full/archive), whether RPC is public or internal
- For persistent slowness with adequate resources: may indicate software issue
- Escalate to: `[TODO: infra team / dev team contact]`
