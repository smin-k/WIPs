# WIP-7: Rockies Reward and Treasury

<pre>
  WIP: 7
  Title: Rockies Reward and Treasury
  Status: Draft
  Type: Core
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-06-03
  License: GNU Lesser General Public License v3.0
</pre>

## Simple Summary

This WIP defines the block reward and treasury rules that apply after `VCTBlock`.

Before `VCTBlock`, Seoul keeps the existing ECCPoW reward rules. Starting at `VCTBlock`, the chain applies the Rockies reward policy: 20 WL base block reward, 80% to the miner, and 20% to the protocol treasury.

## Abstract

Rockies changes the reward state transition for the VCT section of WorldLand Seoul mainnet. The consensus rules for VCT-ECCPoW are defined in WIP-6. This WIP only defines reward accounting.

The base block reward starts at 20 WL from `VCTBlock`. The reward halves every 12,614,400 blocks. With a 10-second target block time, this is 1,460 days, or four years. The previous maturity schedule and 4% annual inflation rule are removed from the Rockies reward section.

## Motivation

Reward processing is part of the state transition. All nodes must apply the same reward rule at the same block height. If the fork boundary or treasury split is unclear, nodes can compute different state roots for the same block.

Rockies also separates block reward treasury from later Web3 Cloud or marketplace service-fee rules. Service-fee settlement is out of scope for this WIP.

## Specification

### Fork Boundary

```text
if block.number < VCTBlock:
    apply existing Seoul/ECCPoW reward policy
else:
    apply Rockies reward and treasury policy
```

### Rockies Base Reward

At and after `VCTBlock`, the base block reward is:

```text
RockiesBaseReward = 20 WL
```

The base reward is split as follows:

```text
MinerReward    = 80% of total block reward
TreasuryReward = 20% of total block reward
```

The candidate treasury address is:

```text
0x4C7dE6771DC602176b25fD4E1ae5550A3eAa06dF
```

### Halving Schedule

`VCTBlock` becomes the new halving base point.

```text
RockiesHalvingInterval = 12,614,400 blocks
```

At a 10-second target block time:

```text
12,614,400 blocks * 10 seconds = 126,144,000 seconds = 1,460 days
```

The reward schedule is:

| Period after VCTBlock | Base block reward |
|-----------------------|-------------------|
| 0 intervals | 20 WL |
| 1 interval | 10 WL |
| 2 intervals | 5 WL |
| 3 intervals | 2.5 WL |

The old maturity schedule and 4% annual inflation rule are not applied after Rockies activation.

### Uncle Rewards

Uncle reward handling remains compatible with the existing implementation. The 80/20 split applies to the total reward amount processed for the block according to the implementation rule.

## Backwards Compatibility

This change applies only at and after `VCTBlock`. Blocks before `VCTBlock` keep the existing Seoul/ECCPoW reward processing.

Clients that apply the Rockies reward rule before `VCTBlock`, or fail to apply it after `VCTBlock`, will compute a different state root.

## Security Considerations

The treasury address, fork block, base reward, and halving interval are consensus-critical values. They must be fixed in the final hardfork configuration before mainnet activation.

## Copyright

Copyright and related rights waived via CC0.
