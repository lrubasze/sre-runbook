# Runbook: Parachain Onboarding Issues

## Quick check

**1. Is the parachain registered?**
```
paras.paraLifecycles(paraId)
# Not found → registration not submitted or failed
# "Onboarding" → normal, wait for 2 session boundaries
```

**2. How long has it been onboarding?**
- Onboarding takes **2 session boundaries** (~2 hours on Polkadot, ~10 minutes on testnets)
- Check current session: `session.currentIndex`
- Stuck >3 sessions → see [Stuck in Onboarding](#stuck-in-onboarding)

**3. Are code and head registered?**
```
paras.heads(paraId)
paras.currentCodeHash(paraId)
# Both must return values
```

## Triage

1. **Not registered at all?**
   - Para ID not found in `paras.paraLifecycles`
   - Registration transaction not submitted or failed
   - Check `registrar.register` extrinsic in recent blocks

2. **Status: `Onboarding`?**
   - Normal — wait for 2 session boundaries
   - Stuck >3 sessions → see [Stuck in Onboarding](#stuck-in-onboarding)

3. **Status: `Parathread` but expected `Parachain`?**
   - `Parathread` = on-demand coretime only
   - Needs bulk coretime assignment to become `Parachain`
   - See [coretime](coretime.md)

4. **Status: `OffboardingParathread`?**
   - Parachain is being removed — check if intentional (governance/admin action)

5. **Registration failed?**
   - Validation code too large? Max ~5MB: `configuration.activeConfig().max_code_size`
   - Deposit insufficient? Check `registrar.registrationDeposit`
   - Genesis head invalid? Must be valid state root for the parachain runtime

## Resolution

### Stuck in Onboarding

If the parachain has been in `Onboarding` for more than 3 sessions:
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
