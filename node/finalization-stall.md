# Runbook: Finalization Stall

> **Triggered by alerts:** `NodesFinalizationSlow (>30%)`, `NodesFinalizationSlow (>50%)`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. Network-wide or single node?**
- Compare finalization gap across nodes:
  `substrate_block_height{status="best"} - substrate_block_height{status="finalized"}`
- \>50% of nodes with gap >20 → **network-wide, escalate immediately**
- \>30% of nodes with gap >20 → trending, monitor closely
- Single node → continue

**2. Is best block still advancing?**
- `substrate_block_height{status="best"}` increasing but `{status="finalized"}` stuck = finalization stall
- Both stuck = different problem, see [not-syncing](not-syncing.md)

**3. What's the gap?**
```bash
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"chain_getFinalizedHead","params":[]}' \
  http://<node>:9944 | jq .
```

## Triage

### Single node behind on finalization

1. **Node synced?** → must be fully synced to receive GRANDPA messages
2. **Has peers?** → needs peers to receive GRANDPA votes
3. **After restart?** → GRANDPA may take a few minutes to catch up. Normal if <10 minutes. If >10 minutes: restart again
4. None of the above → [Escalation](#escalation)

### Network-wide finalization stall

**This is critical — escalate immediately while performing these checks.**

1. **Validator availability?**
   - How many validators are online?
   - GRANDPA requires >2/3 of validators to finalize
   - If validators went offline (e.g., deployment), finalization stalls

2. **Active disputes?** (relay chain)
   - Disputes intentionally block finalization
   - Check dispute-coordinator logs across validators
   - If disputes are active: expected behavior, wait for resolution

3. **Common triggers?**
   - Recent runtime upgrade
   - Recent validator set rotation
   - Recent network deployment/restart of validators

### GRANDPA stuck in a round

- Logs show same round number for extended period
- Possible causes: not enough validators online (need 2/3+1), clock skew, network partition

### GRANDPA voter not running

- Node is a validator but GRANDPA not active
- Check keystore for GRANDPA key (prefix: `6772616e`)
- Check logs for GRANDPA initialization

## Deep Investigation

**Note:** Measure finality lag as `best - finalized` gap, not just "finalized block not moving." A slow block production rate can be confused with a finalization stall. For collators with elastic scaling, finality thresholds may need adjustment.

### Useful Loki queries

```logql
# GRANDPA activity
{instance="<node>"} |~ "grandpa|GRANDPA|finali"

# GRANDPA errors
{instance="<node>"} |~ "grandpa" |= "ERROR"

# Dispute-related (relay chain)
{instance="<node>"} |~ "dispute"
```

### Useful Prometheus queries

```promql
# Finalization gap
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}

# GRANDPA round
substrate_finality_grandpa_round
```

### Memory impact of finalization stalls

When finalization stalls, the state DB accumulates "journals" (uncommitted state changes) in memory. This can lead to OOM. If you see both:
- Finalization stall
- Growing memory usage

The memory issue is a **symptom** of the finalization stall, not the cause. Fix finalization first, and memory will stabilize as journals are flushed.

## Resolution

### Single node — restart

1. Check sync status — node must be fully synced
2. Check peer connectivity — needs peers to receive GRANDPA messages
3. Restart the node — clears any stuck GRANDPA state

### Network-wide — escalate + investigate

1. Check validator availability (>2/3 online?)
2. Check for active disputes
3. Check for common triggers (runtime upgrade, validator rotation, deployment)

## Escalation

**Finalization stalls on production networks are high priority.**

- Collect: finalization gap, GRANDPA round number, dispute status, validator count
- Check: recent deployments or changes that correlate with the stall
- Escalate to: `[TODO: consensus team / on-call dev]`
