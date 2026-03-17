# Runbook: Parachain Onboarding Issues

## Symptoms

- New parachain not appearing as active on relay chain
- Parachain stuck in "Onboarding" state
- Genesis head or validation code not accepted

## Background

Parachain onboarding steps:
1. **Reserve a para ID** on the relay chain
2. **Register** genesis head (state) and validation code (WASM)
3. **Acquire coretime** (bulk or on-demand)
4. Parachain transitions: `Onboarding → Parathread/Parachain`

This process takes **2 session boundaries** (~2 hours on Polkadot, ~10 minutes on testnets).

## Quick Health Check

**On-chain:**
```
# Check lifecycle status
paras.paraLifecycles(paraId)
# Possible values: Onboarding, Parathread, Parachain, UpgradingParathread,
#                  DowngradingParachain, OffboardingParathread, OffboardingParachain

# Check if code and head are registered
paras.heads(paraId)
paras.currentCodeHash(paraId)
```

## Decision Tree

```
Parachain not active
│
├─ Not registered at all?
│  └─ para ID not found in paras.paraLifecycles
│     → Registration transaction not submitted or failed
│     Check: registrar.register extrinsic in recent blocks
│
├─ Status: "Onboarding"?
│  └─ This is normal — wait for 2 session boundaries
│     Check current session: session.currentIndex
│     If stuck >3 sessions: may be a registration issue
│
├─ Status: "Parathread" but expected "Parachain"?
│  └─ Parathread = on-demand coretime only
│     Needs bulk coretime assignment to become "Parachain"
│     → see [coretime](coretime.md)
│
├─ Status: "OffboardingParathread"?
│  └─ Parachain is being removed
│     This is a governance/admin action — check if intentional
│
└─ Registration failed?
   ├─ Validation code too large?
   │  └─ Max code size is ~5MB (relay chain configured)
   │     Check: configuration.activeConfig().max_code_size
   │
   ├─ Deposit insufficient?
   │  └─ Registration requires a deposit
   │     Check: registrar.registrationDeposit
   │
   └─ Genesis head invalid?
      └─ Must be valid state root for the parachain runtime
         Re-export genesis with correct runtime
```

## Resolution

### Stuck in Onboarding

If the parachain has been in "Onboarding" for more than 3 sessions:
1. Verify both `paras.heads(paraId)` and `paras.currentCodeHash(paraId)` return values
2. If either is missing: registration was incomplete — re-submit
3. Check relay chain logs around session boundaries for errors mentioning this para ID
4. Escalate if all looks correct but state doesn't transition

### Registration failed

1. Check the registration extrinsic result — look for dispatch errors
2. Common issues:
   - Insufficient balance for deposit
   - Para ID already taken
   - Code size exceeds limit
3. Fix the issue and re-submit the registration transaction

### Collator can't connect after onboarding

Once the parachain is active:
1. Start the collator with the correct chain spec and para ID
2. Ensure the collator connects to the relay chain
3. Wait for relay chain sync on the collator
4. First parachain block may take a few minutes after relay chain sync completes

## Escalation

- Collect: para ID, lifecycle status, registration extrinsic hash, session index
- Escalate to: `[TODO: parachain team]`
