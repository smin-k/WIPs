# WIP-7: Rokis Reward and Treasury

<pre>
  WIP: 7
  Title: Rokis Reward and Treasury
  Status: Draft
  Type: Core
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-06-03
  License: GNU Lesser General Public License v3.0
</pre>

## Abstract / 개요

Rokis 하드포크 이후 WorldLand의 블록 보상 및 프로토콜 트레저리 분배 규칙을 정의한다. VCT 합의의 proposer eligibility, VRF sortition, ECCPoW seed, 블록 검증 규칙은 WIP-6에서 정의하며, VCTBlock 이후 적용되는 보상 상태 전이는 별도 규칙으로 다룬다.

Rokis 이전의 pre-VCT 구간에서는 기존 Seoul/ECCPoW 보상 정책을 그대로 유지한다. Rokis 이후의 VCT 구간에서는 블록 기본 보상을 20 WL로 설정하고, 블록 생성자 보상과 uncle 보상을 합산한 총 보상 중 20%를 프로토콜 트레저리 주소로 지급하며, 나머지 80%를 블록 생성자에게 지급한다.

## Motivation / 동기

VCT는 기존 ECCPoW 채굴 규칙에 VRF 기반 사전 자격 조건과 계정 바인딩 mining signature를 추가한다. 이 변경은 채굴 참여 구조와 보안 모델을 바꾸므로, Rokis 이후의 보상 정책도 명시적인 합의 규칙으로 고정되어야 한다.

보상 정책은 state transition의 일부이다. 따라서 하드포크 경계에서 어떤 블록에 레거시 보상을 적용하고, 어떤 블록에 Rokis 보상을 적용하는지가 모든 클라이언트에서 동일해야 한다. 이 규칙이 불명확하면 동일한 블록을 검증하더라도 state root가 달라져 네트워크가 분리될 수 있다.

## Specification / 명세

### Fork Boundary

`VCTBlock` 이전의 모든 블록은 기존 Seoul/ECCPoW 보상 정책을 사용한다. `VCTBlock` 이상인 모든 블록은 Rokis 보상 정책을 사용한다.

```text
if block.number < VCTBlock:
    apply legacy Seoul/ECCPoW reward policy
else:
    apply Rokis reward and treasury policy
```

### Pre-VCT Reward Policy

VCTBlock 이전의 Seoul 구간에서는 기존 ECCPoW 보상 정책을 그대로 사용한다. 즉 pre-VCT 블록은 레거시 WorldLand 보상 스케줄에 따라 기본 보상 4 WL 및 기존 halving/maturity 규칙을 적용하며, 트레저리 분배는 수행하지 않는다.

이 구간은 기존 ECCPoW 노드와 동일한 state transition을 유지해야 하므로 보상 공식도 레거시와 일치해야 한다.

### Rokis Reward Policy

VCTBlock 이후 Rokis/VCT 구간에서는 블록 기본 보상을 20 WL로 설정한다.

```text
RokisBlockReward = 20 WL
TreasuryShare    = 20%
MinerShare       = 80%
```

블록 생성자 보상과 uncle 보상을 합산한 총 보상 중 20%는 프로토콜 트레저리 주소로 지급하고, 나머지 80%는 블록 생성자에게 지급한다.

### Treasury Address

Rokis v1.0의 프로토콜 트레저리 주소는 다음과 같다.

```text
0x4C7dE6771DC602176b25fD4E1ae5550A3eAa06dF
```

트레저리 주소는 합의 규칙의 일부이다. 모든 클라이언트는 동일한 주소, 동일한 분배 비율, 동일한 VCTBlock 경계를 사용해야 한다.

## Rationale

보상 규칙을 WIP-6에서 분리하는 이유는 VCT 합의의 기술적 검증 규칙과 경제적 상태 전이 규칙을 독립적으로 검토하기 위해서다. VRF sortition, S0 잔액 자격, mining signature는 블록의 유효성을 결정하는 합의 규칙이고, 보상과 트레저리 분배는 유효 블록이 state에 미치는 경제적 효과를 결정한다.

Rokis 이후 보상을 20 WL로 설정하는 것은 VCT 활성화 이후의 채굴 참여 유인을 명확히 하기 위한 정책 선택이다. 트레저리 20% 분배는 프로토콜 운영, 연구 개발, 생태계 지원을 위한 장기 재원을 확보하기 위한 장치이다.

pre-VCT 구간에서 레거시 보상을 유지하는 것은 하드포크 이전 블록의 state root 호환성을 보장하기 위해 필수이다. VCT 엔진을 탑재한 클라이언트라도 VCTBlock 이전에는 기존 ECCPoW 클라이언트와 동일한 보상 계산을 수행해야 한다.

## Backwards Compatibility / 하위 호환성

Rokis 보상 규칙은 하드포크를 통해 활성화되며, VCTBlock 이후의 state transition을 변경한다. 이 규칙을 구현하지 않은 클라이언트는 VCTBlock 이후 블록의 보상 상태 전이를 다르게 계산할 수 있으므로 Rokis 체인과 호환되지 않는다.

VCTBlock 이전 블록은 기존 Seoul/ECCPoW 보상 정책을 그대로 사용하므로 하위 호환성을 유지한다.

## Reference Implementation / 구현

참조 구현은 WorldLand 클라이언트의 block reward accumulation 경로에서 다음 분기를 적용해야 한다.

```text
if !IsVCT(block.number):
    accumulateRewardsLegacy(...)
else:
    accumulateRewardsRokis(...)
```

구현은 VCTBlock 경계에서 레거시 보상과 Rokis 보상을 명확히 분기해야 하며, pre-VCT 블록에는 트레저리 분배를 적용하지 않아야 한다.

## References / 참고문헌

[1] WIP-6: Verifiable Coin Toss.
[2] WIP-8: Rokis Hardfork Meta.

## Copyright / 저작권

Copyright and related rights released under the [GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.html).
