# Runbook: Finalization Stall

> **Triggered by alerts:** `NodesFinalizationSlow (>30%)`, `NodesFinalizationSlow (>50%)`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Symptoms

- `substrate_block_height{status="finalized"}` not increasing
- `substrate_block_height{status="best"}` keeps increasing (blocks are produced but not finalized)
- Gap between best and finalized growing: `best - finalized > 20`
- Logs may show GRANDPA warnings

## Quick Health Check

**Prometheus:**
```
# Finalization gap (should be <10 normally)
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}

# GRANDPA round
substrate_finality_grandpa_round
```

**Loki:**
```logql
# GRANDPA activity
{instance="<node>"} |~ "grandpa|GRANDPA|finali"

# Look for specific errors
{instance="<node>"} |~ "grandpa" |= "ERROR"

# Dispute-related (relay chain)
{instance="<node>"} |~ "dispute"
```

```bash
# Check finalized block via RPC
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"chain_getFinalizedHead","params":[]}' \
  http://<node>:9933 | jq .
```

## Decision Tree

```
Finalized block not advancing
│
├─ Network-wide or single node?
│  ├─ Check if other nodes also stalled
│  │  └─ If network-wide: likely consensus issue → escalate immediately
│  │
│  └─ Single node only:
│     └─ This node may be partitioned or behind
│        Check peer count and sync status
│
├─ GRANDPA voter not running?
│  ├─ Node is not a validator → finalization is passive (follows network)
│  │  └─ If network finalizes but this node doesn't: DB or sync issue
│  │
│  └─ Node is a validator but GRANDPA not active
│     └─ Check keystore for GRANDPA key (prefix: 6772616e)
│        Check logs for GRANDPA initialization
│
├─ GRANDPA stuck in a round?
│  └─ Logs show same round number for extended period
│     Possible causes:
│     - Not enough validators online (need 2/3+1)
│     - Clock skew between validators
│     - Network partition among validators
│
├─ Dispute blocking finalization? (relay chain only)
│  └─ Active disputes prevent finalization by design
│     Check: {instance="<node>"} |~ "dispute"
│     If disputes are active: this is expected behavior, wait for resolution
│
└─ After node restart with snapshot/DB?
   └─ Initialization timing issue
      GRANDPA may try to finalize before subsystems are ready
      Usually resolves after first active leaf is processed
      If persistent: restart the node
```

## Resolution

### Single node behind on finalization

If the network is finalizing but this node isn't keeping up:
1. Check sync status — node must be fully synced
2. Check peer connectivity — needs peers to receive GRANDPA messages
3. Restart the node — clears any stuck GRANDPA state

### Network-wide finalization stall

**This is critical — escalate immediately** while performing initial checks.

**How to determine network-wide vs single-node:**
```
# Compare finalization gap across nodes
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}

# >50% of nodes with gap >20 = network-wide (alert: NodesFinalizationSlow >50%)
# >30% of nodes with gap >20 = trending issue (alert: NodesFinalizationSlow >30%)
# Single node with gap >20 = isolated node issue
```

**Note:** Measure finality lag as `best - finalized` gap, not just "finalized block not moving." A slow block production rate can be confused with a finalization stall. For collators with elastic scaling, finality thresholds may need adjustment.

1. **Check validator availability:**
   - How many validators are online?
   - GRANDPA requires >2/3 of validators to finalize
   - If validators went offline (e.g., deployment), finalization stalls

2. **Check for active disputes** (relay chain):
   - Disputes intentionally block finalization
   - Check dispute-coordinator logs across validators

3. **Check for common triggers:**
   - Recent runtime upgrade
   - Recent validator set rotation
   - Recent network deployment/restart of validators

### Post-restart finalization delay

After restarting a node (especially from a snapshot):
- There can be a brief period where GRANDPA catches up
- Finalization typically resumes within a few minutes
- If >10 minutes: check logs for GRANDPA errors and restart again

### Memory impact of finalization stalls

When finalization stalls, the state DB accumulates "journals" (uncommitted state changes) in memory. This can lead to OOM. If you see both:
- Finalization stall
- Growing memory usage

The memory issue is a **symptom** of the finalization stall, not the cause. Fix finalization first, and memory will stabilize as journals are flushed.

## Escalation

**Finalization stalls on production networks are high priority.**

- Collect: finalization gap, GRANDPA round number, dispute status, validator count
- Check: recent deployments or changes that correlate with the stall
- Escalate to: `[TODO: consensus team / on-call dev]`
