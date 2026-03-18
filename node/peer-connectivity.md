# Runbook: Peer Connectivity Issues

> **Triggered by alerts:** `NumberOfPeersLow (>30%)`, `NumberOfPeersLow (>50%)`, `ConnectRequestsHigh`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Symptoms

- `substrate_sub_libp2p_peers_count` is 0 or very low (<5)
- Logs show "Disconnected" or "Connection refused"
- Node is isolated — not receiving new blocks

## Quick Health Check

```bash
# Check peer count via RPC
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_health","params":[]}' \
  http://<node>:9933 | jq .

# List connected peers
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_peers","params":[]}' \
  http://<node>:9933 | jq '.result | length'
```

**Prometheus:**
```
substrate_sub_libp2p_peers_count{instance="<node>"}
```

**Loki:**
```logql
{instance="<node>"} |~ "peer|connect|disconnect|libp2p"
```

## Decision Tree

```
Low or zero peers
│
├─ Fleet-wide or single node?
│  ├─ Check substrate_sub_libp2p_peers_count across fleet
│  │
│  ├─ >50% of nodes affected?
│  │  └─ Likely infrastructure issue, not a node bug
│  │     Check: bootnode health, DNS changes, firewall/security group changes
│  │     Check: shared datacenter/region/provider issues
│  │
│  ├─ >30% of nodes affected?
│  │  └─ Monitor for escalation. Check if affected nodes share a common factor
│  │     (same region, same provider, same binary version)
│  │
│  └─ Single node → continue below
│
├─ Validators specifically?
│  └─ Validators need more peers than full nodes (validation protocol)
│     Expected: ~600 peers (Polkadot), ~700 peers (Kusama)
│     Alert threshold: < 90% of expected peer count
│
├─ 0 peers since startup?
│  ├─ Check firewall / security groups
│  │  └─ Port 30333 (or custom) must be open inbound + outbound
│  │
│  ├─ Check bootnodes reachable?
│  │  └─ telnet/nc to bootnode IP:port
│  │     If unreachable: network/firewall issue
│  │
│  └─ Wrong chain spec or genesis?
│     └─ Node won't connect to peers on different chain
│        Check: node logs for chain name at startup
│
├─ Had peers, then lost them?
│  ├─ Check if node was banned by peers
│  │  └─ Logs: "Banned" or "reputation"
│  │     Cause: node sending invalid data (corrupt DB, wrong fork)
│  │
│  ├─ Network issue on the host?
│  │  └─ Check: ping, traceroute, DNS resolution
│  │
│  └─ Resource exhaustion (too many connections)?
│     └─ Check: ulimit -n (file descriptors), netstat/ss connection count
│
└─ Peers connected but no data flowing?
   └─ Check logs for sync protocol issues
      Possible protocol version mismatch (outdated binary)
      → Resolution: Upgrade node binary
```

## Resolution

### Firewall / Security group

```bash
# Check if port is open locally
ss -tlnp | grep 30333

# Test connectivity from outside
nc -zv <node-ip> 30333

# If behind NAT, ensure port forwarding or use --public-addr flag
```

Ensure:
- **Inbound** TCP on P2P port (default 30333)
- **Outbound** TCP to peer P2P ports
- If using `--public-addr`, ensure it matches the actual public IP

### Bootnode issues

```bash
# Check configured bootnodes in chain spec
# Bootnodes are multiaddrs like: /dns/bootnode.polkadot.io/tcp/30333/p2p/<peer_id>

# Test bootnode connectivity
nc -zv bootnode.polkadot.io 30333
```

If bootnodes changed: update chain spec or use `--bootnodes` CLI flag to specify additional ones.

### Node banned by peers

If the node has a bad reputation (sending invalid blocks/transactions):
1. Check for DB corruption → see [not-syncing](not-syncing.md)
2. Check node binary version matches network
3. As last resort: purge chain + re-sync

### File descriptor limits

```bash
# Check current limits
ulimit -n
# Should be at least 10000 for a node

# Check actual usage
ls /proc/<node-pid>/fd | wc -l
```

If too low: increase in systemd unit or `/etc/security/limits.conf`.

## Escalation

- Collect: peer count history, firewall rules, bootnode reachability test results
- Check: are other nodes in same network/datacenter also affected?
- Escalate to: `[TODO: infra team / dev team contact]`
