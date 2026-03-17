# Runbook: Coretime Issues

## Symptoms

- Parachain not getting relay chain slots to include blocks
- Coretime expiring / expired
- On-demand parachain orders not being fulfilled

## Background

**Coretime** is how parachains get execution time on the relay chain:
- **Bulk coretime**: reserved blocks of execution time, purchased in advance (via Coretime chain)
- **On-demand coretime**: pay-per-block, purchased individually (formerly "parathreads")

Without coretime, a parachain cannot have blocks included on the relay chain.

## Quick Health Check

**On-chain (relay chain):**
```
# Check if parachain has a core assigned
# Use Coretime chain or relay chain state

# Check parachain lifecycle
paras.paraLifecycles(paraId)
# Expected: "Parachain" (bulk) or "Parathread" (on-demand)

# Check on-demand queue
onDemandAssignmentProvider.freeEntries()
```

**Loki:**
```logql
# Core assignment logs on validators
{instance="<validator>"} |~ "coretime|assignment|core.*sched"
```

## Decision Tree

```
Parachain not getting slots
│
├─ Bulk coretime?
│  ├─ Check if allocation is active
│  │  └─ Expired? → need to renew on Coretime chain
│  │
│  ├─ Allocation active but no blocks included?
│  │  └─ Core is assigned but collator isn't using it
│  │     → see [not-producing](not-producing.md)
│  │
│  └─ Wrong para ID on allocation?
│     └─ Verify the assignment maps to the correct para ID
│
├─ On-demand coretime?
│  ├─ Orders being placed?
│  │  └─ No → application must submit on-demand orders
│  │     Not an SRE issue — notify the parachain team
│  │
│  ├─ Orders placed but not fulfilled?
│  │  └─ On-demand queue may be congested
│  │     Check queue depth and spot price
│  │
│  └─ Order fulfilled but block not produced?
│     └─ Collator must produce within the assigned slot
│        → see [not-producing](not-producing.md)
│
└─ Not sure which type?
   └─ Check paras.paraLifecycles(paraId)
      "Parachain" = bulk, "Parathread" = on-demand
```

## Resolution

### Bulk coretime expired

1. **Check expiry on Coretime chain** (system parachain)
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
