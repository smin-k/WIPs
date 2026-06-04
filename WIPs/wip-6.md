# WIP-6: Verifiable Coin Toss

<pre>
  WIP: 6
  Title: Verifiable Coin Toss
  Status: Draft
  Type: Core
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-05-07
  License: GNU Lesser General Public License v3.0
</pre>

## 목차
* [Abstract / 개요](#abstract--개요)
* [Motivation / 동기](#motivation--동기)
* [Scope / 범위](#scope--범위)
* [Specification / 명세](#specification--명세)
  - [참여 자격 조건](#참여-자격-조건)
  - [ECVRF 구조](#ecvrf-구조)
  - [VRF 키 쌍 사용 방식](#vrf-키-쌍-사용-방식)
  - [Verifiable Coin Toss 함수](#verifiable-coin-toss-함수)
  - [점진적 타임아웃](#점진적-타임아웃)
  - [논스별 채굴 서명과 sealHash](#논스별-채굴-서명과-sealhash)
  - [블록 생성 및 검증](#블록-생성-및-검증)
  - [p 조정](#p-조정)
* [Rationale](#rationale)
* [Security Analysis / 보안 분석](#security-analysis--보안-분석)
  - [점진적 타임아웃과 활성성](#점진적-타임아웃과-활성성)
* [Backwards Compatibility / 하위 호환성](#backwards-compatibility--하위-호환성)
* [Reference Implementation / 구현](#reference-implementation--구현)
* [References / 참고문헌](#references--참고문헌)
* [Copyright / 저작권](#copyright--저작권)

## Abstract / 개요

이 문서는 WorldLand 블록체인을 위한 합의 메커니즘인 Verifiable Coin Toss (VCT) 프로토콜을 설명한다. VCT는 잔액 임계값 기반 채굴 자격 조건, secp256k1 기반 VRF 선발, 계정 바인딩 ECCPoW, Nakamoto 방식의 최대 유효 작업량 포크 선택을 결합한다.

각 블록 높이에서, 부모 상태 잔액이 $S_0$ 이상인 계정은 자신의 계정 키를 사용하여 부모 블록 해시와 목표 높이에 대해 secp256k1 기반 VRF를 평가할 수 있다. VRF 출력이 네트워크 자격 임계값 $p$ 미만이면 해당 계정은 ECCPoW를 시도할 자격을 얻는다. 생성된 블록에는 계정 공개 키, VRF 출력, VRF 증명, ECCPoW nonce, 그리고 해당 nonce에 대한 논스별 채굴 서명 $\sigma_\nu$가 포함되어야 한다. $\sigma_\nu$는 ECCPoW 탐색 중 각 nonce 시도마다 계정 개인 키로 생성되며, ECCPoW hash-vector seed에 포함된다. 따라서 공개된 VRF 자격 증명만으로는 제3자가 독립적으로 유효한 ECCPoW 입력을 생성할 수 없다. 단, 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하는 경우 위탁 채굴 자체를 암호학적으로 완전히 차단하지는 못한다.

VCT는 검증자 레지스트리나 지분 가중 리더 선출을 도입하지 않는다. 단일 계정에 $S_0$ 이상을 보유해도 해당 계정의 자격 확률은 증가하지 않는다. 여러 자격 시도를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

## Motivation / 동기

PoW(Proof-of-Work) 시스템은 탁월한 검열 저항성을 제공하지만, 모든 노드가 매 블록 경쟁하기 때문에 산업적 규모의 에너지를 소비한다. PoS(Proof-of-Stake) 시스템은 물리적 비용을 자본 가중 투표로 대체하는 대가로 극적인 에너지 절감을 달성한다.

VCT 프로토콜은 다른 위치를 점한다. 스테이킹 컨트랙트나 검증자 레지스트리를 도입하지 않고, 부모 상태의 계정 잔액만을 자격 기준으로 사용한다. VRF 자격 임계값은 각 블록에서 ECCPoW를 수행할 수 있는 계정 수를 제어하며, 최종 블록 경쟁은 여전히 자격을 갖춘 계정들 간의 ECCPoW 연산 경쟁으로 결정된다. 기본 VRF 자격 임계값은 합의 파라미터로 고정하고, 평균 블록 생성 시간 조정은 ECCPoW difficulty가 담당한다.

**초기 네트워크 보안과 잔액 기반 자격 조건.** VRF 기반 블록 선택 체계에서는 계정 수가 직접적인 보안 변수가 된다. 블록당 기대 즉시 자격 계정 수는 $(P_{\mathrm{base}}/256) \cdot N$이므로, 참여 장벽이 없다면 공격자는 계정을 대량 생성하는 것만으로 자격 획득 확률을 임의로 높일 수 있다. 계정 생성 비용이 근본적으로 낮은 Ethereum 계정 모델에서 이는 Sybil 공격에 구조적으로 취약한 설계가 된다 [7, 8]. 특히 초기 네트워크 단계에서는 정직한 참여자 수가 적어 공격자의 상대적 계정 비율이 높아지기 쉽다.

잔액 임계값 $S_0$는 이 취약점을 완화하기 위한 ECCPoW 참여 사전 자격(prerequisite eligibility) 설계다. 각 자격 계정은 $S_0$ 이상의 잔액을 보유해야 하므로, 공격자가 $N_A$개의 자격 계정을 확보하려면 최소 $N_A \times S_0$의 자본을 소비해야 한다. 이는 PoS의 지분 가중 리더 선출과 구분된다. PoS에서는 지분 크기가 선출 확률에 직접 비례하지만, VCT에서는 임계값을 충족한 모든 계정이 동일한 VRF 자격 기회를 받는다. $S_0$는 지분에 비례한 영향력이 아니라 ECCPoW 참여를 위한 최소 진입 비용으로 설계된다.

## Scope / 범위

WIP-6은 balance-gated VRF 자격 조건, 계정 바인딩 ECCPoW 입력, VCTBlock 이후의 블록 검증 규칙을 정의한다. 다음 항목은 이 문서의 범위에 포함하지 않는다.

- `P_base`의 timeout 빈도 기반 동적 조정 또는 ECCPoW difficulty와 결합된 joint controller.
- 장기 lock-up, bonded staking, unbonding period, stake-duration 조건, 최근 여러 블록의 최소/평균 잔액 조건.
- VDF, 외부 randomness beacon, commit-reveal 등 부모 해시 그라인딩을 추가로 완화하는 randomness layer.
- VCT 이후의 블록 보상, uncle 보상, 트레저리 분배, 발행량 정책.

## Specification / 명세

### 참여 자격 조건

$a_i$를 WorldLand 계정 주소, $PK_i$를 그 secp256k1 공개 키라 하면 $a_i = \mathrm{Address}(PK_i)$이다. 계정은 다음 조건을 만족할 때 블록 높이 $h$에 대해 채굴 자격을 갖춘다:

$$\mathrm{Balance}_{h-\ell}(a_i) \geq S_0$$

여기서 $S_0$는 프로토콜 파라미터로 정해진 잔액 임계값이고, $\ell \geq 1$은 잔액 참조 블록의 룩백 깊이(lookback depth)이다.

**기존 설계.** 자격 조건을 $\mathrm{Balance}_{h-1}(a_i) \geq S_0$으로 고정하면 부모 블록 상태를 직접 참조하므로 구현이 가장 단순하다. 그러나 $\ell$을 명시하지 않으면 향후 보안 요구에 따라 조정할 유연성이 없다.

**설계 옵션.** 잔액 기반 자격은 별도 등록 트랜잭션 없이 표준 계정 상태에서 직접 확인할 수 있다는 장점이 있다. staking 컨트랙트나 검증자 레지스트리 방식과 달리 온체인 등록 절차나 lock-up 메커니즘이 필요하지 않다. $\ell > 1$로 설정하면 현재 블록 생성자가 상태 조작을 통해 자격 계정 집합을 블록 경계에서 조작하기 어렵게 만들 수 있으나, 구현 복잡성이 증가한다.

**WIP-6 선택.** $\ell = 1$을 기본값으로 채택한다. 즉, 블록 높이 $h$에 대한 자격은 부모 블록($h - 1$) 상태 잔액으로 판단한다. 별도 레지스트리나 등록 트랜잭션 없이 블록 검증 시 부모 상태에서 직접 확인된다.

**구현 파라미터.** 클라이언트 구현은 하드포크 설정에 포함된 `VctConfig.MinEligibleBalance`를 $S_0$ 값으로 사용한다. 모든 검증 노드는 동일한 하드포크 설정에서 동일한 $S_0$ 값을 사용해야 한다.

**이유.** WIP-6의 목표는 PoS 스타일 자격 조건, VRF 기반 코인 토스, ECCPoW 채굴을 하나의 실행 가능한 합의 파이프라인으로 통합하여 검증하는 것이다. 따라서 이 문서는 스테이킹 컨트랙트나 시스템 컨트랙트를 도입하지 않으며, 자격 확인은 계정 잔액만으로 수행한다.

**한계.** $\ell = 1$은 장기 lock-up이나 bonded staking을 의미하지 않는다. 계정은 부모 상태에서 $S_0$ 이상 잔액을 보유하기만 하면 다음 블록의 자격 조건을 만족하므로, 단기 차입이나 일시적 잔액 이동을 통해 자격을 얻는 전략은 가능하다. 이 문서는 이를 명시적인 설계 trade-off로 받아들인다. $S_0$는 지분 지속 기간을 강제하는 장치가 아니라 Sybil 계정 생성과 ECCPoW 참여에 필요한 최소 유동성 비용을 부과하는 장치다.

각 잔액 조건을 충족한 계정은 블록당 하나의 VRF 자격 기회를 받는다. 동일 계정에 $S_0$ 이상을 보유해도 자격 확률이 증가하지 않는다. 여러 자격 기회를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

### ECVRF 구조

VCT는 계정별 블록 자격 증명을 위해 secp256k1 기반 ECVRF 구성을 사용한다. ECVRF는 ECDSA 서명과 별개의 암호 구조이다. ECDSA는 논스별 채굴 서명에만 사용되고, ECVRF는 특정 블록 높이에서 계정이 ECCPoW 참여 자격을 얻었는지를 증명하는 데 사용된다.

WorldLand 클라이언트의 VCT 구현은 Aergo Foundation copyright가 포함된 secp256k1 VRF 구현 [10]을 기반으로 한다. 다만 외부 구현 자체가 합의 규칙이 되는 것은 아니며, 클라이언트 간 상호운용성을 위해 hash-to-curve, point encoding, scalar encoding, challenge generation, nonce derivation, public-key validation, proof-to-hash 절차는 아래 바이트 수준 정의와 적합성 벡터로 고정한다.

**WorldLand secp256k1 ECVRF ciphersuite.** VCT에서 사용하는 VRF ciphersuite는 다음 바이트 수준 규칙으로 고정된다. 아래 절차와 적합성 벡터는 클라이언트 간 동일한 VRF proof와 output을 보장하기 위한 규범적 정의이다.

- **Curve:** secp256k1, compressed point encoding 33 bytes. 공개키와 모든 curve point는 `0x02` 또는 `0x03` prefix를 갖는 compressed SEC1 형식으로 인코딩한다.
- **Suite byte:** `0xFE`.
- **Hash function:** SHA-256.
- **Scalar encoding:** 32-byte big-endian secp256k1 scalar. Challenge `c`는 16 bytes이며, 검증 시 16 zero bytes를 앞에 붙인 32-byte scalar로 해석한다.
- **VRF input message:** `VCT_VRF || chainId || parentHash || blockNumber`, where `VCT_VRF` is 7 ASCII bytes, `chainId` is 32-byte big-endian, `parentHash` is 32 bytes, and `blockNumber` is 8-byte big-endian.
- **Hash-to-curve:** try-and-increment. For counter `ctr` starting at `0x00`, compute `digest = SHA256(0xFE || 0x01 || PK_compressed || alpha || ctr)`. Form a compressed point candidate as `0x02 || digest`; the first candidate that parses as a non-infinity secp256k1 point is `H`.
- **Nonce derivation:** deterministic RFC6979 nonce generation with private scalar `sk` and message `SHA256(H_compressed)`. Retry the RFC6979 counter until the nonce is a nonzero scalar smaller than the curve order.
- **Challenge generation:** let `Gamma = sk * H`, `kB = k * G`, and `kH = k * H`. Compute `c = SHA256(0xFE || 0x02 || H_compressed || Gamma_compressed || kB_compressed || kH_compressed)[0:16]`.
- **Proof scalar:** `s = c * sk + k mod n`, encoded as 32-byte big-endian scalar.
- **Proof encoding:** `pi = Gamma_compressed || c || s`, with lengths `33 || 16 || 32 = 81` bytes.
- **Proof-to-hash:** `beta = SHA256(0xFE || 0x03 || Gamma_compressed)`, 32 bytes.
- **Verification:** decode `pi` into `(Gamma, c, s)`, recompute `H`, compute `U = s*G - c*PK` and `V = s*H - c*Gamma`, recompute `c' = SHA256(0xFE || 0x02 || H || Gamma || U || V)[0:16]`, and accept iff `c' = c`. The VRF output is then `beta`.

**Ciphersuite conformance vector.** WIP-6을 구현하는 클라이언트는 다음 secret key와 VRF input에 대해 동일한 proof와 VRF output을 재현해야 한다.

```text
secret key:
0000000000000000000000000000000000000000000000000000000000000001

compressed public key:
0279be667ef9dcbbac55a06295ce870b07029bfcdb2dce28d959f2815b16f81798

VRF input alpha:
5643545f565246000000000000000000000000000000000000000000000000000000000000289f11111111111111111111111111111111111111111111111111111111111111110000000000000064

proof pi:
02c7026636565add26a1cf2ec857f77c407609ba8893a0dfae5218f87cea06d20e9139789ff96b5ff2996092b188a4605cb9043c9c1266c8447a7503ee9a1b3fe38c284f3ffd5dc2556294f6db60b6281a

VRF output beta:
3fbb5d74ce6104b219c7989ce71ce65957e573fa492e703e1dbedd1c00565164
```

이 문서에서 $\pi_i$는 위 ciphersuite가 생성하는 81-byte proof object를 의미한다. VCT는 deterministic ECDSA signature를 VRF proof로 재해석하지 않는다.

### VRF 키 쌍 사용 방식

WorldLand 계정은 secp256k1 개인 키 $sk_i$와 공개 키 $PK_i$를 보유하며, 계정 주소는 다음과 같이 유도된다:

$$a_i = \mathrm{Address}(PK_i) = \mathrm{Keccak256}(PK_i)[12{:}]$$

VCT는 이 계정 키 쌍을 ECVRF 입력 키로 재사용한다.

**기존 설계.** 별도의 VRF 키 쌍을 생성하여 온체인에 등록하거나, 합의 레이어 전용 키를 계정 키와 분리하는 방식이 일반적이다. 이 경우 키 등록 트랜잭션, 별도 키 관리, 계정-합의키 바인딩 검증 로직 등의 운영 오버헤드가 발생한다.

**WIP-6 선택.** 계정의 secp256k1 키 쌍 $(sk_i, PK_i)$를 ECVRF 입력 키로 그대로 사용한다. 블록 헤더에 $PK_i$를 포함하며, 검증자는 먼저 주소 일치를 확인한다:

$$\mathrm{Address}(PK_i) = a_i$$

이 주소 확인 이후, 동일한 $PK_i$로 VRF 증명과 논스별 채굴 서명을 각각 검증한다:

$$\mathrm{ECVRF.Verify}(PK_i,\, M_h,\, y_i,\, \pi_i) = 1$$

$$\mathrm{Verify}_{PK_i}(m_\nu,\, \sigma_\nu) = 1$$

이 두 검증을 통해 블록 생성자가 계정 주소 $a_i$에 대응하는 개인 키를 보유하고 있으며, 동일한 키로 VRF 자격 증명과 채굴 서명을 모두 생성했음을 확인한다. 별도 VRF 키 등록 절차가 없으며, 계정 식별과 VRF 자격 증명이 통합된다.

**이유.** 계정 키 재사용은 별도 VRF 공개 키 등록 절차 없이 기존 Ethereum 계정 인프라를 그대로 활용하며, 구현과 사용자 경험을 단순화한다.

**키 사용 규칙.** VRF 증명과 논스별 채굴 서명은 coinbase 주소에 대응하는 실제 계정 개인 키로 생성되어야 한다.

### Verifiable Coin Toss 함수

$\mathsf{phash}_{h-1}$을 높이 $h-1$의 부모 블록 해시라 하자. 높이 $h$에서 계정 $a_i$는 다음을 계산한다:

$$(y_i, \pi_i) \leftarrow \mathrm{ECVRF.Prove}\!\left(sk_i,\; \texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h\right)$$

VCT는 32-byte VRF 출력 `beta` 전체를 big-endian unsigned 256-bit integer로 해석하고, 블록 헤더에 포함된 합의 임계값 `sortitionThreshold`와 비교한다. 다음 조건을 만족하면 계정 $a_i$는 높이 $h$에 대해 **기본(즉시) VRF 자격**을 얻는다:

$$\mathrm{uint256}(\beta_i) < \texttt{sortitionThreshold}_h$$

여기서 `sortitionThreshold`는 $0 < P_h \le 2^{256}$ 범위의 uint256 값이다. 즉 기본 즉시 자격 확률은 $P_h / 2^{256}$이다. `sortitionThreshold = 2^256`이면 모든 가능한 VRF 출력이 자격을 얻고, `sortitionThreshold = 2^253`이면 기본 즉시 자격 확률은 $1/8$이다.

### 점진적 타임아웃

고정 자격 임계값 `sortitionThreshold`만을 사용하면, 블록 높이 $h$에서 모든 채굴 참여자가 동시에 부적격(ineligible)인 경우 체인이 영구적으로 정지할 수 있다. $N$명의 독립 채굴자가 있고 기본 확률이 $p_h = P_h/2^{256}$일 때 모든 채굴자가 기본 임계값에서 부적격일 확률은 $(1 - p_h)^N$이며, 채굴자 수가 적을수록 이 위험이 커진다. 점진적 타임아웃은 부모 블록 대비 경과 시간 $\Delta t$가 커질수록 자격 임계값을 자동으로 확대하여 활성성(liveness)을 보장하는 메커니즘이다.

**경과 블록 시간.** 높이 $h$의 후보 블록에 대해:

$$\Delta t := \text{block.timestamp}_h - \text{block.timestamp}_{h-1}$$

$\Delta t$는 블록 헤더의 타임스탬프를 사용하므로, 모든 검증 노드가 동일한 값으로 결정론적으로 계산할 수 있다. 노드의 로컬 벽시계(wall clock)는 사용하지 않는다.

**시간 의존 자격 임계값.** 파라미터 $t_s$, $t_e$ ($t_e > t_s \geq 0$)를 정의한다:

- $t_s$: 타임아웃 시작 임계값(초). $\Delta t < t_s$이면 자격 임계값이 헤더의 기본값 `sortitionThreshold`로 유지된다.
- $t_e$: 타임아웃 종료 임계값(초). $\Delta t \geq t_e$이면 모든 채굴자가 자격을 얻는다.
- $\tau_h$: 지수 스케일 상수. $\tau_h = \dfrac{t_e - t_s}{\ln(2^{256} / P_h)}$, where $P_h$ is the block header's `sortitionThreshold`.

시간 의존 자격 임계값 `P(Δt)`는 다음과 같이 정의된다:

$$P(\Delta t) = \begin{cases} P_h & \text{if } \Delta t < t_s \\ \left\lfloor P_h \cdot \exp\!\left(\dfrac{\Delta t - t_s}{\tau_h}\right) \right\rfloor & \text{if } t_s \leq \Delta t < t_e \\ 2^{256} & \text{if } \Delta t \geq t_e \end{cases}$$

이 정의에서 $P(t_s) = P_h$이고 $P(t_e) = 2^{256}$이므로, 타임아웃 종료 시점에는 모든 32-byte VRF 출력이 자격을 얻는다.

**VRF 임계값.** 자격 조건은 다음과 같이 uint256 비교로 판정한다:

$$\mathrm{uint256}(\beta_i) < P(\Delta t)$$

`P(Δt) = 2^256`인 경우에는 모든 가능한 VRF 출력이 조건을 만족하는 것으로 처리한다.

**채굴자의 제출 대기 시간.** $\mathrm{uint256}(\beta_i) \geq P_h$인 채굴자(기본 임계값에서 부적격)는 $P(\Delta t) > \mathrm{uint256}(\beta_i)$가 되는 최소 경과 시간 $t_\text{submit}(\beta_i)$까지 기다린 후 블록을 제출한다:

$$t_\text{submit}(\beta_i) = t_s + \tau_h \cdot \ln\!\left(\frac{\mathrm{uint256}(\beta_i) + 1}{P_h}\right)$$

즉, 해당 채굴자는 $\text{block.timestamp}_{h-1} + t_\text{submit}(\beta_i)$ 이상의 타임스탬프를 사용한 블록만 제출할 수 있다. $\mathrm{uint256}(\beta_i) < P_h$인 채굴자는 대기 없이($t_\text{submit} = 0$) 즉시 채굴을 시작할 수 있다.

**결정론적 검증.** 검증 노드는 제출된 블록의 $\text{block.timestamp}_h$와 부모 블록의 $\text{block.timestamp}_{h-1}$로 $\Delta t$를 계산하고, `P(Δt)`를 재현하여 $\mathrm{uint256}(\beta_i) < P(\Delta t)$ 조건을 판정한다. 노드의 로컬 시간에 의존하지 않으므로, 동일한 블록 헤더에 대해 네트워크 내 모든 노드가 동일한 판정을 내린다.

**기본 파라미터.** 목표 기본 임계값은 `P_base = 2^253`($p^\star = 1/8$), $t_s = 15$초, $t_e = 60$초로 둔다. 하드포크 직후 bootstrap 구간에서는 `sortitionThreshold = 2^256`으로 시작하여 활성성을 보장하고, 이후 deterministic controller를 통해 `P_base`까지 낮춘다.

**미래 타임스탬프 허용.** 기존 ECCPoW 클라이언트는 네트워크 지연과 노드 시계 오차를 고려하여 제한적인 미래 타임스탬프를 허용한다. 그러나 VCT에서는 블록 타임스탬프가 점진적 타임아웃 자격 판정에 직접 사용되므로, 과도한 미래 타임스탬프 허용은 채굴자가 실제 대기 시간보다 빠르게 타임아웃 자격을 얻는 데 악용될 수 있다.

VCTBlock 이후 클라이언트는 미래 타임스탬프 허용 상수 `allowedFutureBlockTimeSeconds`를 5초로 줄인다. 현재 시각보다 5초를 초과하여 미래인 블록은 future block으로 처리한다. 점진적 타임아웃의 자격 임계값은 부모 블록과 후보 블록의 헤더 타임스탬프 차이 $\Delta t$를 사용하여 계산한다:

$$\mathrm{uint256}(\beta_i) < P(\Delta t)$$

### 논스별 채굴 서명과 sealHash

VCT는 WIP-2 ECCPoW의 LDPC decoder, parity-check matrix 생성 방식, codeword 유효성 조건을 변경하지 않는다. 변경되는 부분은 ECCPoW hash vector $r$를 생성하는 입력 seed이다.

**기존 설계.** WIP-2의 ECCPoW에서는 현재 블록 헤더 $\mathrm{CBH}$와 nonce $\nu$를 사용하여 hash vector를 생성한다:

$$s_1 := \mathrm{Keccak512}(\mathrm{Keccak256}(\mathrm{CBH}) \| \nu).$$

이 설계에서 ECCPoW 입력은 공개된 block template과 nonce만으로 구성되므로, VRF 자격 증명 $(y_i, \pi_i, PK_i)$가 공개된 이후 제3자가 동일한 입력으로 nonce 탐색을 대신 수행(offline delegated mining)할 수 있다.

**WIP-6 선택.** ECCPoW 입력을 계정 개인키에 바인딩된 mining seed로 대체한다. 먼저 채굴자는 nonce loop 전에 block template commitment인 $\texttt{sealHash}$를 계산한다:

$$\texttt{sealHash} := \mathrm{Keccak256}\!\left(\texttt{VCT}\_\texttt{SEAL} \| \mathrm{RLP}(\mathsf{HeaderFields},\, \mathsf{VRFFields},\, \mathsf{ECCPoWParams})\right)$$

`HeaderFields`의 정확한 순서와 필드 목록은 consensus-critical하다. 각 클라이언트가 임의로 선택해서는 안 되며, WorldLand 클라이언트 구현에서 하나의 고정된 ordered tuple로 정의되어야 한다.

최소한 `sealHash`는 다음 항목에 커밋해야 한다:
- chainId
- parent block hash $\mathsf{phash}_{h-1}$
- block height $h$
- block producer address $a_i$
- timestamp
- transaction root, state root, receipt root
- gas limit 및 gas used
- VRF public key $PK_i$, VRF output $y_i$, VRF proof $\pi_i$
- ECCPoW CodeLength 및 고정 ECCPoW parameter

다음 항목은 ECCPoW 탐색 과정에서 결정되므로 `sealHash`에서 제외한다:
- nonce $\nu$, mining signature $\sigma_\nu$, Codeword

각 nonce $\nu$에 대해 채굴자는 다음 메시지를 서명한다:

$$m_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu),$$

$$\sigma_\nu := \mathrm{Sign}_{sk_i}(m_\nu).$$

그 다음 ECCPoW hash-vector seed를 다음과 같이 정의한다:

$$\texttt{powSeed}_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu).$$

ECCPoW hash vector $r$는 $\texttt{powSeed}_\nu$로부터 생성된다:

$$s_1 := \mathrm{Keccak512}(\texttt{powSeed}_\nu).$$

$n > 512$인 경우에는 WIP-2와 동일하게 반복 해싱을 통해 필요한 길이의 hash vector를 생성한다:

$$s_u := \mathrm{Keccak512}(s_{u-1}), \quad u \geq 2.$$

이후 LDPC decoder $D_{\mathrm{mp}}$, parity-check matrix $H$, codeword $c$, 그리고 유효성 조건 $Hc = 0$, $\frac{n}{4} < W(c) < \frac{3n}{4}$는 WIP-2 ECCPoW와 동일하게 유지된다.

**이유.** $m_\nu$와 $\texttt{powSeed}_\nu$에서 $\texttt{chainId}$를 제거한 것은, $\texttt{chainId}$가 이미 $\texttt{sealHash}$의 $\mathsf{HeaderFields}$ 내에 포함되어 있기 때문이다. $\texttt{sealHash}$에 커밋된 이후 다시 포함하는 것은 중복이며, 입력 구성을 단순화한다. 이 구조에서 $sk_i$를 보유하지 않은 외부자는 유효한 $\sigma_\nu$를 생성할 수 없고, 결과적으로 유효한 ECCPoW hash-vector seed도 생성할 수 없다.

### 블록 생성 및 검증

계정 $a_i$의 운영자는 다음 채굴 루프를 실행한다:

```
1.  Select the current canonical parent block at height h-1;
    let phash_{h-1} be its hash, t_{h-1} be its timestamp
2.  Check that Balance_{h-ℓ}(a_i) ≥ S_0   (ℓ = 1 in WIP-6)
3.  (y_i, π_i) ← ECVRF.Prove(sk_i, VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h)
4.  β_i ← VRF output hash derived from π_i
5.  if uint256(β_i) < sortitionThreshold_h then
        ▷ 즉시 자격: 기본 임계값 이내
        t_h ← now()                        ▷ 현재 시간을 블록 타임스탬프로 사용
    else                                   ▷ 대기 자격: 타임아웃 이후 제출 가능
        t_h ← t_{h-1} + t_s + τ_h · ln((uint256(β_i) + 1) / sortitionThreshold_h)
              ▷ τ_h = (t_e - t_s) / ln(2^256 / sortitionThreshold_h)
        Wait until local_time ≥ t_h (or peer block h arrives — then abort)
    end if
    ▷ 이 시점에서 Δt = t_h - t_{h-1}이면 uint256(β_i) < P(Δt)가 보장됨
6.  sealHash ← Keccak256(VCT_SEAL ∥ RLP(HeaderFields[timestamp=t_h], VRFFields, ECCPoWParams))
    ▷ HeaderFields contains chainId; sealHash excludes ν, σ_ν, Codeword
7.  for each nonce ν = 0, 1, 2, … do
8.      m_ν ← Keccak256(VCT_MINE ∥ sealHash ∥ ν)
9.      σ_ν ← Sign_{sk_i}(m_ν)
10.     powSeed_ν ← Keccak256(VCT_ECCPOW ∥ sealHash ∥ ν ∥ σ_ν)
11.     s_1 ← Keccak512(powSeed_ν)  ▷ ECCPoW hash-vector seed
12.     Generate hash vector r from s_1 (extend with Keccak512 if n > 512)
13.     Run LDPC decoder D_mp(r, H) to obtain candidate codeword c
14.     if Hc = 0 and n/4 < W(c) < 3n/4 then
15.         Construct block h with (ν, Codeword=c, CodeLength=n, a_i, PK_i, y_i, π_i, σ_ν, timestamp=t_h)
16.         Broadcast block h; break
17.     end if
18. end for
```

**계정 서명의 역할.** 논스별 채굴 서명 $\sigma_\nu$는 ECCPoW nonce 탐색 과정에서 각 ECCPoW trial을 계정 개인 키 $sk_i$에 바인딩하기 위한 서명이다. 채굴자는 각 nonce $\nu$에 대해 $m_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$를 계산하고, $\sigma_\nu := \mathrm{Sign}_{sk_i}(m_\nu)$를 생성한 뒤, $\texttt{powSeed}_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu)$를 사용하여 ECCPoW hash vector $r$를 생성한다.

따라서 공개된 VRF 자격 증명 $(y_i, \pi_i, PK_i)$만으로는 제3자가 유효한 ECCPoW trial을 독립적으로 생성할 수 없다. 각 trial에는 $a_i$의 개인 키로 생성된 유효한 $\sigma_\nu$가 필요하기 때문이다. 단, 이 구조가 모든 형태의 위탁 채굴을 암호학적으로 불가능하게 만드는 것은 아니다. 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하면 외부 연산자가 서명된 nonce 후보를 받아 ECCPoW 계산을 수행할 수 있다. VCT의 보안 주장은 "공개된 VRF 자격 증명만으로 수행되는 offline delegated mining을 방지한다"로 제한된다.

수신 노드는 높이 $h$의 후보 블록을 다음 순서로 검증한다:
```
1.  Address(PK_i) = a_i
2.  Balance_{h-ℓ}(a_i) ≥ S_0   (ℓ = 1 in WIP-6)
3.  (y_i', valid) ← ECVRF.VerifyAndHash(PK_i,
        VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h, π_i)
4.  Require valid = 1  ▷ π_i가 PK_i와 메시지에 대해 유효한 증명인지 확인
5.  Require y_i' = y_i
6.  Δt ← block.timestamp_h - block.timestamp_{h-1}
7.  Require uint256(β_i) < P(Δt)
    ▷ 헤더의 sortitionThreshold 대신 시간 의존 uint256 임계값 사용; P(Δt)는 점진적 타임아웃 절 참조
8.  Recompute sealHash ← Keccak256(VCT_SEAL ∥ RLP(HeaderFields, VRFFields, ECCPoWParams))
    ▷ HeaderFields contains chainId; sealHash excludes ν, σ_ν, Codeword
9.  m_ν ← Keccak256(VCT_MINE ∥ sealHash ∥ ν)
10. Require ECRecover(σ_ν, m_ν) = a_i
    ▷ σ_ν의 서명자가 헤더에 명시된 블록 생성자 주소 $a_i$와 일치하는지 확인; valid secp256k1 ECDSA, low-S form
11. powSeed_ν ← Keccak256(VCT_ECCPOW ∥ sealHash ∥ ν ∥ σ_ν)
12. s_1 ← Keccak512(powSeed_ν)
13. Generate hash vector r from s_1
14. Regenerate parity-check matrix H from CodeLength and fixed degree parameters
15. Verify that LDPC decoder D_mp(r, H) produces codeword c = header.Codeword
16. Verify Hc = 0 and n/4 < W(c) < 3n/4
```
포크 선택은 누적 체인 작업량 기준 Nakamoto 최장 체인 규칙을 따른다. 블록 $B_h$의 작업량은 $W(B_h) = 1/T_h$이며, 체인 점수는 $W(\text{chain}) = \sum_h W(B_h)$이다. 위 15가지 검증 조건 중 하나라도 실패한 블록은 작업량에 기여하지 않는다. 자격 임계값 $p_h$는 채굴 참여 자격을 제어하는 역할을 하며, 체인 작업량 계산에는 포함되지 않는다.

### p 조정

이 WIP는 timeout 빈도 기반 동적 `P_base` 조정을 도입하지 않는다. 목표 sortition 임계값은 `P_base = 2^253`($p^\star = 1/8$)이며, 하드포크 직후에는 `sortitionThreshold = 2^256`에서 시작하여 deterministic bootstrap controller가 목표값까지 낮춘다. 블록 시간이 지연되는 경우에는 [점진적 타임아웃](#점진적-타임아웃)의 시간 의존 임계값 `P(Δt)`가 해당 블록의 활성성을 보완한다.

이 결정을 내리는 이유는 `P_base`와 ECCPoW difficulty가 모두 블록 생성 시간에 영향을 주기 때문이다. `P_base`는 블록당 즉시 자격을 얻는 계정 수를 조절하는 coarse admission-control 파라미터이고, ECCPoW difficulty는 자격을 얻은 채굴자가 유효 nonce를 찾는 시간을 조절하는 fine-grained block-time 파라미터다. 두 값을 동시에 블록 시간 신호만으로 조정하면 제어 루프가 간섭하여 oscillation이나 과도한 stale block을 만들 수 있다.

또한 자격 채굴자 수는 자연수이므로 `P_base`를 미세하게 조정해도 실제 eligible proposer 수가 매끄럽게 변하지 않을 수 있다. 특히 자격 계정 수가 작을 때는 sortition probability가 정밀한 난이도 대체물이 될 수 없다. 따라서 이 WIP에서는 `P_base`를 고정하고, ECCPoW difficulty가 평균 블록 시간을 조정하며, progressive timeout은 즉시 자격자가 없을 때의 liveness fallback으로만 사용한다.

## Rationale

**잔액 기반 자격 조건.** 별도 스테이킹 컨트랙트나 검증자 레지스트리 없이 부모 블록 상태의 계정 잔액만으로 자격을 확인하는 방식을 선택한 이유는, 등록 트랜잭션·lock-up·슬래싱 메커니즘 없이 Sybil 공격 비용을 부과하면서 설계 복잡성을 최소화하기 위해서다. $S_0$는 지분 비례 영향력이 아니라 ECCPoW 참여의 최소 진입 비용으로 설계된다. 지분 크기에 비례한 선출 확률을 부여하면 자본 집중에 따른 중앙화 압력이 생기므로, VCT는 임계값을 충족한 모든 계정에 동일한 확률 $p$를 부여한다.

**계정 키 재사용.** 별도의 VRF 공개 키 등록 절차를 제거하고 기존 Ethereum 계정 키 쌍 $(sk_i, PK_i)$를 ECVRF 입력 키로 재사용한다. 이는 키 등록 트랜잭션, 계정-합의키 바인딩 검증, 별도 키 관리 오버헤드를 피하기 위해서이며, 구현과 사용자 경험을 단순화한다. 블록 헤더에 $PK_i$를 포함하고 $\mathrm{Address}(PK_i) = a_i$를 확인함으로써, 블록 생성자가 coinbase 주소에 대응하는 개인 키를 실제로 보유하고 있음을 VRF 증명과 채굴 서명이 일관되게 증명한다.

**논스별 채굴 서명.** 각 ECCPoW trial을 계정 개인 키에 바인딩하여 공개된 VRF 자격 증명만으로 수행 가능한 offline delegated mining을 방지한다. 공유된 block template과 nonce만으로 ECCPoW 입력이 결정된다면, VRF 자격을 얻은 계정 정보가 공개된 즉시 제3자가 동일 입력으로 nonce 탐색을 대신 수행할 수 있다. $\sigma_\nu$를 powSeed에 포함함으로써 유효한 ECCPoW hash-vector seed 생성에 $sk_i$ 보유가 필수가 된다. 단, 온라인 서명 서비스 형태의 위탁은 암호학적으로 차단되지 않는다는 점을 명시한다. $m_\nu$와 $\texttt{powSeed}_\nu$에서 `chainId`를 제거한 것은 중복 제거를 위해서다 — `chainId`는 이미 `sealHash`의 `HeaderFields`에 포함되어 있다.

**Nakamoto 포크 선택.** VRF 자격 임계값 `P_base`는 블록당 기대 채굴 시도자 수를 제어하지만 체인 작업량 계산에는 포함되지 않는다. 최종 포크 선택은 누적 ECCPoW 작업량 기준 최장 체인 규칙을 따른다. 이는 VRF 자격 선발을 BFT 투표나 리더 선출로 전환하지 않고 기존 Nakamoto 합의의 보안 보장을 그대로 유지하기 위해서다. BFT-style 위원회 선출을 도입하면 liveness 분석과 네트워크 가정이 복잡해지므로, 이 WIP는 잔액 기반 사전 자격 확인을 Nakamoto ECCPoW 경쟁 위에 얹는 최소 설계를 채택한다.

**점진적 타임아웃 대 고정 임계값.** 고정 임계값 `P_base`만 사용하면 $N$명의 채굴자가 모두 동시에 부적격일 수 있으며, 이 경우 체인이 영구 정지한다. 채굴자 수를 늘리면 이 확률이 지수적으로 감소하지만, 초기 네트워크에서는 $N$이 작아 위험이 높다. 점진적 타임아웃은 경과 시간 $\Delta t$에 따라 바이트 임계값 `P(Δt)`를 확대하여, 충분한 시간이 지나면 반드시 누군가가 블록을 생성할 수 있도록 활성성을 보장한다. 이 메커니즘이 고정 타임아웃(예: Δt ≥ T_hard이면 즉시 모든 채굴자 자격)보다 점진적 확대를 택한 이유는, 급격한 임계값 전환이 블록 스탬핑 공격이나 타임스탬프 조작 인센티브를 증가시키기 때문이다. 단, 타임스탬프 조작에 의한 미약한 그라인딩 여지는 명시적 한계로 남는다.

**VRF 임계값 조정 범위.** 자격 확률을 Bitcoin difficulty처럼 실수 범위에서 연속적으로 조정하면 두 가지 문제가 생긴다. 첫째, 자격 계정 수는 자연수이므로 임계값을 아주 미세하게 바꿔도 실제 자격자 수가 달라지지 않을 수 있다. 둘째, 블록 시간을 이미 ECCPoW difficulty가 조정하고 있으므로, VRF 임계값도 블록 시간 신호로 조정하면 두 제어 루프가 간섭한다. 따라서 이 WIP는 timeout 빈도 기반 `P_base` 조정을 활성화하지 않는다. 다만 하드포크 직후 활성성 보장을 위해 `sortitionThreshold = 2^256`에서 시작하여 목표값 `P_base = 2^253`까지 낮추는 bootstrap controller는 적용한다.

## Security Analysis / 보안 분석

### Sybil 비용

$S_0$는 계정당 자격 임계값이면서 동시에 Sybil 공격의 단위 비용(Sybil cost parameter)이다. 총 잔액 $B_A$를 보유한 공격자가 생성할 수 있는 최대 자격 계정 수는:

$$N_A = \left\lfloor \frac{B_A}{S_0} \right\rfloor$$

$S_0$를 높이면 공격자당 최대 계정 수가 줄어들고, Sybil 공격의 자본 비용이 계정 수에 선형으로 증가한다. 반대로 $S_0$를 낮추면 소액 보유자의 참여가 허용되지만 Sybil 공격 비용이 낮아진다. 프로토콜 설계자는 이 트레이드오프를 감안하여 $S_0$를 결정해야 한다.

### ECCPoW 보안

$N_H$와 $N_A$를 각각 정직한 네트워크와 공격자가 제어하며 실제로 채굴을 시도하는 잔액 조건 충족 계정 수라 하자. 각 계정은 확률 $p_h$로 블록당 하나의 VRF 자격 기회를 받는다.

블록 높이 $h$에서 자격을 갖춘 정직한 계정과 공격자 계정 수는 다음과 같다:

$$X_H \sim \text{Binomial}(N_H, p_h), \qquad X_A \sim \text{Binomial}(N_A, p_h)$$

$(X_H, X_A) = (x_H, x_A)$로 조건화하면 공격자가 ECCPoW 경쟁에서 이길 확률은:

$$\Pr[A \text{ wins} \mid x_H, x_A] = \frac{h_A x_A}{h_A x_A + h_H x_H}$$

여기서 $h_A$와 $h_H$는 자격을 갖춘 계정당 유효 ECCPoW 처리량을 나타낸다. $N_H$, $N_A$가 클 때 $p_h$가 소거되어:

$$\Pr[A \text{ wins}] \approx \frac{h_A N_A}{h_A N_A + h_H N_H}$$

ECCPoW의 ASIC 저항성 하에서 ($h_A \approx h_H$):

$$\Pr[A \text{ wins}] \approx \frac{N_A}{N_A + N_H}$$

지속적인 다수 공격은 다음 조건을 요구한다:

$$h_A N_A > h_H N_H$$

계정 수만으로는 부족하며 공격자는 충분한 ECCPoW 처리량도 보유해야 한다.

### 부모 해시 그라인딩

VRF 입력이 canonical parent block hash를 포함하므로, VCT는 부모 해시 그라인딩을 완전히 제거하지 않는다. 블록 생성자는 트랜잭션 순서, 타임스탬프 선택, 블록 withholding 등을 통해 다음 VRF 입력에 제한적으로 영향을 줄 수 있다.

WIP-6은 이를 명시적 한계로 둔다. 이 문서의 목적은 최소한의 balance-gated VRF-ECCPoW 파이프라인을 구현하는 것이므로, VDF나 외부 randomness beacon은 도입하지 않는다.

### 위탁 채굴 저항성

VCT의 계정 바인딩 ECCPoW는 공개된 VRF 자격 증명만을 이용한 독립적 위탁 채굴을 방지한다. 일반적인 ECCPoW 입력이 공개 block template과 nonce만으로 구성된다면, VRF 자격을 얻은 계정의 정보가 공개된 이후 제3자가 동일한 자격 정보를 사용하여 nonce 탐색을 대신 수행할 수 있다.

VCT에서는 각 ECCPoW trial이 다음 값에 의존한다:

$$\texttt{powSeed}_\nu = \mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu),$$

여기서

$$\sigma_\nu = \mathrm{Sign}_{sk_i}(\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)).$$

따라서 $sk_i$를 보유하지 않은 외부자는 유효한 $\sigma_\nu$를 생성할 수 없고, 결과적으로 유효한 ECCPoW hash-vector seed도 생성할 수 없다. 이는 VRF proof가 공개된 이후에도 제3자가 공개 데이터만으로 ECCPoW nonce search를 독립적으로 수행하는 것을 막는다.

그러나 이 저항성은 완전한 mining-pool 방지를 의미하지 않는다. 계정 소유자가 개인키를 직접 공유하거나, nonce별 서명을 제공하는 온라인 signing service를 운영한다면 외부 worker가 ECCPoW 계산을 수행할 수 있다. VCT는 이러한 online delegation을 암호학적으로 제거하지 않는다. 대신 각 ECCPoW trial에 계정 서명 능력을 요구함으로써 공개 자격 증명만으로 가능한 offline delegation을 차단하고, 위탁 채굴의 운영 비용과 키 관리 위험을 증가시킨다.

### 점진적 타임아웃과 활성성

점진적 타임아웃은 다음 방식으로 체인 정지 문제를 해결한다. 모든 $N$명의 채굴자가 기본 임계값에서 부적격($\mathrm{uint256}(\beta_i) \geq P_h$)인 상황에서도, 경과 시간 $\Delta t$가 $t_e$에 도달하면 `P(Δt) = 2^256`이 되어 모든 채굴자가 자격을 얻는다. 따라서 체인은 최대 $t_e$초 지연 후 반드시 다음 블록이 생성된다.

$\mathrm{uint256}(\beta_i) < P_h$인 즉시 자격 채굴자가 한 명이라도 존재하면, 해당 채굴자가 정상 블록을 생성하여 타임아웃 없이 체인이 진행된다. $t_s$를 목표 블록 간격 $\Delta_\text{target}$과 동일하게 설정하면, 네트워크가 정상 동작할 때 타임아웃이 거의 발생하지 않는다.

**타임스탬프 조작.** 채굴자가 자신의 VRF 출력에 유리한 블록 타임스탬프를 선택하여 조기에 블록을 제출하려 할 수 있다. 그러나 $\Delta t$가 줄어들면 `P(Δt)`도 줄어들므로, 타임스탬프를 과거로 당기면 오히려 자격 임계값이 낮아져 유리하지 않다. 반대로 미래 타임스탬프를 사용하면 더 높은 임계값을 얻을 수 있지만, VCTBlock 이후 클라이언트는 미래 타임스탬프 허용값을 5초로 줄여 조기 제출의 이득을 제한한다. 또한 VRF 출력은 VRF 입력(phash, 높이)에 의해 이미 결정되므로, 채굴자가 출력 자체를 선택할 수 없다.

**공정한 대기.** uint256으로 해석한 VRF 출력이 낮은 채굴자가 먼저 블록을 제출하고, 출력이 높은 채굴자는 더 오래 기다린다. 이는 VRF 출력에 비례한 대기 순서로, 동일 블록에 여러 채굴자가 자격을 얻는 경우 ECCPoW 경쟁으로 최종 블록이 결정된다는 점에서 기존 Nakamoto 방식의 보안 보장을 유지한다.

### 키 보안

계정 개인 키 $sk_i$가 ECVRF 계산과 논스별 채굴 서명 모두에 사용되며, 각 ECCPoW trial마다 서명이 요구되므로 핫 키가 채굴 노드에 상주해야 한다. 이는 키 노출 위험을 증가시키며, 채굴 노드 운영 시 적절한 키 관리 절차가 필요하다.

서명 키가 핫 환경에 있다는 사실 자체는 서명 기반 채굴 시스템에서 불가피하다. 핵심 완화 방법은 노출 키의 **폭발 반경(blast radius)**을 줄이는 것이다: 채굴 계정에는 $S_0$ 충족에 필요한 최소 잔액만 유지하고, 나머지 자산은 별도 콜드 계정에 보관한다. 이렇게 하면 채굴 노드 침해 시 손실이 채굴 계정 잔액으로 한정된다. 향후 온체인 키 위임 메커니즘(operator key 등록)을 통해 계정 자금 키와 채굴 서명 키를 분리하는 방안을 v2 후보로 검토할 수 있다.

### Snap Sync와 S₀ 검증

S₀는 VCTBlock 이후의 합의 규칙이며, proposer eligibility는 부모 상태 잔액에 의존한다. 따라서 VCT 블록 검증 시 부모 상태를 로드할 수 없으면 S₀ 검증을 건너뛰지 않고 블록을 거부한다.

**설계 결정: 부모 상태 없음 → S₀ 검증 실패.**

이 결정의 근거는 다음과 같다.

1. $S_0$는 VCTBlock 이후 합의 규칙의 일부이므로, 상태 의존 검증을 수행할 수 없는 블록은 완전하게 검증된 블록이 아니다.
2. P2P 블록 import 경로와 로컬 채굴자가 생성한 블록을 체인에 쓰는 경로 모두에서 동일하게 S₀ 검증을 강제해야 한다.
3. header-only 또는 snap sync 중 부모 상태가 없는 구간에서는 해당 블록을 최종 실행 검증된 canonical block으로 간주하지 않는다.

따라서 참조 구현은 부모 상태 로딩 실패를 검증 실패로 처리해야 한다. 이 정책은 "검증 우회 불가"라는 WIP-6의 합의 주장을 명확히 하기 위한 선택이다.

## Backwards Compatibility / 하위 호환성

WIP-6은 하드 포크를 통해 활성화되며, 이전 클라이언트와 호환되지 않는 프로토콜 변경을 포함한다.

**프로토콜 수준 비호환성.** VCTBlock 이후에는 다음 규칙이 기존 규칙을 대체한다.

- **논스별 채굴 서명 공식 변경.** $m_\nu$ 계산에서 `chainId`가 제거된다. `chainId`는 이미 `sealHash` 내부의 `HeaderFields`에 포함되어 있으므로, VCTBlock 이전의 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{chainId} \| \texttt{sealHash} \| \nu)$ 공식은 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$로 단순화된다.

- **ECCPoW seed 공식 변경.** 동일한 이유로 $\texttt{powSeed}_\nu$ 계산에서도 `chainId`가 제거된다.

- **VRF 입력 메시지 변경.** VRF 입력이 epoch 기반 sortition seed hash에서 $\texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h$로 변경된다. 이 변경으로 VRF 자격이 특정 chain, 특정 부모 블록, 특정 높이에 명시적으로 바인딩된다.

- **주소 검증 추가.** 블록 헤더의 $PK_i$로부터 유도한 주소가 $a_i$와 일치하는지 ($\mathrm{Address}(PK_i) = a_i$) 검증하는 단계가 추가된다.

**업그레이드 경로.** VCTBlock 이전까지 모든 노드는 기존 ECCPoW 검증 규칙을 따른다. VCTBlock 활성화 이후, WIP-6을 구현하지 않은 노드는 새 규칙으로 생성된 블록을 거부하여 네트워크에서 분리된다. 충분히 먼 미래의 블록 번호를 VCTBlock으로 설정하여 네트워크 참여자가 클라이언트를 업그레이드할 시간을 확보한다.

**pre-VCT 레거시 호환성.** VCT 엔진을 탑재한 클라이언트라도 VCTBlock 이전 블록은 기존 ECCPoW와 동일하게 처리해야 한다. pre-VCT 블록은 `VCT_SEAL` 도메인 프리픽스를 포함하지 않는 레거시 `sealHash`를 사용하고, ECCPoW hash-vector seed도 기존 `sealHash ∥ nonce` 입력을 사용한다. 이 구간에서는 VRF sortition, VRF proof/public key, 논스별 채굴 서명, $S_0$ 잔액 검증을 적용하지 않는다. 이렇게 해야 기존 ECCPoW 노드가 생성한 pre-VCT 블록과 VCT 엔진이 생성한 pre-VCT 블록의 `MixDigest` 및 state root가 일치한다.

**채굴자 요구 사항.** VCTBlock 활성화 이후 채굴자는 각 ECCPoW trial마다 $\sigma_\nu$ 생성에 계정 개인 키가 필요하므로, 개인 키를 채굴 노드에 직접 제공해야 한다.

## Reference Implementation / 구현

WIP-6의 참조 구현은 https://github.com/cryptoecc/WorldLand 에서 관리되는 go-ethereum의 EVM 호환 포크인 WorldLand 클라이언트의 합의 엔진(`consensus/vct/`)에 제공된다. VCTBlock 이전 블록에는 기존 규칙이 적용되며, VCTBlock 이후부터 아래 명세가 활성화된다.

WIP-6은 기존 ECCPoW 클라이언트 대비 다음 일곱 가지 변경을 요구한다.

1. **잔액 기반 자격 확인** — 블록 검증 시 $\mathrm{Balance}_{h-\ell}(a_i) \geq S_0$를 직접 확인한다($\ell = 1$). 별도 레지스트리나 등록 트랜잭션이 필요 없다.

2. **secp256k1 기반 ECVRF 통합** — 클라이언트는 secp256k1 ECVRF 구현을 통합하고, 채굴 루프에서 블록 높이마다 $\mathrm{ECVRF.Prove}(sk_i,\, \texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h)$를 한 번 평가한다. 사용할 secp256k1 ECVRF ciphersuite는 합의 규칙으로 고정되어야 한다.

3. **계정 키 재사용 및 공개 키-주소 바인딩** — 블록 헤더에 $a_i$와 $PK_i$를 포함한다. 검증자는 $\mathrm{Address}(PK_i) = a_i$를 확인한 후, 동일한 $PK_i$로 VRF 증명과 논스별 채굴 서명을 각각 검증한다.

4. **논스별 채굴 서명 및 ECCPoW seed 교체** — 각 nonce $\nu$마다 $m_\nu \leftarrow \mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$를 계산하고 $\sigma_\nu \leftarrow \mathrm{Sign}_{sk_i}(m_\nu)$를 생성한다. 기존 WIP-2의 $\mathrm{Keccak256}(\mathrm{CBH}) \| \nu$ 기반 hash-vector seed를 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu)$로 대체한다. LDPC decoder와 codeword 유효성 조건은 WIP-2와 동일하게 유지한다.

5. **sealHash 및 블록 검증 갱신** — 블록 헤더에 $PK_i$, $y_i$, $\pi_i$, $\nu$, Codeword, CodeLength, $\sigma_\nu$ 필드를 추가한다. 클라이언트는 고정된 HeaderFields에서 `sealHash`를 재계산하고, $\mathrm{ECRecover}(\sigma_\nu, m_\nu) = a_i$를 확인한 뒤, ECCPoW hash vector를 재생성하여 codeword를 검증한다. 검증 실패 블록은 체인 작업량에 기여하지 않는다.

6. **하드포크 경계와 검증 경로** — 클라이언트는 VCTBlock 경계에서 레거시 ECCPoW 규칙과 VCT 규칙을 명확히 분기해야 한다. P2P 블록 import 경로와 로컬 채굴자가 생성한 블록을 체인에 쓰는 경로 모두에서 $S_0$ 자격 검증을 수행해야 하며, 부모 상태를 로드할 수 없는 경우 검증을 건너뛰지 않고 블록을 거부해야 한다.

7. **adaptive sortition 임계값** — VCT 클라이언트는 블록 헤더의 `sortitionThreshold`를 검증하고, 하드포크 직후 `2^256`에서 시작하여 목표값 `P_base = 2^253`($p^\star = 1/8$)까지 낮추는 deterministic bootstrap controller를 적용한다.

구현이 완료된 것으로 간주되려면 다음 테스트 요구 사항을 충족해야 한다.

- [x] `computeVRFMsg` 출력이 `VCT_VRF ∥ chainId ∥ phash ∥ h` 포맷과 일치하는지 검증하는 단위 테스트
- [x] `computeMiningSigMsgVCT` 출력이 `VCT_MINE ∥ sealHash ∥ ν` 포맷과 일치하는지 검증하는 단위 테스트
- [x] `computePowSeedVCT` 출력이 `VCT_ECCPOW ∥ sealHash ∥ ν ∥ σ` 포맷과 일치하는지 검증하는 단위 테스트
- [x] VRF 증명 생성·검증 왕복 테스트 (`VRFProve` → `VRFVerify`)
- [x] `Address(PK_i) = coinbase` 검증: 일치하는 경우 통과, 불일치하는 경우 거부 테스트
- [ ] `uint256(beta) < sortitionThreshold` 기본 자격 임계값 판정 테스트 (경계값 포함)
- [x] `ECRecover(σ_ν, m_ν) = a_i` 채굴 서명 검증 테스트
- [x] ECCPoW hash vector가 VCT seed (`powSeedVCT`)와 레거시 seed (`powSeed`) 각각에서 올바르게 생성되는지 테스트
- [x] VCTBlock 경계 테스트: VCTBlock-1은 레거시 규칙, VCTBlock은 WIP-6 규칙 적용
- [ ] 잘못된 VRF 증명, 잘못된 채굴 서명, ECCPoW 실패 블록이 각각 거부되는 통합 검증
- [x] 점진적 타임아웃: $\Delta t < t_s$일 때 `P(Δt) = P_base`임을 검증하는 단위 테스트
- [x] 점진적 타임아웃: $\Delta t = t_e$일 때 `P(Δt) = 2^256`임을 검증하는 단위 테스트
- [ ] 점진적 타임아웃: $t_\text{submit}(\mathrm{uint256}(\beta_i))$ 이전의 타임스탬프를 사용한 블록이 검증자에 의해 거부되는 테스트
- [x] 점진적 타임아웃: $\Delta t \geq t_e$인 블록에서 임의의 VRF 출력이 모두 자격을 얻는 테스트
- [x] pre-VCT `legacySealHash`가 기존 ECCPoW `SealHash`와 일치하는지 검증하는 단위 테스트
- [ ] `P_base = 2^253` 목표 임계값과 `sortitionThreshold = 2^256` bootstrap 임계값이 올바르게 적용되는지 확인하는 단위 테스트

## References / 참고문헌

[1] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008.
[2] H. Park, S. Kim, and H.-N. Lee, "Time-Varying LDPC Code-Based Proof-of-Work for Cryptocurrency Mining," Symmetry, vol. 12, no. 6, 2020.
[3] H.-N. Lee et al., "Error Correction Code Verifiable Computation Consensus," IEEE, doc. 11048962, 2025.
[4] S. Micali, M. Rabin, and S. Vadhan, "Verifiable Random Functions," in Proc. FOCS, 1999.
[5] J. Chen and S. Micali, "Algorand: A Secure and Efficient Distributed Ledger," Theor. Comput. Sci., vol. 777, pp. 155–183, 2019.
[6] D. Papadopoulos et al., "Making NSEC5 Practical for DNSSEC," 2017; IETF RFC 9381, "Verifiable Random Functions (VRFs)," 2023.
[7] S. Kim et al., "VRF-PoW: Proof of Work Consensus With Verifiable Random Function," IEEE Xplore, doc. 11362952.
[8] J. R. Douceur, "The Sybil Attack," in Proc. IPTPS, 2002.
[9] Ethereum Foundation, go-ethereum crypto/secp256k1 — libsecp256k1 wrapper, https://github.com/ethereum/go-ethereum/tree/master/crypto/secp256k1.
[10] aergoio, secp256k1-vrf — secp256k1 VRF implementation, https://github.com/aergoio/secp256k1-vrf.
## Copyright / 저작권

Copyright and related rights released under the [GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.html).
