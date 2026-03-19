# Runbook: Peer Connectivity Issues

> **Triggered by alerts:** `NumberOfPeersLow (>30%)`, `NumberOfPeersLow (>50%)`, `ConnectRequestsHigh`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. Fleet-wide or single node?**
- Check `substrate_sub_libp2p_peers_count` across fleet
- \>50% affected → infrastructure issue (bootnode health, DNS, firewall/security group changes, datacenter/provider)
- \>30% affected → monitor, check if affected nodes share a common factor (region, provider, binary version)
- Single node → continue

**2. How many peers?**
```bash
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_peers","params":[]}' \
  http://<node>:9944 | jq '.result | length'
```

**3. Has it ever had peers?**
- 0 since startup → likely firewall/bootnode issue
- Had peers, then lost them → likely ban/network/resource issue

## Triage

### 0 peers since startup

1. **Firewall / security groups?**
   - Port 30333 (or custom) must be open inbound + outbound
   - See [Firewall resolution](#firewall--security-group)

2. **Bootnodes reachable?**
   - `nc -zv <bootnode-ip> <port>`
   - Unreachable → network/firewall issue or bootnode down
   - See [Bootnode resolution](#bootnode-issues)

3. **Wrong chain spec or genesis?**
   - Node won't connect to peers on a different chain
   - Check node logs for chain name at startup

### Had peers, then lost them

1. **Node banned by peers?**
   - Logs: `"Banned"` or `"reputation"`
   - Cause: node sending invalid data (corrupt DB, wrong fork)
   - See [Ban resolution](#node-banned-by-peers)

2. **Network issue on the host?**
   - Check: `ping`, `traceroute`, DNS resolution

3. **Resource exhaustion?**
   - Check `ulimit -n` (file descriptors) and connection count: `ss -tn | grep :30333 | wc -l`
   - See [File descriptor resolution](#file-descriptor-limits)

### Peers connected but no data flowing

- Check logs for sync protocol issues
- Possible protocol version mismatch (outdated binary) → upgrade node binary

### Validators specifically

- Validators need more peers than full nodes (validation protocol)
- Expected: ~600 peers (Polkadot), ~700 peers (Kusama)
- Alert threshold: < 90% of expected peer count

## Deep Investigation

### Useful Loki queries

```logql
{instance="<node>"} |~ "peer|connect|disconnect|libp2p"
```

### Useful Prometheus queries

```promql
substrate_sub_libp2p_peers_count{instance="<node>"}
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
