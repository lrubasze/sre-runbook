# Runbook: Block Production Issues

> **Triggered by alerts:** `NetworkBlockProductionStalled`, `NodesStoppedValidating`
>
> See also: [monitoring/alert-reference](../monitoring/alert-reference.md)

## Quick check

**1. Network-wide or just this node?**
- Check `substrate_block_height{status="best"}` across multiple nodes
- ALL stopped? **Escalate immediately.** Single node? Continue.

**2. Is the node synced?**
- `substrate_block_height{status="best"}` matches chain head?
- No → see [not-syncing](not-syncing.md)

**3. Is it an authority?**
```bash
curl -s -H "Content-Type: application/json" \
  -d '{"id":1,"jsonrpc":"2.0","method":"system_nodeRoles","params":[]}' \
  http://<node>:9944 | jq .
```
- `["Full"]` → not started with `--validator` flag. Restart with `--validator`.
- `["Authority"]` → good. Continue to [Triage](#triage).

## Triage

> Relay chain validators use **BABE** (VRF slot assignment).
> Parachain collators use **AURA** (round-robin).
> If this is a collator, skip to [Collator Triage](#collator-triage).

### Validator triage

Node is synced + Authority + still not producing:

1. **Session keys present?**
   - `author_hasSessionKeys` RPC → must return `true`
   - Missing → [Keystore Resolution](#keystore-issues)

2. **In active validator set?**
   - Check `session.validators()` on-chain
   - Not in set → not SRE, notify team

3. **Check authoring logs:**
   - `"Starting authorship at slot: <N>"` → slot was claimed, block build attempted
     - then `"🔖 Pre-sealed block for proposal at <N>"` → working, block was built
     - then `"Proposing failed: <err>"` → [Resource Bottleneck](#resource-bottleneck-during-authoring)
     - then `"⌛️ Discarding proposal for slot <N>; block production took too long"` → [Resource Bottleneck](#resource-bottleneck-during-authoring)
     - no follow-up log → hung, check CPU/IO
   - `"Claimed slot <N>"` but no `"Starting authorship"` → inherent data or proposer creation failed. Look for `"Unable to author block"` or `"Unable to create inherent data"`
   - No authoring logs at all? → normal for BABE (VRF only assigns some slots per validator). Only a problem if NEVER gets a slot across multiple epochs.

4. **None of the above** → [Escalation](#escalation)

### Collator triage

Collator not producing parachain blocks:

1. **Embedded relay chain synced?**
   - Compare relay best block vs network head
   - Not synced → wait, or check relay chain peer connectivity

2. **Core assigned?**
   - Bulk or on-demand coretime?
   - No core → [parachain/coretime](../parachain/coretime.md)

3. **Claiming AURA slots?**
   - Check logs: `"Building block."` / `"Not building block."`
   - `"Not building block due to insufficient authoring duration."` → timing issue, see [Resource Bottleneck](#resource-bottleneck-during-authoring)

4. **Block built but not included on relay chain?**
   - `"Submitting collation for core."` present → collation was sent
   - No `"Submitting collation"` → look for `"Unable to build collation."` errors
   - Verify relay-side: `ParaInclusion::CandidateBacked` events for your para_id
     - No backed candidates → validators aren't backing → [PVF Bottleneck](#pvf-validation-bottleneck-validator-side)
   - See also [parachain/not-producing](../parachain/not-producing.md)

5. **Unincluded segment full?** (async backing)
   - Collator pauses if unincluded segment at capacity
   - Check if parachain blocks are advancing on relay chain

6. **None of the above** → [Escalation](#escalation)

## Deep Investigation

Use this section when quick triage didn't resolve the issue.

> **Principle:** Always verify both sides — collator-side (block built) AND relay-side
> (candidate backed/included). A collator reporting success does not mean the relay chain accepted it.

### Metrics check (always available, independent of log level)

- [ ] `substrate_block_height{status="best"}` — advancing?
- [ ] `substrate_block_height{status="finalized"}` — finality advancing? (check over `[TODO: agree on window]`)
- [ ] Finality lag: `best - finalized` within `[TODO: agree on threshold]` blocks
- [ ] `node_roles` = 4 (validator)
- [ ] `substrate_connected_peers` ≥ `[TODO: agree on minimum]`

### Relay-side verification (for parachains)

- [ ] `ParaInclusion::CandidateBacked` events present for your para_id
- [ ] Backed candidates at expected rate: `[TODO: agree on rate]` per relay block window
- [ ] **Session boundary awareness:** don't measure throughput during the first session after startup or registration

### PVF health (on relay chain validators)

If collator builds blocks but they're never included, check the validator side:

- [ ] `polkadot_pvf_execution_time` p99 well below 2s (backing timeout)
- [ ] `polkadot_pvf_execution_queued_time` not growing (queue pressure)
- [ ] `polkadot_pvf_preparation_time` — check after runtime upgrades
- [ ] `polkadot_pvf_worker_retired` not spiking (workers crashing)

### Critical error signals (must never appear)

- [ ] `"set_validation_data inherent needs to be present in every block"` — consensus hook broken, collator cannot build valid blocks

### Useful Loki queries

```logql
# Authoring-related logs (target: "babe" for validators, "aura" for collators)
{instance="<node>"} |~ "Starting authorship|Claimed slot|Pre-sealed|Proposing failed|Skipping slot"

# Errors during block production
{instance="<node>"} |~ "slots|babe|aura" |~ "WARN|ERROR"
```

### Useful Prometheus queries

```promql
# Node-side
substrate_block_height{status="best"}
substrate_block_height{status="finalized"}
substrate_block_height{status="best"} - substrate_block_height{status="finalized"}

# PVF (on relay chain validators, not collators)
# Execution time — hard 2s timeout for backing, 12s for approval
histogram_quantile(0.99, rate(polkadot_pvf_execution_time_bucket[5m]))
# Preparation time — WASM compilation, one-time per code version
histogram_quantile(0.99, rate(polkadot_pvf_preparation_time_bucket[5m]))
# Queue pressure
histogram_quantile(0.99, rate(polkadot_pvf_execution_queued_time_bucket[5m]))
# Worker pool health
polkadot_pvf_worker_spawned
polkadot_pvf_worker_retired
```

## Resolution

### Keystore issues

The keystore holds session keys. Without them, the node can't sign blocks.

1. **Check keystore directory:**
   ```bash
   ls <base-path>/chains/<chain>/keystore/
   # Expected files:
   #   62616265<hex>  (babe - relay chain block production)
   #   6772616e<hex>  (gran - GRANDPA finality)
   #   61757261<hex>  (aura - parachain block production, for collators)
   ```
2. **Verify keys match on-chain:**
   ```bash
   curl -s -H "Content-Type: application/json" \
     -d '{"id":1,"jsonrpc":"2.0","method":"author_hasSessionKeys","params":["<session_keys_hex>"]}' \
     http://<node>:9944 | jq .
   # Expected: {"result": true}
   # Returns true only if keystore has private keys for ALL session key types
   ```
3. Keys missing → **re-insert session keys** (coordinate with validator operator)

### Resource bottleneck during authoring

Key log: `"⌛️ Discarding proposal for slot <N>; block production took too long"`

- Relay chain: must build within the 6s BABE slot window
- Parachains: 500ms minimum block production interval; if insufficient, collator logs `"Not building block due to insufficient authoring duration."`

Actions:
- Check CPU usage during the slot
- Check state trie size (deep storage reads)
- Check RocksDB compaction competing for IO
- See [high-resource-usage](high-resource-usage.md)

> **Tip:** If compiled in debug mode, you'll see:
> `"👉 Recompile your node in --release mode to mitigate this problem."`

### PVF validation bottleneck (validator-side)

If a collator builds blocks but `ParaInclusion::CandidateBacked` events are missing,
the problem may be on relay chain validators.

**PVF** (Parachain Validation Function) has two phases:
1. **Preparation** — WASM compilation. One-time per code version, cached on disk. Slow after runtime upgrades.
2. **Execution** — `validate_block` call. Runs on every candidate. Hard 2s timeout for backing (`DEFAULT_BACKING_EXECUTION_TIMEOUT`). Exceeding this = candidate dropped.

**Diagnosis (on relay chain validators):**
```promql
# Executions timing out?
histogram_quantile(0.99, rate(polkadot_pvf_execution_time_bucket[5m]))

# Queue backing up?
histogram_quantile(0.99, rate(polkadot_pvf_execution_queued_time_bucket[5m]))

# Workers dying?
rate(polkadot_pvf_worker_retired[5m])
```

Actions:
- Check validator CPU/memory — PVF execution is CPU-bound
- After runtime upgrade: monitor `polkadot_pvf_preparation_time` for new WASM compilation
- Workers crashing: check validator logs for PVF worker panics

## Escalation

- Collect: node role (`system_nodeRoles`), `--validator` flag, key presence (`author_hasSessionKeys`), sync status, authoring logs
- For collators: relay chain sync + parachain sync + core assignment + `ParaInclusion::CandidateBacked` status
- Escalate to: `[TODO: dev team contact / channel]`
