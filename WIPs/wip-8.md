# WIP-8: VCT Minimum Difficulty

<pre>
  WIP: 8
  Title: VCT Minimum Difficulty
  Status: Draft
  Type: Core
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-06-28
  License: GNU Lesser General Public License v3.0
</pre>

## Simple Summary

This WIP sets the minimum ECCPoW difficulty used in the VCT section to 65,536.

## Abstract

VCT keeps the WIP-5 difficulty control rule. WIP-5 lowered the difficulty sensitivity to `1/1024` so that long block delays do not push difficulty down too sharply.

However, a `1/1024` sensitivity can stop changing difficulty when the difficulty is too small. With the old Seoul minimum difficulty of `1023`, integer division gives:

```text
floor(1023 / 1024) = 0
```

When this happens, the difficulty adjustment term becomes zero. To keep the WIP-5 rule active in the VCT section, this WIP sets:

```text
VCTMinimumDifficulty = 65,536
```

## Motivation

The old Seoul difficulty rule used high sensitivity. If block generation time became very large, difficulty could fall sharply and then clamp to the minimum difficulty.

WIP-5 fixed that sharp drop by using `1/1024` sensitivity. But with a very small minimum difficulty, the adjustment amount can become zero. VCT should not run with a difficulty range where the controller cannot move.

## Specification

After `VCTBlock`, the client first computes raw difficulty with the WIP-5 rule:

```text
D_raw =
  D_parent
  + floor(D_parent / 1024)
    * max((1 or 2) - floor(BGT / 7), -99)
```

Then the VCT section applies the minimum difficulty:

```text
D_next = max(D_raw, VCTMinimumDifficulty)
VCTMinimumDifficulty = 65,536
```

`65,536 / 1024 = 64`, so the WIP-5 adjustment term remains nonzero at the minimum.

## Backwards Compatibility

This rule applies only at and after `VCTBlock`. Blocks before `VCTBlock` keep the existing Seoul/ECCPoW difficulty rule.

## Security Considerations

This WIP does not change the WIP-5 controller. It only sets a VCT-section lower bound so the controller can keep adjusting difficulty near the minimum range.

## Copyright

Copyright and related rights waived via CC0.
