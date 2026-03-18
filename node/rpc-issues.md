# Runbook: RPC Issues

> **Triggered by alerts:** `UnhealthyRPCEndpoint`, `SlowRPCCallTime`, `UnhealthyRPCCloudflareLBPool`

## Symptoms

- RPC endpoint returning errors or timeouts
- RPC calls taking >5 seconds
- Health check probes failing
- WebSocket connections dropping
- Cloudflare LB marking pool members unhealthy

## Quick Health Check

```bash
# Basic HTTP health check
curl -s -o /dev/null -w "%{http_code}" http://<node>:9944/health

# Test a simple RPC call with timing
time curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9944

# Test a WebSocket connection
websocat ws://<node>:9944 --ping-interval 5 -1

# Check if the port is open and the process is listening
ss -tlnp | grep 9944

# Check node process
systemctl status <service>
```

**Prometheus:**
```
# Node process running and responsive?
up{instance="<node>"}

# CPU — is the node overloaded?
rate(process_cpu_seconds_total{instance="<node>"}[5m])

# Memory — is it swapping?
process_resident_memory_bytes{instance="<node>"}

# Open file descriptors — approaching limit?
process_open_fds{instance="<node>"}
```

**On the host:**
```bash
# Is the system swapping?
free -h
vmstat 1 5   # check si/so columns

# Connection count to RPC port
ss -s
ss -tn | grep :9944 | wc -l

# CPU load
uptime
```

## Decision Tree

```
RPC endpoint unhealthy or slow
│
├─ Node process not running?
│  └─ Check crash cause → see [crash-triage](crash-triage.md)
│
├─ Node running but port not responding?
│  ├─ RPC disabled? Check CLI flags for --rpc-port, --unsafe-rpc-external
│  │  └─ Ensure RPC is enabled and bound to correct interface
│  │
│  ├─ Too many connections?
│  │  └─ Check: ss -tn | grep :9944 | wc -l
│  │     Default max connections may be hit
│  │     → Resolution: Connection limits
│  │
│  └─ Process is hung/unresponsive?
│     └─ Check CPU — may be stuck in long operation
│        Check if node is syncing (block import is prioritized over RPC)
│        Last resort: restart the node
│
├─ RPC responding but slow (>5s)?
│  ├─ All calls slow?
│  │  ├─ CPU overloaded?
│  │  │  └─ Node doing heavy work (sync, compaction, block execution)
│  │  │     See [high-resource-usage](high-resource-usage.md)
│  │  │
│  │  └─ Swapping?
│  │     └─ free -h shows high swap usage
│  │        Cause: not enough RAM
│  │        → Increase memory or reduce --state-cache-size
│  │
│  ├─ Only state queries slow?
│  │  └─ Large state reads on archive nodes can be inherently slow
│  │     state_getStorage, state_queryStorageAt, etc.
│  │     Not necessarily a problem — check if query is reasonable
│  │
│  └─ Only subscriptions timing out?
│     └─ WebSocket backpressure — too many subscribers
│        Check subscription count, consider adding RPC nodes
│
├─ Intermittent failures?
│  ├─ Correlated with block production times?
│  │  └─ Block execution competing for CPU with RPC
│  │     Consider dedicated RPC nodes (non-validator)
│  │
│  └─ Correlated with RocksDB compaction?
│     └─ Temporary I/O spikes
│        See [high-resource-usage](high-resource-usage.md)
│
└─ Cloudflare LB specific?
   ├─ All backends failing health checks?
   │  └─ Check each backend directly — may be node-side issue
   │
   ├─ Health check config mismatch?
   │  └─ Ensure CF health check path/port matches node config
   │     Common: health check uses wrong port or path
   │
   └─ Backends healthy but LB unhealthy?
      └─ Cloudflare-side issue — check Cloudflare status page
```

## Resolution

### Connection limits

```bash
# Check current connections
ss -tn | grep :9944 | wc -l

# Check file descriptor usage
ls /proc/<node-pid>/fd | wc -l
ulimit -n  # should be >= 10000
```

If connection count is very high:
1. Check for connection leaks on the client side (clients not closing connections)
2. Increase `--rpc-max-connections` if set too low
3. Add more RPC nodes behind the load balancer
4. Consider rate limiting at the LB level

### Node too busy for RPC

If the node is a validator and RPC is slow due to block production load:
1. **Best practice:** Run dedicated RPC nodes (non-validator) for serving RPC traffic
2. Validators should not serve public RPC — it competes with consensus duties
3. If validator must serve RPC: limit exposure to internal/monitoring use only

### Swap/memory causing slowness

```bash
# Check swap usage
free -h
swapon --show

# If heavily swapping:
# 1. Reduce state cache
# polkadot --state-cache-size 536870912  (512 MB)

# 2. Or increase RAM
```

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
3. If backends are healthy but CF marks them down: check health check configuration (path, port, expected response code, timeout)
4. If a subset of backends are down: fix individual nodes per other runbooks

## Escalation

- Collect: RPC response times, connection count, CPU/memory at the time, which RPC methods are slow
- Note: node role (validator/full/archive), whether RPC is public or internal
- For persistent slowness with adequate resources: may indicate node software issue → escalate to dev team
- Escalate to: `[TODO: infra team / dev team contact]`
