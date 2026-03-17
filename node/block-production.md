# Runbook: Block Production Issues

## Symptoms

- Validator is missing its authoring slots
- Collator is not producing parachain blocks
- Logs show "Skipping slot" or "failed to propose"

## Quick Health Check

```logql
# Check for authoring-related logs
{instance="<node>"} |~ "author|slot|propose|Pre-sealed"

# Check for errors during block production
{instance="<node>"} |~ "author|propose" |= "ERROR"
```

```bash
# Check if node considers itself a validator
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_nodeRoles","params":[]}' \
  http://<node>:9933 | jq .
# Expected: ["Authority"] for validators
```

**Prometheus:**
```
# Is the node keeping up with the chain?
substrate_block_height{status="best"}
substrate_block_height{status="finalized"}

# Gap between best and finalized (should be small, <10)
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}
```

## Decision Tree

```
Validator/collator not producing blocks
│
├─ Node not synced?
│  └─ Must be fully synced to produce blocks
│     Go to [not-syncing](not-syncing.md)
│
├─ Node role is not "Authority"?
│  └─ Keystore issue — validator keys not loaded
│     → Resolution: Keystore
│
├─ Keys present but not in active set?
│  └─ Validator not elected / collator not registered
│     → Resolution: On-chain status (not an SRE issue, notify team)
│
├─ Logs show "Skipping slot" or "not leader"?
│  └─ Normal — BABE only assigns some slots to each validator
│     Only a problem if the node NEVER gets a slot
│
├─ Logs show timeout or execution errors?
│  └─ Block execution too slow / runtime issue
│     → Resolution: Resource bottleneck or runtime bug
│
└─ Collator-specific: not producing?
   ├─ Is the relay chain node synced?
   │  └─ Collator embeds a relay chain node. Both must be synced.
   │
   └─ Is the collator registered on-chain?
      └─ Check with the parachain team
```

## Resolution

### Keystore issues

The keystore holds the validator's session keys. Without them, the node can't sign blocks.

1. **Check keystore directory exists** and contains key files:
   ```bash
   ls <base-path>/chains/<chain>/keystore/
   # Should contain files like: 62616265<hex>  (babe), 6772616e<hex> (grandpa), etc.
   ```
2. **Verify keys match on-chain session keys:**
   ```bash
   # Check via RPC
   curl -s -H "Content-Type: application/json" \
     -d '{"id":1,"jsonrpc":"2.0","method":"author_hasSessionKeys","params":["<session_keys_hex>"]}' \
     http://<node>:9933 | jq .
   # Expected: {"result": true}
   ```
3. If keys are missing: **re-insert session keys** (coordinate with the validator operator)

### Resource bottleneck during authoring

If block production times out (>2s for relay chain, >0.5s for parachains):
- Check CPU usage during the slot
- Check if state trie is too large (deep storage reads)
- See [high-resource-usage](high-resource-usage.md)

## Escalation

- Collect: node role, key presence, sync status, logs from missed slots
- For collators: relay chain sync status + parachain sync status
- Escalate to: `[TODO: dev team contact / channel]`
