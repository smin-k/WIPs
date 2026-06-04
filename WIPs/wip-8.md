# WIP-8: Rokis Hardfork Meta

<pre>
  WIP: 8
  Title: Rokis Hardfork Meta
  Status: Draft
  Type: Meta
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-06-03
  License: GNU Lesser General Public License v3.0
</pre>

## Abstract / 개요

이 문서는 Rokis 하드포크의 활성화 파라미터와 포함 WIP 목록을 정의하는 메타 WIP이다. Rokis는 Seoul/ECCPoW 체인을 WIP-6의 VCT 합의 규칙으로 전환하고, WIP-7의 보상 및 트레저리 규칙을 활성화한다.

본 문서는 VCT의 세부 합의 규칙을 다시 정의하지 않는다. VCT의 proposer eligibility, ECVRF ciphersuite, sortition, progressive timeout, mining signature, ECCPoW seed, 블록 검증 규칙은 WIP-6에 정의되어 있다. Rokis 이후의 보상 및 트레저리 분배 규칙은 WIP-7에 정의되어 있다.

## Motivation / 동기

Ethereum의 hardfork meta EIP처럼, WorldLand도 개별 프로토콜 변경 WIP와 특정 네트워크 업그레이드의 활성화 정보를 분리할 필요가 있다. WIP-6과 WIP-7은 각각 합의 규칙과 보상 규칙을 정의하지만, Seoul 네트워크에서 어느 블록에 어떤 변경을 함께 활성화할지는 별도의 메타 문서가 담당하는 것이 명확하다.

이 분리는 두 가지 장점을 가진다. 첫째, 프로토콜 설계와 배포 일정을 독립적으로 검토할 수 있다. 둘째, 클라이언트 구현자가 특정 하드포크에 포함되는 WIP 목록을 한 곳에서 확인할 수 있다.

## Specification / 명세

### Included WIPs

Rokis 하드포크는 다음 WIP를 포함한다.

```text
WIP-6: Verifiable Coin Toss
WIP-7: Rokis Reward and Treasury
```

### Seoul Mainnet Parameters

Rokis는 Seoul 네트워크의 미래 하드포크로 활성화된다.

```text
network: Seoul
chainId: 103
VCTBlock: 10,000,000
pre-VCT phase: block 0..9,999,999, Seoul/ECCPoW compatible
VCT phase: block 10,000,000+, Rokis/WIP-6
S0: 100 WL

fork difficulty policy: inherit parent difficulty
initial sortitionThreshold: 2^256 (p = 1.0, bootstrap all eligible)
P_base (target SortitionBase): 2^253 (12.5%, p* = 1/8)
TimeoutStart: 15 seconds
TimeoutEnd: 60 seconds

VCT block reward: 20 WL
treasury split: 20%
treasury address: 0x4C7dE6771DC602176b25fD4E1ae5550A3eAa06dF
```

하드포크 경계에서 VCT는 기존 ECCPoW difficulty를 초기화하거나 대체하지 않는다. 첫 Rokis 블록은 Seoul 체인의 부모 difficulty를 상속한다. Seoul 체인은 이미 유의미한 ECCPoW difficulty에서 동작하고 있으므로, Rokis 활성화 시 difficulty를 별도 floor로 재설정하지 않는다.

VRF sortition threshold는 전환 시점의 추가 활성성 병목을 피하기 위해 완전히 열린 값 `2^256`에서 시작한다. 이후 deterministic bootstrap controller가 목표값 `P_base = 2^253`까지 낮춘다.

`initialSortitionThreshold`는 전체 uint256 threshold와 1..256 범위의 compact byte-scale 값을 모두 허용할 수 있다. byte-scale 형식에서 `32`는 `2^253`($p = 1/8$), `128`은 `2^255`($p = 1/2$), `256`은 `2^256`($p = 1$)으로 해석된다. Rokis는 `256`을 bootstrap initial threshold로만 사용하며, post-bootstrap 목표값은 `P_base = 2^253`이다.

## Rationale

Rokis 하드포크는 합의 엔진을 VCT로 전환하지만 기존 Seoul 체인의 history와 difficulty를 보존한다. 따라서 하드포크 시점에서 difficulty를 재설정하지 않고 부모 difficulty를 상속하는 것이 가장 보수적인 선택이다.

초기 sortition threshold를 `2^256`으로 두는 이유는 하드포크 직후 VRF gate가 활성성 병목이 되는 것을 피하기 위해서다. 이후 목표값 `2^253`까지 낮추면 모든 계정이 항상 즉시 자격을 얻는 상태에서 벗어나 VCT의 selective proposer admission이 작동한다.

## Backwards Compatibility / 하위 호환성

Rokis는 하드포크이다. VCTBlock 이후 WIP-6과 WIP-7을 구현하지 않은 클라이언트는 Rokis 블록을 거부하거나 state root를 다르게 계산하여 네트워크에서 분리된다.

VCTBlock 이전에는 기존 Seoul/ECCPoW 규칙을 그대로 유지한다. VCT 엔진을 탑재한 클라이언트라도 pre-VCT 블록은 legacy sealHash, legacy ECCPoW seed, legacy reward policy를 사용해야 한다.

## Reference Implementation / 구현

클라이언트는 Seoul chain config에 다음 정보를 포함해야 한다.

```text
VCTBlock: 10,000,000
Vct:
  MinEligibleBalance: 100 WL
  InitialSortitionThreshold: 256
```

구현이 완료된 것으로 간주되려면 다음 테스트 요구 사항을 충족해야 한다.

- [ ] Seoul 기존 DB에서 VCT 엔진 선택이 문제없이 수행되는지 확인
- [ ] 신규 Seoul node에서 VCTBlock 이전 pre-VCT 블록이 기존 ECCPoW와 호환되는지 확인
- [ ] VCTBlock 경계에서 legacy rule과 VCT rule이 정확히 분기되는지 확인

## References / 참고문헌

[1] WIP-6: Verifiable Coin Toss.  
[2] WIP-7: Rokis Reward and Treasury.  
[3] EIP-1716: Hardfork Meta: Petersburg.  

## Copyright / 저작권

Copyright and related rights released under the [GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.html).
