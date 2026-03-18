# Node Runbooks

Debugging and operational guides for Polkadot/Kusama node issues.

**Start here:** [overview.md](overview.md) — node architecture primer.

## Runbooks

| Runbook | When to use |
|---|---|
| [not-syncing](not-syncing.md) | Node stuck, not importing blocks, falling behind |
| [block-production](block-production.md) | Validator/collator not authoring blocks |
| [peer-connectivity](peer-connectivity.md) | 0 peers, peer drops, connection issues |
| [high-resource-usage](high-resource-usage.md) | OOM, high CPU, disk full |
| [finalization-stall](finalization-stall.md) | Best block advancing, finalized block stuck |
| [collator-operations](collator-operations.md) | Parachain collator specific issues |
| [crash-triage](crash-triage.md) | Node crashed — OOM, panic, segfault diagnosis |
| [rpc-issues](rpc-issues.md) | RPC unhealthy, slow calls, LB pool failures |
