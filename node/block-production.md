# Runbook: Block Production Issues

> **Triggered by alerts:** `NetworkBlockProductionStalled`, `NodesStoppedValidating`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Symptoms

- Validator is missing its authoring slots
- Collator is not producing parachain blocks
- Logs show `"Skipping slot: major sync is in progress."` or `"Proposing failed: <error>"`
- No `"🔖 Pre-sealed block for proposal at ..."` logs when the node should be authoring

## Quick Health Check

```logql
# Check for authoring-related logs (target: "babe" for validators, "aura" for collators)
{instance="<node>"} |~ "Starting authorship|Claimed slot|Pre-sealed|Proposing failed|Skipping slot"

# Check for errors during block production
{instance="<node>"} |~ "slots|babe|aura" |~ "WARN|ERROR"
```

```bash
# Check if node considers itself a validator/authority
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_nodeRoles","params":[]}' \
  http://<node>:9944 | jq .
# Expected: ["Authority"] for validators/collators
# If ["Full"]: node was not started with --validator flag
```

**Prometheus:**
```promql
# Is the node keeping up with the chain?
substrate_block_height{status="best"}
substrate_block_height{status="finalized"}

# Gap between best and finalized (should be small, <10)
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}
```

## Decision Tree

> **Note:** Relay chain validators use **BABE** consensus (VRF-based slot assignment).
> Parachain collators use **AURA** consensus (round-robin slot assignment).
> The debugging steps differ — see collator-specific section below.

```
Validator/collator not producing blocks
│
├─ 1. Network-wide or single node?
│  ├─ Check substrate_block_height{status="best"} across multiple nodes
│  │  └─ If ALL nodes stopped: network-wide consensus failure → escalate immediately
│  │     Check: recent runtime upgrade, validator availability, dispute status
│  │
│  └─ Single node only → continue below
│
├─ 2. Node synced?
│  ├─ Check substrate_block_height{status="best"} against chain head
│  ├─ Logs: "Skipping slot: major sync is in progress." → still syncing
│  └─ Not synced → go to [not-syncing](not-syncing.md)
│
├─ 3. Started with --validator flag?
│  ├─ Check: system_nodeRoles RPC → must return ["Authority"]
│  ├─ If ["Full"] → node was NOT started with --validator
│  │  └─ Fix: restart the node with --validator flag
│  └─ If ["Authority"] → continue below
│
├─ 4. Session keys in keystore?
│  ├─ Check keystore directory for key files (see Resolution: Keystore)
│  ├─ Check: author_hasSessionKeys RPC → must return true
│  └─ Keys missing → Resolution: Keystore issues
│
├─ 5. In active validator set / collator registered?
│  ├─ Validator: check session.validators() on-chain
│  ├─ Collator: check on-chain registration
│  └─ Not in set → not an SRE issue, notify team
│
├─ 6. Check authoring logs
│  ├─ "Starting authorship at slot: <N>" → slot was claimed, block build attempted
│  │  ├─ Followed by "🔖 Pre-sealed block for proposal at <N>" → block was built ✓
│  │  ├─ Followed by "Proposing failed: <err>" → proposal error, check error details
│  │  ├─ Followed by "⌛️ Discarding proposal for slot <N>; block production took too long"
│  │  │  └─ Block build exceeded slot deadline → Resolution: Resource bottleneck
│  │  └─ No follow-up log → block build hung, check CPU/IO
│  │
│  ├─ "Claimed slot <N>" (BABE debug) but no "Starting authorship"
│  │  └─ Slot was claimed but inherent data or proposer creation failed
│  │     Check for: "Unable to author block" or "Unable to create inherent data" warns
│  │
│  └─ No authoring logs at all?
│     └─ BABE: normal — VRF only assigns some slots to each validator
│        Only a problem if the node NEVER gets a slot across multiple epochs
│
└─ 7. Collator-specific issues → see below
```

### Collator-specific decision tree

Parachain collators use AURA consensus and have additional dependencies
beyond what relay chain validators need.

```
Collator not producing parachain blocks
│
├─ Is the embedded relay chain synced?
│  ├─ Collator runs an embedded relay chain client — both chains must be synced
│  ├─ Check relay chain sync: compare relay best block vs network head
│  └─ Not synced → wait, or check relay chain peer connectivity
│
├─ Does the parachain have a core assigned?
│  ├─ Bulk coretime: check on.demand.core_count or coretime allocation
│  ├─ On-demand: check if order was placed and core assigned
│  └─ No core → see [parachain/coretime](../parachain/coretime.md)
│
├─ Is the collator claiming AURA slots?
│  ├─ AURA is round-robin — each collator gets predictable slots
│  ├─ Check logs for "Building block." or "Not building block."
│  │  (target: "aura", from slot-based block builder)
│  └─ "Not building block due to insufficient authoring duration." → timing issue
│
├─ Block built but not included on relay chain?
│  ├─ Collation submitted but not backed by validators
│  ├─ Check: "Submitting collation for core." log → collation was sent
│  ├─ If no "Submitting collation" → "Unable to build collation." errors
│  └─ See [parachain/not-producing](../parachain/not-producing.md) for relay-side diagnosis
│
└─ Unincluded segment full? (async backing)
   ├─ Modern collators can build multiple blocks per relay block
   ├─ If unincluded segment is at capacity, collator pauses building
   ├─ Check: parachain blocks advancing on relay chain (inclusion pipeline)
   └─ If stuck → check relay chain validators are backing the parachain
```

## Resolution

### Keystore issues

The keystore holds the validator's session keys. Without them, the node can't sign blocks.

1. **Check keystore directory exists** and contains key files:
   ```bash
   ls <base-path>/chains/<chain>/keystore/
   # Should contain files like:
   #   62616265<hex>  (babe - relay chain block production)
   #   6772616e<hex>  (gran - GRANDPA finality)
   #   61757261<hex>  (aura - parachain block production, for collators)
   ```
2. **Verify keys match on-chain session keys:**
   ```bash
   curl -s -H "Content-Type: application/json" \
     -d '{"id":1,"jsonrpc":"2.0","method":"author_hasSessionKeys","params":["<session_keys_hex>"]}' \
     http://<node>:9944 | jq .
   # Expected: {"result": true}
   # Returns true only if the keystore has private keys for ALL session key types
   ```
3. If keys are missing: **re-insert session keys** (coordinate with the validator operator)

### Resource bottleneck during authoring

Key log to look for: `"⌛️ Discarding proposal for slot <N>; block production took too long"`

- Relay chain: block must be built within the 6s BABE slot window (proposer gets a fraction of this)
- Parachains: minimum block production interval is 500ms (`BLOCK_PRODUCTION_MINIMUM_INTERVAL_MS`); if authoring duration is insufficient, the collator logs `"Not building block due to insufficient authoring duration."` and skips

Actions:
- Check CPU usage during the slot
- Check if state trie is too large (deep storage reads)
- Check for RocksDB compaction competing for IO
- See [high-resource-usage](high-resource-usage.md)

> **Tip:** If the node was compiled in debug mode, you'll see:
> `"👉 Recompile your node in --release mode to mitigate this problem."`
> Always run production nodes with `--release`.

## Escalation

- Collect: node role (`system_nodeRoles`), `--validator` flag presence, key presence (`author_hasSessionKeys`), sync status, logs from missed slots
- For collators: relay chain sync status + parachain sync status + core assignment status
- Escalate to: `[TODO: dev team contact / channel]`
