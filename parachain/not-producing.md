# Runbook: Parachain Not Producing Blocks

> **Triggered by alert:** `CollatorNotProducingBlocks` (when collations built but not included)
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. Is this a node issue or a chain-level issue?**
- Is the collator node synced? (both relay + parachain) → if not, see [node/collator-operations](../node/collator-operations.md)
- Is the collator producing collations locally? Check logs for collation-related messages
- Collator building blocks but they're not appearing on-chain → continue here

**2. Is the parachain active?**
- Check `paras.paraLifecycles(paraId)` on relay chain
- Not found / Onboarding → see [onboarding](onboarding.md)

**3. Does it have a core?**
- Parachain needs coretime to get blocks included
- See [coretime](coretime.md) if no core assigned

## Triage

1. **Parachain not active on relay chain?**
   - `paras.paraLifecycles(paraId)` → not found or `Onboarding` → see [onboarding](onboarding.md)
   - Status `Parathread` (on-demand) not `Parachain` (bulk)? → on-demand parachains only get blocks when orders are placed. See [Coretime / on-demand orders](#coretime--on-demand-orders)

2. **No core assigned?**
   - Parachain has no coretime allocation → see [coretime](coretime.md)

3. **Core assigned but collation not backed?**
   - Validators not receiving collation? → network issue between collator and assigned validators. Check collator peer count, validator group connectivity. See [Validators not receiving collation](#validators-not-receiving-collation)
   - Collation validation failing? → PVF execution error on relay chain validators. See [PVF execution failure](#pvf-execution-failure)
   - Collation backed but not included? → availability issue, not enough validators stored chunks. Check relay chain logs for availability/bitfield data

4. **Intermittent — some blocks included, some not?**
   - Multiple collators competing? → only one collation per core per relay block, normal to skip some slots
   - Collation timing? → collation arrives too late for the relay block. See [Collation arriving too late](#collation-arriving-too-late)

## Deep Investigation

### Useful Loki queries

```logql
# Collation built but not included
{instance="<collator>"} |~ "collat|Built|backing|candidate"

# Parachain-specific errors
{instance="<collator>"} |~ "parachain" |= "ERROR"
```

### On-chain checks

```
# Check parachain lifecycle
paras.paraLifecycles(paraId)

# Check recent relay chain blocks for parachain inclusion
# Look for paraInherent extrinsics including this para ID
```

## Resolution

### Coretime / on-demand orders

If parachain is using **on-demand coretime** (parathread):
- Blocks are only produced when coretime is purchased per-block
- Check if orders are being placed — this is an application-level concern
- If the parachain should have **bulk coretime**: verify assignment on-chain

See [coretime](coretime.md) for details.

### Validators not receiving collation

The collator advertises collations to the assigned validator group:
1. Check collator has relay chain peers (validators need to be reachable)
2. Check logs for `"Advertised collation"` and `"Sent collation"`
3. If no validators are connecting: possible network partition
4. Try restarting the collator — sometimes fixes stale connections

### PVF execution failure

If validators reject the collation due to PVF errors:
1. Check **relay chain validator logs** (not collator) for: `"PVF execution error"`, `"validation failed"`, `"timeout"`
2. Common causes:
   - Parachain runtime too heavy (execution > 2s hard limit)
   - WASM compilation issue after runtime upgrade
   - PoV (Proof of Validity) too large (>5 MB limit)
3. This requires **parachain runtime team** intervention
4. See also [node/block-production — PVF bottleneck](../node/block-production.md#pvf-validation-bottleneck-validator-side)

### Collation arriving too late

The collator must build and submit the collation within the relay chain slot window:
1. Check collation build time in logs
2. If consistently slow: parachain runtime may be too heavy
3. Check collator CPU — block building is CPU-bound
4. Consider reducing block weight limits in parachain runtime config

## Escalation

- Collect: para ID, collator logs, relay chain validator logs (for PVF errors)
- Check: `paras.paraLifecycles`, coretime assignment, last included block number
- Escalate to: `[TODO: parachain team / core protocol team]`
