# Runbook: Coretime Issues

## Quick check

**1. What type of coretime?**
- **Bulk coretime**: reserved in advance via Coretime chain
- **On-demand coretime**: pay-per-block (formerly "parathreads")
- Not sure? Check `paras.paraLifecycles(paraId)` → `Parachain` = bulk, `Parathread` = on-demand

**2. Is the parachain active?**
```
paras.paraLifecycles(paraId)
# Expected: "Parachain" (bulk) or "Parathread" (on-demand)
```

**3. Without coretime, a parachain cannot have blocks included on the relay chain.**

## Triage

### Bulk coretime

1. **Allocation active?**
   - Check expiry on Coretime chain
   - Expired → see [Bulk coretime expired](#bulk-coretime-expired)

2. **Allocation active but no blocks included?**
   - Core is assigned but collator isn't using it
   - See [not-producing](not-producing.md)

3. **Wrong para ID on allocation?**
   - Verify the assignment maps to the correct para ID

### On-demand coretime

1. **Orders being placed?**
   - No → application must submit on-demand orders. Not SRE — notify parachain team.

2. **Orders placed but not fulfilled?**
   - Queue may be congested. Check queue depth and spot price.
   - See [On-demand queue congested](#on-demand-queue-congested)

3. **Order fulfilled but block not produced?**
   - Collator must produce within the assigned slot
   - See [not-producing](not-producing.md)

## Deep Investigation

### Useful Loki queries

```logql
# Core assignment logs on validators
{instance="<validator>"} |~ "coretime|assignment|core.*sched"
```

### On-chain checks

```
# Check on-demand queue
onDemandAssignmentProvider.freeEntries()
```

## Resolution

### Bulk coretime expired

1. Check expiry on Coretime chain (system parachain)
2. Renewal must be done before or shortly after expiry
3. This requires a transaction on the Coretime chain — not an SRE fix
4. Notify the parachain team / governance to purchase or renew

### On-demand queue congested

If on-demand orders are queued but not being scheduled:
1. Check queue depth — many parachains competing for on-demand cores
2. Check spot price — price increases with demand
3. Options:
   - Increase bid price
   - Switch to bulk coretime for predictable scheduling
4. This is an economic/application decision, not SRE

### Core assigned but no blocks

If the parachain has coretime but blocks aren't being produced:
- The issue is at the collator/validation level, not coretime
- Go to [not-producing](not-producing.md)

## Escalation

- Collect: para ID, coretime type (bulk/on-demand), allocation status, expiry info
- Check: Coretime chain state, relay chain scheduler state
- Escalate to: `[TODO: coretime / parachain team]`
