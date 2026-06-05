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
  - [블록 헤더 확장](#블록-헤더-확장)
  - [점진적 타임아웃](#점진적-타임아웃)
  - [논스별 채굴 서명과 sealHash](#논스별-채굴-서명과-sealhash)
  - [블록 생성 및 검증](#블록-생성-및-검증)
  - [Sortition 임계값 조정](#sortition-임계값-조정)
* [Rationale](#rationale)
* [Reference Implementation / 구현](#reference-implementation--구현)
* [References / 참고문헌](#references--참고문헌)
* [Copyright / 저작권](#copyright--저작권)

## Abstract / 개요

Verifiable Coin Toss (VCT)는 WorldLand 블록체인을 위한 합의 메커니즘이다. VCT는 잔액 임계값 기반 채굴 자격 조건, secp256k1 기반 VRF 선발, 계정 바인딩 ECCPoW, Nakamoto 방식의 최대 유효 작업량 포크 선택을 결합한다.

각 블록 높이에서, 부모 상태 잔액이 $S_0$ 이상인 계정은 자신의 계정 키를 사용하여 부모 블록 해시와 목표 높이에 대해 secp256k1 기반 VRF를 평가할 수 있다. VRF 출력이 네트워크 자격 임계값 $p$ 미만이면 해당 계정은 ECCPoW를 시도할 자격을 얻는다. 생성된 블록에는 계정 공개 키, VRF 증명, ECCPoW nonce, 그리고 해당 nonce에 대한 논스별 채굴 서명 $\sigma_\nu$가 포함되어야 한다. $\sigma_\nu$는 ECCPoW 탐색 중 각 nonce 시도마다 계정 개인 키로 생성되며, ECCPoW hash-vector seed에 포함된다. 따라서 공개된 VRF 자격 증명만으로는 제3자가 독립적으로 유효한 ECCPoW 입력을 생성할 수 없다. 단, 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하는 경우 위탁 채굴 자체를 암호학적으로 완전히 차단하지는 못한다.

VCT는 검증자 레지스트리나 지분 가중 리더 선출을 도입하지 않는다. 단일 계정에 $S_0$ 이상을 보유해도 해당 계정의 자격 확률은 증가하지 않는다. 여러 자격 시도를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

## Motivation / 동기

PoW(Proof-of-Work) 시스템은 탁월한 검열 저항성을 제공하지만, 모든 노드가 매 블록 경쟁하기 때문에 산업적 규모의 에너지를 소비한다. PoS(Proof-of-Stake) 시스템은 물리적 비용을 자본 가중 투표로 대체하는 대가로 극적인 에너지 절감을 달성한다.

VCT 프로토콜은 다른 위치를 점한다. 스테이킹 컨트랙트나 검증자 레지스트리를 도입하지 않고, 부모 상태의 계정 잔액만을 자격 기준으로 사용한다. VRF 자격 임계값은 각 블록에서 ECCPoW를 수행할 수 있는 계정 수를 제어하며, 최종 블록 경쟁은 여전히 자격을 갖춘 계정들 간의 ECCPoW 연산 경쟁으로 결정된다. 기본 VRF 자격 임계값은 합의 파라미터로 고정하고, 평균 블록 생성 시간 조정은 ECCPoW difficulty가 담당한다.

**초기 네트워크 보안과 잔액 기반 자격 조건.** VRF 기반 블록 선택 체계에서는 계정 수가 직접적인 보안 변수가 된다. 블록당 기대 즉시 자격 계정 수는 $(P_{\mathrm{base}}/256) \cdot N$이므로, 참여 장벽이 없다면 공격자는 계정을 대량 생성하는 것만으로 자격 획득 확률을 임의로 높일 수 있다. 계정 생성 비용이 근본적으로 낮은 Ethereum 계정 모델에서 이는 Sybil 공격에 구조적으로 취약한 설계가 된다 [7, 8]. 특히 초기 네트워크 단계에서는 정직한 참여자 수가 적어 공격자의 상대적 계정 비율이 높아지기 쉽다.

잔액 임계값 $S_0$는 이 취약점을 완화하기 위한 ECCPoW 참여 사전 자격(prerequisite eligibility) 설계다. 각 자격 계정은 $S_0$ 이상의 잔액을 보유해야 하므로, 공격자가 $N_A$개의 자격 계정을 확보하려면 최소 $N_A \times S_0$의 자본을 소비해야 한다. 이는 PoS의 지분 가중 리더 선출과 구분된다. PoS에서는 지분 크기가 선출 확률에 직접 비례하지만, VCT에서는 임계값을 충족한 모든 계정이 동일한 VRF 자격 기회를 받는다. $S_0$는 지분에 비례한 영향력이 아니라 ECCPoW 참여를 위한 최소 진입 비용으로 설계된다.

## Scope / 범위

VCT 합의 규칙은 balance-gated VRF 자격 조건, 계정 바인딩 ECCPoW 입력, VCTBlock 이후의 블록 검증 규칙으로 한정된다. 다음 항목은 범위에 포함하지 않는다.

- `P_base`의 timeout 빈도 기반 동적 조정 또는 ECCPoW difficulty와 결합된 joint controller.
- 장기 lock-up, bonded staking, unbonding period, stake-duration 조건, 최근 여러 블록의 최소/평균 잔액 조건.
- VDF, 외부 randomness beacon, commit-reveal 등 부모 해시 그라인딩을 추가로 완화하는 randomness layer.
- VCT 이후의 블록 보상, uncle 보상, 트레저리 분배, 발행량 정책.

## Specification / 명세

### 참여 자격 조건

$a_i$를 WorldLand 계정 주소, $PK_i$를 그 secp256k1 공개 키라 하면 $a_i = \mathrm{Address}(PK_i)$이다. 계정은 다음 조건을 만족할 때 블록 높이 $h$에 대해 채굴 자격을 갖춘다:

$$\mathrm{Balance}_{h-\ell}(a_i) \geq S_0$$

여기서 $S_0$는 프로토콜 파라미터로 정해진 잔액 임계값이고, $\ell \geq 1$은 잔액 참조 블록의 룩백 깊이(lookback depth)이다.

$\ell = 1$을 기본값으로 채택한다. 즉, 블록 높이 $h$에 대한 자격은 부모 블록($h - 1$) 상태 잔액으로 판단한다. 별도 레지스트리나 등록 트랜잭션 없이 블록 검증 시 부모 상태에서 직접 확인된다.

**구현 파라미터.** 클라이언트 구현은 하드포크 설정에 포함된 `VctConfig.MinEligibleBalance`를 $S_0$ 값으로 사용한다. 모든 검증 노드는 동일한 하드포크 설정에서 동일한 $S_0$ 값을 사용해야 한다.

이 조건은 장기 lock-up이나 bonded staking을 의미하지 않는다. $S_0$는 지분 지속 기간을 강제하는 장치가 아니라 Sybil 계정 생성과 ECCPoW 참여에 필요한 최소 유동성 비용을 부과하는 장치다.

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

$\pi_i$는 위 ciphersuite가 생성하는 81-byte proof object를 의미한다. VCT는 deterministic ECDSA signature를 VRF proof로 재해석하지 않는다.

### VRF 키 쌍 사용 방식

WorldLand 계정은 secp256k1 개인 키 $sk_i$와 공개 키 $PK_i$를 보유하며, 계정 주소는 다음과 같이 유도된다:

$$a_i = \mathrm{Address}(PK_i) = \mathrm{Keccak256}(PK_i)[12{:}]$$

VCT는 이 계정 키 쌍을 ECVRF 입력 키로 재사용한다.

**기존 설계.** 별도의 VRF 키 쌍을 생성하여 온체인에 등록하거나, 합의 레이어 전용 키를 계정 키와 분리하는 방식이 일반적이다. 이 경우 키 등록 트랜잭션, 별도 키 관리, 계정-합의키 바인딩 검증 로직 등의 운영 오버헤드가 발생한다.

계정의 secp256k1 키 쌍 $(sk_i, PK_i)$를 ECVRF 입력 키로 그대로 사용한다. 블록 헤더에 $PK_i$를 포함하며, 검증자는 먼저 주소 일치를 확인한다:

$$\mathrm{Address}(PK_i) = a_i$$

이 주소 확인 이후, 동일한 $PK_i$로 VRF 증명과 논스별 채굴 서명을 각각 검증한다:

$$\mathrm{ECVRF.Verify}(PK_i,\, M_h,\, \pi_i) = \beta_i$$

$$\mathrm{Verify}_{PK_i}(m_\nu,\, \sigma_\nu) = 1$$

이 두 검증을 통해 블록 생성자가 계정 주소 $a_i$에 대응하는 개인 키를 보유하고 있으며, 동일한 키로 VRF 자격 증명과 채굴 서명을 모두 생성했음을 확인한다. 별도 VRF 키 등록 절차가 없으며, 계정 식별과 VRF 자격 증명이 통합된다.

**이유.** 계정 키 재사용은 별도 VRF 공개 키 등록 절차 없이 기존 Ethereum 계정 인프라를 그대로 활용하며, 구현과 사용자 경험을 단순화한다.

**키 사용 규칙.** VRF 증명과 논스별 채굴 서명은 coinbase 주소에 대응하는 실제 계정 개인 키로 생성되어야 한다.

### Verifiable Coin Toss 함수

$\mathsf{phash}_{h-1}$을 높이 $h-1$의 부모 블록 해시라 하자. 높이 $h$에서 계정 $a_i$는 다음을 계산한다:

$$(\beta_i, \pi_i) \leftarrow \mathrm{ECVRF.Prove}\!\left(sk_i,\; \texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h\right)$$

VCT는 32-byte VRF 출력 `beta` 전체를 big-endian unsigned 256-bit integer로 해석하고, 블록 헤더에 포함된 합의 임계값 `sortitionThreshold`와 비교한다. 다음 조건을 만족하면 계정 $a_i$는 높이 $h$에 대해 **기본(즉시) VRF 자격**을 얻는다:

$$\mathrm{uint256}(\beta_i) < \texttt{sortitionThreshold}_h$$

여기서 `sortitionThreshold`는 $0 < P_h \le 2^{256}$ 범위의 uint256 값이다. 즉 기본 즉시 자격 확률은 $P_h / 2^{256}$이다. `sortitionThreshold = 2^256`이면 모든 가능한 VRF 출력이 자격을 얻고, `sortitionThreshold = 2^253`이면 기본 즉시 자격 확률은 $1/8$이다.

### 블록 헤더 확장

VCTBlock 이후 블록 헤더는 기존 ECCPoW 헤더에 다음 필드를 추가한다:

- `vrfPublicKey`: coinbase 계정에 대응하는 compressed secp256k1 public key.
- `vrfProof`: `VCT_VRF || chainId || parentHash || blockNumber`에 대한 81-byte ECVRF proof.
- `vrfSignature`: nonce별 채굴 메시지에 대한 65-byte secp256k1 ECDSA signature.
- `sortitionThreshold`: 해당 블록의 기본 VRF 자격 임계값.

WIP-2 ECCPoW의 `codeword`와 `codeLength` 필드는 그대로 사용한다. 별도의 `vrfOutput` 필드는 두지 않으며, 검증자는 `vrfProof`에서 VRF output `beta`를 계산한다.

VCTBlock 이전 블록에서는 `vrfPublicKey`, `vrfProof`, `vrfSignature`, `sortitionThreshold`를 사용하지 않는다.

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

이 설계에서 ECCPoW 입력은 공개된 block template과 nonce만으로 구성되므로, VRF 자격 증명 $(\beta_i, \pi_i, PK_i)$가 공개된 이후 제3자가 동일한 입력으로 nonce 탐색을 대신 수행(offline delegated mining)할 수 있다.

ECCPoW 입력을 계정 개인키에 바인딩된 mining seed로 대체한다. 먼저 채굴자는 nonce loop 전에 block template commitment인 $\texttt{sealHash}$를 계산한다:

$$\texttt{sealHash} := \mathrm{Keccak256}\!\left(\texttt{VCT}\_\texttt{SEAL} \| \mathrm{chainId} \| \mathrm{RLP}(\mathsf{HeaderFields},\, \mathsf{VCTFields})\right)$$

VCTBlock 이후 `sealHash`는 기존 ECCPoW header commitment에 VCT 자격 증명 필드를 추가로 커밋한다. 추가되는 필드는 `vrfPublicKey`, `vrfProof`, `sortitionThreshold`이다. 반대로 nonce $\nu$, mining signature $\sigma_\nu$, Codeword, CodeLength는 ECCPoW 탐색 과정에서 결정되므로 `sealHash`에서 제외한다.

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

### 블록 생성 및 검증

VCTBlock 이후 블록 생성 절차는 기존 ECCPoW 채굴 루프 앞에 세 가지 단계를 추가한다. 첫째, 채굴 계정은 부모 상태에서 $S_0$ 이상의 잔액을 가져야 한다. 둘째, 부모 블록 해시와 목표 높이에 대해 ECVRF를 평가하고, VRF 출력이 현재 sortition 임계값 또는 점진적 타임아웃 임계값을 만족해야 한다. 셋째, 블록 헤더에는 VCT 자격 증명 필드 `vrfPublicKey`, `vrfProof`, `sortitionThreshold`가 포함되어야 한다.

이후 LDPC 기반 채굴 절차는 WIP-2 ECCPoW와 동일하다.

**계정 서명의 역할.** 논스별 채굴 서명 $\sigma_\nu$는 ECCPoW nonce 탐색 과정의 각 trial을 계정 개인 키 $sk_i$에 바인딩한다. 공개된 VRF 자격 증명 $(\beta_i, \pi_i, PK_i)$만으로는 제3자가 유효한 ECCPoW trial을 독립적으로 생성할 수 없으며, 각 nonce에는 $a_i$의 개인 키로 생성된 유효한 $\sigma_\nu$가 필요하다. 단, 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하는 경우까지 암호학적으로 차단하지는 않는다. VCT의 보안 주장은 공개된 VRF 자격 증명만으로 수행되는 offline delegated mining을 방지하는 것으로 제한된다.

VCTBlock 이후 수신 노드는 기존 ECCPoW 검증에 다음 검증을 추가한다. 먼저 블록에 포함된 공개키가 coinbase 주소와 일치하는지 확인하고, 부모 상태에서 해당 계정의 잔액이 $S_0$ 이상인지 확인한다. 다음으로 부모 블록 해시와 블록 높이에 대한 ECVRF 증명과 출력이 유효한지 검증하고, $\mathrm{uint256}(\beta_i) < P(\Delta t)$ 조건으로 sortition 자격을 판정한다. 마지막으로 고정된 `sealHash`에 대해 논스별 채굴 서명 $\sigma_\nu$가 coinbase 계정 키로 생성되었는지 확인하고, 이 서명을 포함한 `powSeed`로 기존 ECCPoW hash vector와 LDPC codeword 검증을 수행한다.

### Sortition 임계값 조정

VCTBlock 이후 블록 헤더는 `sortitionThreshold`를 포함한다. 목표 임계값은 `P_base = 2^253`($p^\star = 1/8$)이며, 하드포크 직후에는 `sortitionThreshold = 2^256`에서 시작하여 deterministic bootstrap controller가 목표값까지 낮춘다. 블록 시간이 지연되는 경우에는 [점진적 타임아웃](#점진적-타임아웃)의 시간 의존 임계값 `P(Δt)`가 해당 블록의 활성성을 보완한다.

여기서 `P_base`는 계속 움직이는 값이 아니라 `sortitionThreshold`가 수렴해야 하는 목표값이다. 평균 블록 생성 시간은 ECCPoW difficulty가 조정하고, VRF 임계값은 하드포크 직후 bootstrap 구간과 timeout 구간에서만 활성성 보완을 위해 조정된다.

자격 채굴자 수는 자연수이므로 sortition probability는 ECCPoW difficulty처럼 정밀한 난이도 대체물이 되기 어렵다. 따라서 `sortitionThreshold`는 admission-control 파라미터로 사용하고, 평균 블록 시간 조정은 ECCPoW difficulty에 맡긴다.

## Rationale

**잔액 기반 자격 조건.** 별도 스테이킹 컨트랙트나 검증자 레지스트리 없이 부모 블록 상태의 계정 잔액만으로 자격을 확인하는 방식을 선택한 이유는, 등록 트랜잭션·lock-up·슬래싱 메커니즘 없이 Sybil 공격 비용을 부과하면서 설계 복잡성을 최소화하기 위해서다. $S_0$는 지분 비례 영향력이 아니라 ECCPoW 참여의 최소 진입 비용으로 설계된다. 지분 크기에 비례한 선출 확률을 부여하면 자본 집중에 따른 중앙화 압력이 생기므로, VCT는 임계값을 충족한 모든 계정에 동일한 확률 $p$를 부여한다.

**계정 키 재사용.** 별도의 VRF 공개 키 등록 절차를 제거하고 기존 Ethereum 계정 키 쌍 $(sk_i, PK_i)$를 ECVRF 입력 키로 재사용한다. 이는 키 등록 트랜잭션, 계정-합의키 바인딩 검증, 별도 키 관리 오버헤드를 피하기 위해서이며, 구현과 사용자 경험을 단순화한다. 블록 헤더에 $PK_i$를 포함하고 $\mathrm{Address}(PK_i) = a_i$를 확인함으로써, 블록 생성자가 coinbase 주소에 대응하는 개인 키를 실제로 보유하고 있음을 VRF 증명과 채굴 서명이 일관되게 증명한다.

**논스별 채굴 서명.** 각 ECCPoW trial을 계정 개인 키에 바인딩하여 공개된 VRF 자격 증명만으로 수행 가능한 offline delegated mining을 방지한다. 공유된 block template과 nonce만으로 ECCPoW 입력이 결정된다면, VRF 자격을 얻은 계정 정보가 공개된 즉시 제3자가 동일 입력으로 nonce 탐색을 대신 수행할 수 있다. $\sigma_\nu$를 powSeed에 포함함으로써 유효한 ECCPoW hash-vector seed 생성에 $sk_i$ 보유가 필수가 된다. 단, 온라인 서명 서비스 형태의 위탁은 암호학적으로 차단되지 않는다는 점을 명시한다.

**점진적 타임아웃 대 고정 임계값.** 고정 임계값 `P_base`만 사용하면 $N$명의 채굴자가 모두 동시에 부적격일 수 있으며, 이 경우 체인이 영구 정지한다. 채굴자 수를 늘리면 이 확률이 지수적으로 감소하지만, 초기 네트워크에서는 $N$이 작아 위험이 높다. 점진적 타임아웃은 경과 시간 $\Delta t$에 따라 바이트 임계값 `P(Δt)`를 확대하여, 충분한 시간이 지나면 반드시 누군가가 블록을 생성할 수 있도록 활성성을 보장한다. 이 메커니즘이 고정 타임아웃(예: Δt ≥ T_hard이면 즉시 모든 채굴자 자격)보다 점진적 확대를 택한 이유는, 급격한 임계값 전환이 블록 스탬핑 공격이나 타임스탬프 조작 인센티브를 증가시키기 때문이다. 단, 타임스탬프 조작에 의한 미약한 그라인딩 여지는 명시적 한계로 남는다.

**VRF 임계값 조정 범위.** `sortitionThreshold` 조정은 ECCPoW difficulty의 대체물이 아니다. 자격 계정 수는 자연수이므로 임계값을 아주 미세하게 바꿔도 실제 자격자 수가 달라지지 않을 수 있고, 블록 시간은 이미 ECCPoW difficulty가 조정한다. 따라서 VRF 임계값 조정은 하드포크 직후 bootstrap과 점진적 타임아웃에 한정한다.

## Reference Implementation / 구현

참조 구현은 https://github.com/cryptoecc/WorldLand 의 WorldLand 클라이언트 합의 엔진(`consensus/vct/`)에 제공된다.

## References / 참고문헌

[1] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System,"
    2008.
[2] H. Park, S. Kim, and H.-N. Lee, "Time-Varying LDPC Code-Based
    Proof-of-Work for Cryptocurrency Mining," Symmetry, vol. 12,
    no. 6, 2020.
[3] H.-N. Lee et al., "Error Correction Code Verifiable Computation
    Consensus," IEEE, doc. 11048962, 2025.
[4] S. Micali, M. Rabin, and S. Vadhan, "Verifiable Random Functions,"
    in Proc. FOCS, 1999.
[5] J. Chen and S. Micali, "Algorand: A Secure and Efficient Distributed
    Ledger," Theor. Comput. Sci., vol. 777, pp. 155–183, 2019.
[6] D. Papadopoulos et al., "Making NSEC5 Practical for DNSSEC," 2017;
    IETF RFC 9381, "Verifiable Random Functions (VRFs)," 2023.
[7] S. Kim et al., "VRF-PoW: Proof of Work Consensus With Verifiable
    Random Function," IEEE Xplore, doc. 11362952.
[8] J. R. Douceur, "The Sybil Attack," in Proc. IPTPS, 2002.
[9] Ethereum Foundation, go-ethereum crypto/secp256k1 — libsecp256k1
    wrapper, https://github.com/ethereum/go-ethereum/tree/master/crypto/secp256k1.
[10] aergoio, secp256k1-vrf — secp256k1 VRF implementation,
     https://github.com/aergoio/secp256k1-vrf.

## Copyright / 저작권

Copyright and related rights released under the [GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.html).
