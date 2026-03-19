# Node Overview

Background knowledge for SRE team. Read this first to understand what you're operating.

## What is a Polkadot node?

A Polkadot node is a Rust binary that:
1. **Connects to peers** via libp2p (P2P networking layer)
2. **Syncs the chain** — downloads and verifies blocks from peers
3. **Maintains state** — stores blockchain data in a RocksDB database
4. **Runs consensus** — participates in block production (BABE) and finality (GRANDPA)
5. **Executes the runtime** — runs on-chain logic in a WASM sandbox

## Node startup flow

```
Binary starts
  → Load chain spec (network identity, genesis, bootnodes)
  → Open database (RocksDB)
  → Initialize runtime (load WASM from DB or chain spec)
  → Start networking (libp2p)
    → Connect to bootnodes
    → Discover peers via Kademlia DHT
  → Start sync
    → Warp sync (default): download finality proofs → state → fill gaps
    → Full sync: download all blocks from genesis
  → Start consensus subsystems (if validator/collator)
    → BABE (block authoring)
    → GRANDPA (finality voting)
    → Approval voting, backing, dispute resolution (relay chain validators)
  → Start RPC server (if enabled)
  → Node is "ready"
```

## Key components

### Sync modes

| Mode | What it does | When used |
|---|---|---|
| **Warp sync** | Download GRANDPA finality proofs, then state at finalized block, then fill block gaps | Default for new nodes |
| **Full sync** | Download all blocks sequentially from genesis or last known block | `--sync full` flag |
| **Fast sync** | Download blocks without executing them, then state | Less common |

### Database

- **RocksDB** — the storage backend
- **State DB** — stores trie state (account balances, storage items, etc.)
- **Block DB** — stores block headers and bodies
- **Pruning** — by default, old state is pruned (only last N blocks kept). Configurable via `--state-pruning`
  - `archive` — keep all state (high disk usage)
  - `N` — keep last N blocks of state (default: 256)

### Consensus

- **BABE** — block production. Validators take turns producing blocks in slots (6 seconds on Polkadot)
- **GRANDPA** — finality. Validators vote to finalize blocks. Finalized = irreversible
- **Finalized block** should trail best block by a few blocks. Large gap = problem

### Networking

- **libp2p** — the networking stack
- **Bootnodes** — initial peers to connect to (from chain spec)
- **Kademlia DHT** — peer discovery protocol
- Healthy node: 20-50+ peers
- Minimum to function: ~3-5 peers

### Relay chain vs Parachain

```
┌─────────────────────────┐
│     Relay Chain Node     │  ← Polkadot/Kusama validators
│  (consensus, finality)   │
└──────────┬──────────────┘
           │ validates
┌──────────▼──────────────┐
│    Parachain Collator    │  ← Produces parachain blocks
│  (relay client embedded) │  ← Contains an embedded relay chain node
└─────────────────────────┘
```

A parachain collator runs **two sync processes**:
1. Relay chain sync (needs to follow the relay chain)
2. Parachain sync (its own chain)

Both must be healthy for the collator to function.

## Default ports

| Port | Service |
|---|---|
| `30333` | P2P (libp2p) |
| `9944` | RPC (HTTP + WebSocket, unified since recent SDK versions) |
| `9615` | Prometheus metrics |

## Useful CLI flags for debugging

```bash
# Increase log verbosity
--log sync=debug,grandpa=debug,network=debug

# Check node version
polkadot --version

# Validate chain spec
polkadot build-spec --chain <chain> --raw

# Export blocks for analysis
polkadot export-blocks --from <N> --to <M> --chain <chain>
```
