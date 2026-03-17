# Runbook: XCM Message Delivery Issues

## Symptoms

- Cross-chain transfer stuck (tokens sent but not received on destination)
- XCM messages queued but not processed
- `xcmpQueue` or `dmpQueue` growing without being consumed

## Quick Health Check

**Identify the direction:**
- **DMP** (Downward Message Passing): relay chain → parachain
- **UMP** (Upward Message Passing): parachain → relay chain
- **XCMP/HRMP** (Horizontal): parachain → parachain (via relay chain channels)

**On-chain checks:**
```
# Check HRMP channels exist between parachains
hrmp.hrmpChannels([senderParaId, recipientParaId])

# Check DMP queue for a parachain
dmp.downwardMessageQueues(paraId)

# Check parachain's inbound XCMP queue
# (on the parachain itself)
xcmpQueue.inboundXcmpMessages(senderParaId)
```

**Loki:**
```logql
# XCM-related logs
{instance="<node>"} |~ "xcm|XCM|hrmp|dmp|ump"

# Queue processing
{instance="<node>"} |~ "message.*queue|MessageQueue"
```

## Decision Tree

```
XCM message not delivered
│
├─ Which direction?
│  ├─ Relay → Parachain (DMP)
│  │  ├─ Message in relay DMP queue?
│  │  │  ├─ Yes → parachain not processing DMP
│  │  │  │  └─ Is parachain producing blocks?
│  │  │  │     └─ No → see [not-producing](not-producing.md)
│  │  │  │     └─ Yes → DMP queue processing stalled
│  │  │  │        → Resolution: DMP queue
│  │  │  └─ No → message may not have been sent
│  │  │     └─ Check sending extrinsic succeeded on relay chain
│  │  │
│  │  └─ Message delivered but not executed?
│  │     └─ XCM execution error on destination
│  │        → Resolution: XCM execution errors
│  │
│  ├─ Parachain → Relay (UMP)
│  │  ├─ Message in parachain's outbound UMP queue?
│  │  │  └─ Check if parachain blocks are being included on relay
│  │  │     UMP messages are delivered when parachain block is included
│  │  │
│  │  └─ Message delivered to relay but execution failed?
│  │     └─ → Resolution: XCM execution errors
│  │
│  └─ Parachain → Parachain (HRMP)
│     ├─ HRMP channel exists?
│     │  └─ No → channel must be opened (requires governance or sudo on both sides)
│     │
│     ├─ Channel open but message not delivered?
│     │  └─ Check if both parachains are producing blocks
│     │     HRMP messages require both chains to be active
│     │
│     └─ Channel capacity full?
│        └─ HRMP channels have a max capacity (messages + bytes)
│           Messages queue until space is available
│           → Resolution: HRMP channel capacity
│
└─ Message delivered but funds/action not visible?
   └─ XCM executed but may have trapped assets
      Check: xcmPallet.assetTraps or events on destination chain
      → Resolution: Trapped assets
```

## Resolution

### DMP queue processing stalled

If the parachain is producing blocks but not processing DMP messages:
1. Check `MessageQueue` pallet on the parachain — is it processing?
2. Check for weight exhaustion — DMP processing is limited per block
3. Large message backlogs may take many blocks to clear
4. If stuck: parachain runtime issue — escalate

### XCM execution errors

Common XCM execution failures:
| Error | Meaning | Action |
|---|---|---|
| `TooExpensive` | Insufficient fee payment in XCM | Check fee configuration, may need more fee assets |
| `AssetNotFound` | Destination doesn't recognize the asset | Check asset registration on destination chain |
| `Barrier` | XCM message rejected by destination's barrier | Check destination's XCM barrier configuration |
| `WeightNotComputable` | Can't determine execution weight | XCM program issue — escalate to dev team |
| `Trap` | Explicit trap — assets held in trap storage | See Trapped assets below |

**How to check:** Look for XCM execution events on the **destination** chain:
- `xcmPallet.Attempted` with `Incomplete` or `Error` outcome
- `messageQueue.ProcessingFailed` events

### HRMP channel capacity

HRMP channels have limits configured on the relay chain:
- `hrmpChannelMaxCapacity` — max messages in queue
- `hrmpChannelMaxTotalSize` — max bytes in queue

If a channel is full:
1. Destination parachain must produce blocks to consume messages
2. If destination is stalled → fix destination first
3. If consistently hitting limits: request capacity increase via governance

### Trapped assets

If XCM execution partially fails, assets may be "trapped" on the destination:
1. Check `xcmPallet.AssetTrap` events on destination
2. Trapped assets can be claimed by the owner
3. This is an application/governance level action, not SRE

## Escalation

- Collect: source chain, destination chain, XCM message hash/extrinsic, direction (DMP/UMP/HRMP)
- Check: channel existence, queue status, execution events on destination
- Escalate to: `[TODO: XCM team / parachain team]`
