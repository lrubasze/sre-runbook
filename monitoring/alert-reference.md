# Alert Reference

Entry point for on-call operators. Find your alert below, follow the first steps, then jump to the full runbook.

## Critical Alerts

### VMNodeProcessCrashed

**Node crashed and restarted.**

| First steps | |
|---|---|
| 1. Check crash cause | `dmesg \| grep -i "oom\|killed process"` |
| 2. Check for panic | `journalctl -u <service> \| grep -i "panic\|segfault\|SIGSEGV"` |
| 3. Check DB corruption | Look for "database", "corruption" errors in logs |

**Then go to:** [node/crash-triage.md](../node/crash-triage.md)

---

### NetworkBlockProductionStalled

**No blocks produced for >5 minutes.**

| First steps | |
|---|---|
| 1. Network or single node? | Check `substrate_block_height{status="best"}` across multiple nodes |
| 2. If network-wide | Escalate immediately — possible consensus failure |
| 3. If single node | Check sync status, peers, validator role |

- **Network-wide:** Escalate + check recent deployments, validator count, disputes
- **Single node — validator:** [node/block-production.md](../node/block-production.md)
- **Single node — collator:** [node/collator-operations.md](../node/collator-operations.md) → [parachain/not-producing.md](../parachain/not-producing.md)

---

### NodesFinalizationSlow (>50%)

**More than 50% of nodes not finalizing for >5 minutes.**

| First steps | |
|---|---|
| 1. Check finalization gap | `substrate_block_height{status="best"} - substrate_block_height{status="finalized"}` |
| 2. How many nodes affected? | Check gap across fleet — >50% means likely network-wide |
| 3. Check for disputes | `{instance="<node>"} \|~ "dispute"` |

- **Network-wide (>50%):** Likely GRANDPA consensus issue or dispute — escalate immediately
- **Check:** validator availability (need >2/3+1), recent runtime upgrades, recent validator set rotations

**Then go to:** [node/finalization-stall.md](../node/finalization-stall.md) (see "Network-wide vs single-node" section)

---

### NumberOfPeersLow (>50%)

**More than 50% of nodes have <3 peers for >15 minutes.**

| First steps | |
|---|---|
| 1. Fleet-wide or localized? | Check if affected nodes share datacenter, region, or provider |
| 2. Bootnode health | Verify bootnodes are reachable: `nc -zv <bootnode-ip> 30333` |
| 3. Recent infra changes? | DNS, firewall, security group, or network changes |

- **Fleet-wide:** Likely infrastructure or bootnode issue — not a node bug
- **Localized to region/provider:** Network or firewall change

**Then go to:** [node/peer-connectivity.md](../node/peer-connectivity.md) (see "Fleet-wide vs single-node" section)

---

### UnhealthyRPCEndpoint

**WebSocket RPC endpoint unhealthy for >10 minutes.**

| First steps | |
|---|---|
| 1. Is the node process running? | `systemctl status <service>` |
| 2. Is the RPC port responding? | `curl -s http://<node>:9944/health` |
| 3. Is the node overloaded? | Check CPU, memory, open connections |

**Then go to:** [node/rpc-issues.md](../node/rpc-issues.md)

---

### UnhealthyRPCCloudflareLBPool

**Cloudflare LB has an unhealthy pool for >10 minutes.**

| First steps | |
|---|---|
| 1. Which pool members are down? | Check Cloudflare dashboard or API |
| 2. Are the backend nodes healthy? | Test each backend RPC directly |
| 3. Is it a Cloudflare issue? | Check [Cloudflare status](https://www.cloudflarestatus.com/) |

- **All backends unhealthy:** Likely node-side issue → check each node individually
- **Some backends unhealthy:** Specific node issues → [node/rpc-issues.md](../node/rpc-issues.md)
- **Backends healthy but LB unhealthy:** Cloudflare config or health check issue

---

### NodesStoppedValidating

**A Validator is not in the active set.**

| First steps | |
|---|---|
| 1. Is the node running? | `systemctl status <service>` |
| 2. Check node role | `system_nodeRoles` RPC — must return `["Authority"]` |
| 3. Check session keys | `author_hasSessionKeys` RPC — must return `true` |

- **Node running, keys present:** Validator may have been rotated out (on-chain issue, not SRE)
- **Node running, keys missing:** Keystore issue → [node/block-production.md](../node/block-production.md)
- **Node not running:** Check crash cause → [node/crash-triage.md](../node/crash-triage.md)

---

### CollatorNotProducingBlocks

**Collator has not produced blocks for >50 minutes.**

| First steps | |
|---|---|
| 1. Is the collator in the active set? | Check on-chain collator status for the parachain |
| 2. Is relay chain synced? | Check relay chain block height on the collator |
| 3. Are collations being built? | `{instance="<collator>"} \|~ "Starting collation\|Built collation"` |

- **Collator not in set:** On-chain issue, not SRE → notify parachain team
- **Relay chain not synced:** [node/collator-operations.md](../node/collator-operations.md)
- **Collations built but not included:** [parachain/not-producing.md](../parachain/not-producing.md)
- **No collations being built:** [node/collator-operations.md](../node/collator-operations.md)

---

## Warning Alerts

### NumberOfPeersLow (>30%)

**More than 30% of nodes have <3 peers for >15 minutes.**

Same investigation as critical version above, but lower urgency. Monitor for escalation to >50%.

**Note for validators:** Expected peer count is higher due to validation protocol peers (~600 Polkadot, ~700 Kusama). Use `< 90% of expected` as the threshold rather than <3.

**Then go to:** [node/peer-connectivity.md](../node/peer-connectivity.md)

---

### NodesFinalizationSlow (>30%)

**More than 30% of nodes not finalizing for >5 minutes.**

Same investigation as critical version above, but lower urgency. Monitor for escalation to >50%.

**Note:** Measure finality lag as `best - finalized` rather than just "not finalizing." For collators with elastic scaling, finality thresholds need adjustment.

**Then go to:** [node/finalization-stall.md](../node/finalization-stall.md)

---

### SlowRPCCallTime

**RPC calls taking >5 seconds for >3 hours.**

| First steps | |
|---|---|
| 1. Which calls are slow? | Check RPC access logs for slow endpoints |
| 2. Is the node overloaded? | Check CPU, memory — is it swapping? |
| 3. Is it query-specific? | State queries on archive nodes can be inherently slow |

**Then go to:** [node/rpc-issues.md](../node/rpc-issues.md)

---

### ConnectRequestsHigh

**Bootnode receiving >1000 requests/sec for >15 minutes.**

| First steps | |
|---|---|
| 1. Is a new chain/version launching? | Spikes expected during releases |
| 2. Is it a DDoS? | Check source IP distribution |
| 3. Are nodes failing to connect? | High requests + low success = bootnode overload |

- **Expected spike (release/restart):** Monitor, should subside
- **Sustained overload:** Consider adding bootnode capacity, rate limiting
- **Partial context:** [node/peer-connectivity.md](../node/peer-connectivity.md) for client-side peer issues

---

## Notice Alerts

### VMNodeProcessManualRestart

**Node was manually restarted.**

Informational only. No action required unless followed by other alerts.

- If node doesn't come back healthy within 5 minutes, check logs
- Post-restart finalization delay is normal (see [node/finalization-stall.md](../node/finalization-stall.md#post-restart-finalization-delay))
