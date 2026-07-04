# WIP-9: Hardfork Meta - Rockies

<pre>
  WIP: 9
  Title: Hardfork Meta - Rockies
  Status: Draft
  Type: Meta
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-06-03
  License: GNU Lesser General Public License v3.0
</pre>

## Simple Summary

This WIP defines the activation settings for the Rockies Upgrade on WorldLand Seoul mainnet.

## Abstract

Rockies activates WIP-6, WIP-7, and WIP-8 together on Seoul mainnet. This hardfork moves Seoul from the existing ECCPoW section to the VCT-ECCPoW section, applies the Rockies reward and treasury rule, and sets the VCT minimum difficulty.

This WIP does not define higher-layer compute features. DID/TPM physical node checks, Tail Certificate, compute marketplace, VCC settlement, attested runtime, AI workload verification, and service-fee logic are out of scope for this hardfork.

## Included WIPs

```text
WIP-6: Verifiable Coin Toss
WIP-7: Rockies Reward and Treasury
WIP-8: VCT Minimum Difficulty
```

## Seoul Mainnet Parameters

| Item | Value |
|------|-------|
| Target network | Seoul mainnet |
| Chain ID | 103 |
| VCTBlock candidate | 10,000,000 |
| Consensus algorithm | VCT-ECCPoW |
| S0 | 100 WL |
| Initial eligibility threshold | 2^256 |
| Target eligibility threshold | 2^253 |
| Target immediate eligibility probability | 1/8 |
| Timeout start | 15 seconds |
| Timeout end | 70 seconds |
| Future time tolerance | 5 seconds |
| VCT minimum difficulty | 65,536 |
| Base block reward | 20 WL |
| Treasury split | 20% |
| Halving interval | 12,614,400 blocks |

## Reference Implementation

The Seoul chain config must include the Rockies activation block and VCT config values:

```go
SeoulChainConfig.VCTBlock = big.NewInt(10_000_000)
SeoulChainConfig.Vct = &VctConfig{
    MinEligibleBalance:          100 WL,
    InitialEligibilityThreshold: big.NewInt(256),
}
```

The public header/API field name should use `eligibilityThreshold`; old temporary naming should not be used in the public spec.

## Backwards Compatibility

This is a hardfork. Nodes that do not apply the Rockies rules at `VCTBlock` will reject Rockies blocks or compute different state roots.

Blocks before `VCTBlock` keep the existing Seoul/ECCPoW rules.

## Copyright

Copyright and related rights waived via CC0.
