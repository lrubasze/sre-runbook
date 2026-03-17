# Runbook: Runtime Upgrade Checklist

## Overview

Runtime upgrades change the on-chain logic (WASM) without requiring a node binary update. However, they can introduce behavioral changes that affect node operation.

## Pre-Upgrade Checklist

- [ ] **Know the block number** — when will the upgrade enact?
- [ ] **Read the release notes** — what changed? any breaking changes?
- [ ] **Check node binary compatibility** — some runtime upgrades require a minimum client version
- [ ] **Ensure monitoring is active** — watch dashboards during the upgrade window
- [ ] **Notify the team** — ensure on-call knows the upgrade is happening

## What to Monitor During Upgrade

**Block of enactment:**
```logql
# Watch for runtime upgrade execution
{instance="<node>"} |~ "runtime|Runtime|upgrade|wasm"
```

**Key metrics to watch:**
```
# Block production should continue uninterrupted
rate(substrate_block_height{status="best"}[1m])

# Finalization should continue
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}

# Block execution time may spike briefly
substrate_block_height_number

# Peer count should remain stable
substrate_sub_libp2p_peers_count
```

## Post-Upgrade Checks

- [ ] Blocks are still being produced at normal rate
- [ ] Finalization is progressing (gap not growing)
- [ ] No new ERROR logs related to runtime
- [ ] Peer count stable
- [ ] Memory/CPU within normal range (runtime may change resource profile)

## Common Issues After Runtime Upgrade

| Issue | Possible cause | Action |
|---|---|---|
| Node crashes after upgrade block | Client binary too old for new runtime | Upgrade node binary |
| Block production slows | New runtime heavier (more computation) | Monitor; may need more CPU |
| New ERROR logs appearing | Runtime behavior change exposed client issue | Collect logs, escalate |
| Peer disconnections | Peers on incompatible versions | Check if peers need updates |

## Rollback

Runtime upgrades **cannot be rolled back** directly — they require governance to enact a new upgrade. If the upgrade causes critical issues:

1. **Do not restart nodes** unless they are crashing — restarting won't undo the runtime upgrade
2. **Collect evidence** — logs, metrics, block numbers where issues started
3. **Escalate immediately** to runtime/governance team
4. A fix requires either:
   - A new runtime upgrade via governance (takes time)
   - A client-side workaround deployed to nodes

## Escalation

- Collect: exact block of upgrade, symptoms, logs before/after, binary versions
- Escalate to: `[TODO: runtime team / governance contact]`
