# Runbook: Parachain Not Producing Blocks

> **Triggered by alert:** `CollatorNotProducingBlocks` (when collations built but not included)
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Symptoms

- Parachain block height not increasing
- Collator logs show collations built but not included
- Relay chain shows no new parachain blocks backed/included for this para ID

## Quick Health Check

**Confirm it's a chain-level issue, not a node issue:**
1. Is the collator node synced? (both relay + para chain) → if not, see [node/collator-operations](../node/collator-operations.md)
2. Is the collator producing collations locally? Check logs for `"Starting collation"` or `"Built collation"`
3. If the collator is building blocks but they're not appearing on-chain → continue here

**Loki:**
```logql
# Collation built but not included
{instance="<collator>"} |~ "collat|Built|backing|candidate"

# Check for parachain-specific errors
{instance="<collator>"} |~ "parachain" |= "ERROR"
```

**On-chain check (via relay chain RPC or explorer):**
- Check `paras.paraLifecycles(paraId)` — is the parachain active?
- Check recent relay chain blocks for `paraInherent` extrinsics including this para ID

## Decision Tree

```
Collator builds blocks but they're not included on relay chain
│
├─ Parachain not active on relay chain?
│  ├─ Check paras.paraLifecycles(paraId)
│  │  └─ Not found / Onboarding → see [onboarding](onboarding.md)
│  │
│  └─ Status is "Parathread" (on-demand) not "Parachain" (bulk)?
│     └─ On-demand parachains only get blocks when orders are placed
│        → Resolution: Coretime / on-demand orders
│
├─ No core assigned?
│  └─ Parachain has no coretime allocation
│     → see [coretime](coretime.md)
│
├─ Core assigned but collation not backed?
│  ├─ Validators not receiving collation?
│  │  └─ Network issue between collator and assigned validators
│  │     Check: collator peer count, validator group connectivity
│  │
│  ├─ Collation validation failing?
│  │  └─ PVF (Parachain Validation Function) execution error
│  │     Check relay chain validator logs for PVF errors
│  │     Possible: WASM runtime too heavy, execution timeout
│  │
│  └─ Collation backed but not included?
│     └─ Availability issue — not enough validators stored chunks
│        Check relay chain logs for availability/bitfield data
│
└─ Intermittent — some blocks included, some not?
   ├─ Multiple collators competing?
   │  └─ Only one collation per core per relay block
   │     Normal to skip some slots with multiple collators
   │
   └─ Collation timing?
      └─ Collation arrives too late for the relay block
         Check collation build time in logs
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
1. Check collator has relay chain peers (the validators need to be reachable)
2. Check logs for `"Advertised collation"` and `"Sent collation"`
3. If no validators are connecting: possible network partition
4. Try restarting the collator — sometimes fixes stale connections

### PVF execution failure

If validators reject the collation due to PVF errors:
1. Check **relay chain validator logs** (not collator) for:
   - `"PVF execution error"`, `"validation failed"`, `"timeout"`
2. Common causes:
   - Parachain runtime too heavy (execution > 2s hard limit)
   - WASM compilation issue after runtime upgrade
   - PoV (Proof of Validity) too large (>5 MB limit)
3. This requires **parachain runtime team** intervention

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
