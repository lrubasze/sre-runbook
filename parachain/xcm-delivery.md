# Runbook: XCM Message Delivery Issues

## Quick check

**1. Identify the direction:**
- **DMP** (Downward Message Passing): relay chain → parachain
- **UMP** (Upward Message Passing): parachain → relay chain
- **XCMP/HRMP** (Horizontal): parachain → parachain (via relay chain channels)

**2. Was the message sent?**
- Check the sending extrinsic succeeded on the source chain

**3. Are both chains producing blocks?**
- XCM delivery requires both source and destination to be producing blocks

## Triage

### Relay → Parachain (DMP)

1. **Message in relay DMP queue?**
   - Check `dmp.downwardMessageQueues(paraId)` on relay chain
   - Not there → message may not have been sent. Check sending extrinsic.
   - Present → parachain not processing DMP. Is parachain producing blocks? If not → see [not-producing](not-producing.md)
   - Parachain producing but DMP not consumed → [DMP queue stalled](#dmp-queue-processing-stalled)

2. **Message delivered but not executed?**
   - XCM execution error on destination → [XCM execution errors](#xcm-execution-errors)

### Parachain → Relay (UMP)

1. **Message in parachain's outbound UMP queue?**
   - UMP messages are delivered when parachain block is included on relay chain
   - Check if parachain blocks are being included

2. **Message delivered but execution failed?**
   - → [XCM execution errors](#xcm-execution-errors)

### Parachain → Parachain (HRMP)

1. **HRMP channel exists?**
   - Check `hrmp.hrmpChannels([senderParaId, recipientParaId])`
   - No channel → must be opened (requires governance or sudo on both sides)

2. **Channel open but message not delivered?**
   - Both parachains must be producing blocks for HRMP to work

3. **Channel capacity full?**
   - HRMP channels have a max capacity (messages + bytes)
   - Messages queue until space is available
   - See [HRMP channel capacity](#hrmp-channel-capacity)

### Message delivered but funds/action not visible

- XCM executed but may have trapped assets
- Check `xcmPallet.assetTraps` or events on destination chain
- See [Trapped assets](#trapped-assets)

## Deep Investigation

### Useful Loki queries

```logql
# XCM-related logs
{instance="<node>"} |~ "xcm|XCM|hrmp|dmp|ump"

# Queue processing
{instance="<node>"} |~ "message.*queue|MessageQueue"
```

### On-chain checks

```
# HRMP channels
hrmp.hrmpChannels([senderParaId, recipientParaId])

# DMP queue
dmp.downwardMessageQueues(paraId)

# Parachain's inbound XCMP queue (on the parachain)
xcmpQueue.inboundXcmpMessages(senderParaId)
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
| `Trap` | Explicit trap — assets held in trap storage | See [Trapped assets](#trapped-assets) |

**How to check:** look for XCM execution events on the **destination** chain:
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
