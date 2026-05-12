# WIP-6: Verifiable Coin Toss (VCT): 잔액 기반 VRF-ECCPoW 합의 프로토콜

<pre>
  Title: Verifiable Coin Toss (VCT): Balance-Gated VRF-ECCPoW Consensus
  Status: Draft
  Type: Core
  Author: Seungmin Kim <@smin-k>, Heungno Lee <@lincolnkerry>
  Created: 2026-05-07
  License: GNU Lesser General Public License v3.0
</pre>

## 목차
* [개요](#개요)
* [동기](#동기)
* [명세](#명세)
  - [참여 자격 조건](#참여-자격-조건)
  - [ECVRF 구조](#ecvrf-구조)
  - [VRF 키 쌍 사용 방식](#vrf-키-쌍-사용-방식)
  - [Verifiable Coin Toss 함수](#verifiable-coin-toss-함수)
  - [논스별 채굴 서명과 sealHash](#논스별-채굴-서명과-sealhash)
  - [블록 생성 및 검증](#블록-생성-및-검증)
  - [p 조정](#p-조정)
* [보안 분석](#보안-분석)
* [하위 호환성](#하위-호환성)
* [구현](#구현)
* [참고문헌](#참고문헌)
* [저작권](#저작권)

## 개요

이 문서는 WorldLand 블록체인을 위한 합의 메커니즘인 Verifiable Coin Toss (VCT) 프로토콜을 설명한다. VCT는 잔액 임계값 기반 채굴 자격 조건, secp256k1 기반 VRF 선발, 계정 바인딩 ECCPoW, Nakamoto 방식의 최대 유효 작업량 포크 선택을 결합한다.

각 블록 높이에서, 부모 상태 잔액이 $S_0$ 이상인 계정은 자신의 계정 키를 사용하여 부모 블록 해시와 목표 높이에 대해 secp256k1 기반 VRF를 평가할 수 있다. VRF 출력이 네트워크 자격 임계값 $p$ 미만이면 해당 계정은 ECCPoW를 시도할 자격을 얻는다. 생성된 블록에는 계정 공개 키, VRF 출력, VRF 증명, ECCPoW nonce, 그리고 해당 nonce에 대한 논스별 채굴 서명 $\sigma_\nu$가 포함되어야 한다. $\sigma_\nu$는 ECCPoW 탐색 중 각 nonce 시도마다 계정 개인 키로 생성되며, ECCPoW hash-vector seed에 포함된다. 따라서 공개된 VRF 자격 증명만으로는 제3자가 독립적으로 유효한 ECCPoW 입력을 생성할 수 없다. 단, 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하는 경우 위탁 채굴 자체를 암호학적으로 완전히 차단하지는 못한다.

VCT는 검증자 레지스트리나 지분 가중 리더 선출을 도입하지 않는다. 단일 계정에 $S_0$ 이상을 보유해도 해당 계정의 자격 확률은 증가하지 않는다. 여러 자격 시도를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

### WIP-6의 범위

WIP-6 v1.0은 WorldLand의 장기 로드맵 전체를 한 번에 구현하는 것을 목표로 하지 않는다. 이 문서의 목적은 다음 세 요소를 하나의 실행 가능한 합의 파이프라인으로 통합하는 것이다.

- 잔액 기반 PoS-style 자격 조건
- secp256k1 기반 VRF coin toss
- ECCPoW 채굴

따라서 WIP-6 v1.0은 DID 기반 신원 바인딩, TPM 기반 하드웨어 바인딩, GPU attestation, GPU marketplace, tiered participation model을 의도적으로 제외한다. 이 요소들은 장기 로드맵에 속하며, 첫 번째 VCT 합의 구현에는 포함하지 않는다.

## 동기

PoW(Proof-of-Work) 시스템은 탁월한 검열 저항성을 제공하지만, 모든 노드가 매 블록 경쟁하기 때문에 산업적 규모의 에너지를 소비한다. PoS(Proof-of-Stake) 시스템은 물리적 비용을 자본 가중 투표로 대체하는 대가로 극적인 에너지 절감을 달성한다.

VCT 프로토콜은 다른 위치를 점한다. 스테이킹 컨트랙트나 검증자 레지스트리를 도입하지 않고, 부모 상태의 계정 잔액만을 자격 기준으로 사용한다. VRF 자격 임계값 $p$가 각 블록에서 ECCPoW를 수행하는 계정 수를 제어하며, 최종 블록 경쟁은 여전히 자격을 갖춘 계정들 간의 ECCPoW 연산 경쟁으로 결정된다. $p$는 등록 계정 수가 아니라 실제 블록 생성 속도에 따라 Bitcoin의 난이도 조정과 유사한 방식으로 조정된다.

**초기 네트워크 보안과 잔액 기반 자격 조건.** VRF 기반 블록 선택 체계에서는 계정 수가 직접적인 보안 변수가 된다. 블록당 기대 자격 계정 수는 $p \cdot N$이므로, 참여 장벽이 없다면 공격자는 계정을 대량 생성하는 것만으로 자격 획득 확률을 임의로 높일 수 있다. 계정 생성 비용이 근본적으로 낮은 Ethereum 계정 모델에서 이는 Sybil 공격에 구조적으로 취약한 설계가 된다 [7, 8]. 특히 초기 네트워크 단계에서는 정직한 참여자 수가 적어 공격자의 상대적 계정 비율이 높아지기 쉽다.

잔액 임계값 $S_0$는 이 취약점을 완화하기 위한 ECCPoW 참여 사전 자격(prerequisite eligibility) 설계다. 각 자격 계정은 $S_0$ 이상의 잔액을 보유해야 하므로, 공격자가 $N_A$개의 자격 계정을 확보하려면 최소 $N_A \times S_0$의 자본을 소비해야 한다. 이는 PoS의 지분 가중 리더 선출과 구분된다. PoS에서는 지분 크기가 선출 확률에 직접 비례하지만, VCT에서는 임계값을 충족한 모든 계정이 동일한 확률 $p$로 VRF 자격 기회를 받는다. $S_0$는 지분에 비례한 영향력이 아니라 ECCPoW 참여를 위한 최소 진입 비용으로 설계된다.

VCT v1.0은 검증자 레지스트리, 위원회 선출, BFT-style 투표 라운드를 도입하지 않는다. 잔액 임계값을 만족한 계정은 로컬에서 VRF를 평가하고, VRF 자격을 얻은 계정만 ECCPoW 채굴 루프에 참여한다. 최종 블록 경쟁은 Nakamoto 방식으로 유지되며, 유효한 ECCPoW 작업량에 의해 결정된다.

## 명세

### 참여 자격 조건

$a_i$를 WorldLand 계정 주소, $PK_i$를 그 secp256k1 공개 키라 하면 $a_i = \mathrm{Address}(PK_i)$이다. 계정은 다음 조건을 만족할 때 블록 높이 $h$에 대해 채굴 자격을 갖춘다:

$$\mathrm{Balance}_{h-\ell}(a_i) \geq S_0$$

여기서 $S_0$는 프로토콜 파라미터로 정해진 잔액 임계값이고, $\ell \geq 1$은 잔액 참조 블록의 룩백 깊이(lookback depth)이다.

**기존 설계.** 자격 조건을 $\mathrm{Balance}_{h-1}(a_i) \geq S_0$으로 고정하면 부모 블록 상태를 직접 참조하므로 구현이 가장 단순하다. 그러나 $\ell$을 명시하지 않으면 향후 보안 요구에 따라 조정할 유연성이 없다.

**설계 옵션.** 잔액 기반 자격은 별도 등록 트랜잭션 없이 표준 계정 상태에서 직접 확인할 수 있다는 장점이 있다. staking 컨트랙트나 검증자 레지스트리 방식과 달리 온체인 등록 절차나 lock-up 메커니즘이 필요하지 않다. $\ell > 1$로 설정하면 현재 블록 생성자가 상태 조작을 통해 자격 계정 집합을 블록 경계에서 조작하기 어렵게 만들 수 있으나, 구현 복잡성이 증가한다.

**WIP-6 선택.** $\ell = 1$을 v1.0 기본값으로 채택한다. 즉, 블록 높이 $h$에 대한 자격은 부모 블록($h - 1$) 상태 잔액으로 판단한다. 별도 레지스트리나 등록 트랜잭션 없이 블록 검증 시 부모 상태에서 직접 확인된다.

**이유.** WIP-6 v1.0의 목표는 PoS 스타일 자격 조건, VRF 기반 코인 토스, ECCPoW 채굴을 하나의 실행 가능한 합의 파이프라인으로 통합하여 검증하는 것이다. 따라서 이 단계에서는 스테이킹 컨트랙트나 시스템 컨트랙트를 도입하지 않으며, 자격 확인은 계정 잔액만으로 수행한다. 향후 $\ell$ 증가나 장기 보유 조건 추가는 가능하지만, historical state lookup 비용 증가, 검증 노드의 추가 상태 보존 부담, 별도 eligibility-age 상태 관리 등의 비용이 따른다. 따라서 장기 보유 조건 또는 lock-up 메커니즘은 향후 WIP의 확장으로 남겨둔다.

각 잔액 조건을 충족한 계정은 블록당 하나의 VRF 자격 기회를 받는다. 동일 계정에 $S_0$ 이상을 보유해도 자격 확률이 증가하지 않는다. 여러 자격 기회를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

### ECVRF 구조

VCT는 계정별 블록 자격 증명을 위해 secp256k1 기반 ECVRF 구성을 사용한다. ECVRF는 ECDSA 서명과 별개의 암호 구조이다. ECDSA는 논스별 채굴 서명에만 사용되고, ECVRF는 특정 블록 높이에서 계정이 ECCPoW 참여 자격을 얻었는지를 증명하는 데 사용된다.

RFC 9381은 ECVRF의 일반 구조를 정의하지만, 표준 ciphersuite에는 secp256k1이 포함되어 있지 않다. 따라서 WorldLand는 hash-to-curve, point encoding, scalar encoding, challenge generation, nonce derivation, public-key validation, proof-to-hash 절차를 포함하는 WorldLand 전용 secp256k1 ECVRF ciphersuite를 합의 규칙으로 고정해야 한다. 이 ciphersuite를 합의 규칙으로 고정하지 않으면 서로 다른 클라이언트가 동일 입력에 대해 서로 다른 VRF 출력을 생성하여 합의를 위반할 수 있다.

이 문서에서 $\pi_i$는 선택된 secp256k1 ECVRF 구현이 생성하는 proof object를 의미한다. 정확한 proof encoding은 이 문서에서 임의로 재정의하지 않고, WorldLand secp256k1 ECVRF ciphersuite에서 고정한다. VCT는 deterministic ECDSA signature를 VRF proof로 재해석하지 않는다.

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

**secp256k1 기반 VRF.** Ethereum 계정 인프라는 이미 트랜잭션과 메시지 인증을 위해 secp256k1 계정 키를 사용하며, 일반적인 구현은 ECDSA signing에서 결정론적 nonce generation을 사용한다. 이는 구현 배경일 뿐이며, ECDSA 자체를 VRF로 사용한다는 의미는 아니다. WIP-6은 동일한 secp256k1 계정 키 쌍을 ECVRF 입력 키 쌍으로 재사용하여 별도 VRF 키 등록 절차를 제거한다. ECVRF와 ECDSA는 서로 다른 역할을 가진다. ECVRF는 블록 자격을 증명하고, ECDSA는 논스별 채굴 메시지를 서명한다. 구현 참조로 aergoio/secp256k1-vrf [10] 및 vechain/go-ecvrf [11]를 사용한다.

**이유.** 계정 키 재사용은 별도 VRF 공개 키 등록 절차 없이 기존 Ethereum 계정 인프라를 그대로 활용하며, DID 없는 v1.0 설계에서 구현과 사용자 경험을 단순화한다.

### Verifiable Coin Toss 함수

$\mathsf{phash}_{h-1}$을 높이 $h-1$의 부모 블록 해시라 하자. 높이 $h$에서 계정 $a_i$는 다음을 계산한다:

$$(y_i, \pi_i) \leftarrow \mathrm{ECVRF.Prove}\!\left(sk_i,\; \texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h\right)$$

다음 조건을 만족하면 계정 $a_i$는 높이 $h$에 대해 VRF 자격을 얻는다:

$$\frac{y_i}{2^{|y|}} < p_h$$

이 구조는 세 가지 핵심 특성을 가진다:

1. **지역성(Locality).** VRF는 $sk_i$와 공개된 $\mathsf{phash}_{h-1}$만으로 계정 $a_i$의 운영자 혼자 계산할 수 있다. 다른 노드와의 통신이 필요하지 않다.
2. **의사난수성(Pseudorandomness).** $sk_i$를 모르는 당사자에게 VRF 출력은 증명이 공개되기 전까지 난수와 계산적으로 구분할 수 없다. 입력에 $\mathsf{phash}_{h-1} \| h$를 포함함으로써 자격이 현재 체인 상태에 바인딩되고 유리한 입력을 자유롭게 선택하는 것이 제한된다. 단, 블록 헤더 그라인딩은 여전히 프로토콜 수준의 위험으로 남는다.
3. **검증 가능성(Verifiability).** VRF 증명 $\pi_i$는 $PK_i$와 $\mathsf{phash}_{h-1}$이 주어진 모든 검증 노드가 $sk_i$를 공개하지 않고도 자격 주장이 정직하게 생성되었음을 검증할 수 있게 한다.

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
1. Select the current canonical parent block at height h-1; let phash_{h-1} be its hash
2. Check that Balance_{h-ℓ}(a_i) ≥ S_0   (ℓ = 1 in VCT v1.0)
3. (y_i, π_i) ← ECVRF.Prove(sk_i, VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h)
4. if y_i / 2^|y| < p_h then              ▷ heads: a_i is VRF-eligible
5.     sealHash ← Keccak256(VCT_SEAL ∥ RLP(HeaderFields, VRFFields, ECCPoWParams))
       ▷ HeaderFields contains chainId; sealHash excludes ν, σ_ν, Codeword
6.     for each nonce ν = 0, 1, 2, … do
7.         m_ν ← Keccak256(VCT_MINE ∥ sealHash ∥ ν)
8.         σ_ν ← Sign_{sk_i}(m_ν)
9.         powSeed_ν ← Keccak256(VCT_ECCPOW ∥ sealHash ∥ ν ∥ σ_ν)
10.        s_1 ← Keccak512(powSeed_ν)  ▷ ECCPoW hash-vector seed
11.        Generate hash vector r from s_1 (extend with Keccak512 if n > 512)
12.        Run LDPC decoder D_mp(r, H) to obtain candidate codeword c
13.        if Hc = 0 and n/4 < W(c) < 3n/4 then
14.            Construct block h with (ν, Codeword=c, CodeLength=n, a_i, PK_i, y_i, π_i, σ_ν)
15.            Broadcast block h; break
16.        end if
17.    end for
18. else                                   ▷ tails: skip this block
19.    Wait for block h from peers
20. end if
```

**계정 서명의 역할.** 논스별 채굴 서명 $\sigma_\nu$는 ECCPoW nonce 탐색 과정에서 각 ECCPoW trial을 계정 개인 키 $sk_i$에 바인딩하기 위한 서명이다. 채굴자는 각 nonce $\nu$에 대해 $m_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$를 계산하고, $\sigma_\nu := \mathrm{Sign}_{sk_i}(m_\nu)$를 생성한 뒤, $\texttt{powSeed}_\nu := \mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu)$를 사용하여 ECCPoW hash vector $r$를 생성한다.

따라서 공개된 VRF 자격 증명 $(y_i, \pi_i, PK_i)$만으로는 제3자가 유효한 ECCPoW trial을 독립적으로 생성할 수 없다. 각 trial에는 $a_i$의 개인 키로 생성된 유효한 $\sigma_\nu$가 필요하기 때문이다. 단, 이 구조가 모든 형태의 위탁 채굴을 암호학적으로 불가능하게 만드는 것은 아니다. 계정 소유자가 개인키를 공유하거나 온라인 서명 서비스를 제공하면 외부 연산자가 서명된 nonce 후보를 받아 ECCPoW 계산을 수행할 수 있다. VCT의 보안 주장은 "공개된 VRF 자격 증명만으로 수행되는 offline delegated mining을 방지한다"로 제한된다.

수신 노드는 높이 $h$의 후보 블록을 다음 순서로 검증한다:
```
1.  Address(PK_i) = a_i
2.  Balance_{h-ℓ}(a_i) ≥ S_0   (ℓ = 1 in VCT v1.0)
3.  (y_i', valid) ← ECVRF.VerifyAndHash(PK_i,
        VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h, π_i)
4.  Require valid = 1  ▷ π_i가 PK_i와 메시지에 대해 유효한 증명인지 확인
5.  Require y_i' = y_i
6.  Require y_i' / 2^|y| < p_h
7.  Recompute sealHash ← Keccak256(VCT_SEAL ∥ RLP(HeaderFields, VRFFields, ECCPoWParams))
    ▷ HeaderFields contains chainId; sealHash excludes ν, σ_ν, Codeword
8.  m_ν ← Keccak256(VCT_MINE ∥ sealHash ∥ ν)
9.  Require ECRecover(σ_ν, m_ν) = a_i
    ▷ σ_ν의 서명자가 헤더에 명시된 블록 생성자 주소 $a_i$와 일치하는지 확인; valid secp256k1 ECDSA, low-S form
10. powSeed_ν ← Keccak256(VCT_ECCPOW ∥ sealHash ∥ ν ∥ σ_ν)
11. s_1 ← Keccak512(powSeed_ν)
12. Generate hash vector r from s_1
13. Regenerate parity-check matrix H from CodeLength and fixed degree parameters
14. Verify that LDPC decoder D_mp(r, H) produces codeword c = header.Codeword
15. Verify Hc = 0 and n/4 < W(c) < 3n/4
```
포크 선택은 누적 체인 작업량 기준 Nakamoto 최장 체인 규칙을 따른다. 블록 $B_h$의 작업량은 $W(B_h) = 1/T_h$이며, 체인 점수는 $W(\text{chain}) = \sum_h W(B_h)$이다. 위 15가지 검증 조건 중 하나라도 실패한 블록은 작업량에 기여하지 않는다. 자격 임계값 $p_h$는 채굴 참여 자격을 제어하는 역할을 하며, 체인 작업량 계산에는 포함되지 않는다.

### p 조정

$p$는 등록 계정 수에서 계산하지 않고, Bitcoin의 난이도 조정과 유사하게 실제 블록 생성 속도를 기반으로 조정된다.

$$p_{e+1} = \mathrm{clip}\!\left(p_e \cdot \frac{\Delta_{\mathrm{observed}}}{\Delta_{\mathrm{target}}},\; \frac{p_e}{r},\; p_e \cdot r\right)$$

여기서:

- $\Delta_{\mathrm{target}}$: 목표 블록 생성 간격
- $\Delta_{\mathrm{observed}}$: 직전 에포크에서 관측된 실제 평균 블록 생성 간격
- $r$: 한 에포크에서 $p$의 최대 변화 배율 (예: $r = 4$)

블록이 너무 자주 생성되면 $p$를 낮춰 참여자 수를 줄이고, 너무 느리게 생성되면 $p$를 높여 더 많은 계정이 ECCPoW를 시도하게 한다. 이 조정은 잔액 조건을 충족한 총 계정 수 $N$을 명시적으로 세지 않아도 작동한다.

에포크 길이, 초기값 $p_0$, 상하한, 타임스탬프 처리 규칙은 활성화 이전에 WorldLand 클라이언트 구현에서 고정되어야 한다.

## 보안 분석

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

VRF 입력이 canonical parent block hash를 포함하므로, VCT v1.0은 부모 해시 그라인딩을 완전히 제거하지 않는다. 블록 생성자는 트랜잭션 순서, 타임스탬프 선택, 블록 withholding 등을 통해 다음 VRF 입력에 제한적으로 영향을 줄 수 있다.

WIP-6은 이를 v1.0 설계의 명시적 한계로 둔다. 이 버전의 목적은 최소한의 balance-gated VRF-ECCPoW 파이프라인을 구현하는 것이므로, VDF나 외부 randomness beacon은 도입하지 않는다.

### 위탁 채굴 저항성

VCT의 계정 바인딩 ECCPoW는 공개된 VRF 자격 증명만을 이용한 독립적 위탁 채굴을 방지한다. 일반적인 ECCPoW 입력이 공개 block template과 nonce만으로 구성된다면, VRF 자격을 얻은 계정의 정보가 공개된 이후 제3자가 동일한 자격 정보를 사용하여 nonce 탐색을 대신 수행할 수 있다.

VCT에서는 각 ECCPoW trial이 다음 값에 의존한다:

$$\texttt{powSeed}_\nu = \mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu),$$

여기서

$$\sigma_\nu = \mathrm{Sign}_{sk_i}(\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)).$$

따라서 $sk_i$를 보유하지 않은 외부자는 유효한 $\sigma_\nu$를 생성할 수 없고, 결과적으로 유효한 ECCPoW hash-vector seed도 생성할 수 없다. 이는 VRF proof가 공개된 이후에도 제3자가 공개 데이터만으로 ECCPoW nonce search를 독립적으로 수행하는 것을 막는다.

그러나 이 저항성은 완전한 mining-pool 방지를 의미하지 않는다. 계정 소유자가 개인키를 직접 공유하거나, nonce별 서명을 제공하는 온라인 signing service를 운영한다면 외부 worker가 ECCPoW 계산을 수행할 수 있다. VCT는 이러한 online delegation을 암호학적으로 제거하지 않는다. 대신 각 ECCPoW trial에 계정 서명 능력을 요구함으로써 공개 자격 증명만으로 가능한 offline delegation을 차단하고, 위탁 채굴의 운영 비용과 키 관리 위험을 증가시킨다.

### 키 보안

계정 개인 키 $sk_i$가 ECVRF 계산과 논스별 채굴 서명 모두에 사용되며, 각 ECCPoW trial마다 서명이 요구되므로 핫 키가 채굴 노드에 상주해야 한다. 이는 키 노출 위험을 증가시키며, 채굴 노드 운영 시 적절한 키 관리 절차가 필요하다.

## 하위 호환성

WIP-6은 하드 포크를 통해 활성화되며, 이전 클라이언트와 호환되지 않는 프로토콜 변경을 포함한다.

**프로토콜 수준 비호환성.** VCTBlock 이후에는 다음 규칙이 기존 규칙을 대체한다.

- **논스별 채굴 서명 공식 변경.** $m_\nu$ 계산에서 `chainId`가 제거된다. `chainId`는 이미 `sealHash` 내부의 `HeaderFields`에 포함되어 있으므로, VCTBlock 이전의 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{chainId} \| \texttt{sealHash} \| \nu)$ 공식은 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$로 단순화된다.

- **ECCPoW seed 공식 변경.** 동일한 이유로 $\texttt{powSeed}_\nu$ 계산에서도 `chainId`가 제거된다.

- **VRF 입력 메시지 변경.** VRF 입력이 epoch 기반 sortition seed hash에서 $\texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h$로 변경된다. 이 변경으로 VRF 자격이 특정 chain, 특정 부모 블록, 특정 높이에 명시적으로 바인딩된다.

- **주소 검증 추가.** 블록 헤더의 $PK_i$로부터 유도한 주소가 $a_i$와 일치하는지 ($\mathrm{Address}(PK_i) = a_i$) 검증하는 단계가 추가된다.

**업그레이드 경로.** VCTBlock 이전까지 모든 노드는 기존 ECCPoW 검증 규칙을 따른다. VCTBlock 활성화 이후, WIP-6을 구현하지 않은 노드는 새 규칙으로 생성된 블록을 거부하여 네트워크에서 분리된다. 충분히 먼 미래의 블록 번호를 VCTBlock으로 설정하여 네트워크 참여자가 클라이언트를 업그레이드할 시간을 확보한다.

**채굴자 요구 사항.** VCTBlock 활성화 이후 채굴자는 각 ECCPoW trial마다 $\sigma_\nu$ 생성에 계정 개인 키가 필요하므로, 개인 키를 채굴 노드에 직접 제공해야 한다.

## 구현

VCT 프로토콜은 https://github.com/cryptoecc/WorldLand 에서 관리되는 go-ethereum의 EVM 호환 포크인 WorldLand 클라이언트의 합의 엔진으로 구현될 예정이다.

WIP-6은 기존 ECCPoW 클라이언트 대비 다음 다섯 가지 변경을 요구한다.

1. **잔액 기반 자격 확인** — 블록 검증 시 $\mathrm{Balance}_{h-\ell}(a_i) \geq S_0$를 직접 확인한다($\ell = 1$, v1.0 기본값). 별도 레지스트리나 등록 트랜잭션이 필요 없다.

2. **secp256k1 기반 ECVRF 통합** — 클라이언트는 secp256k1 ECVRF 구현을 통합하고, 채굴 루프에서 블록 높이마다 $\mathrm{ECVRF.Prove}(sk_i,\, \texttt{VCT}\_\texttt{VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h)$를 한 번 평가한다. 사용할 secp256k1 ECVRF ciphersuite는 합의 규칙으로 고정되어야 한다.

3. **계정 키 재사용 및 공개 키-주소 바인딩** — 블록 헤더에 $a_i$와 $PK_i$를 포함한다. 검증자는 $\mathrm{Address}(PK_i) = a_i$를 확인한 후, 동일한 $PK_i$로 VRF 증명과 논스별 채굴 서명을 각각 검증한다.

4. **논스별 채굴 서명 및 ECCPoW seed 교체** — 각 nonce $\nu$마다 $m_\nu \leftarrow \mathrm{Keccak256}(\texttt{VCT}\_\texttt{MINE} \| \texttt{sealHash} \| \nu)$를 계산하고 $\sigma_\nu \leftarrow \mathrm{Sign}_{sk_i}(m_\nu)$를 생성한다. 기존 WIP-2의 $\mathrm{Keccak256}(\mathrm{CBH}) \| \nu$ 기반 hash-vector seed를 $\mathrm{Keccak256}(\texttt{VCT}\_\texttt{ECCPOW} \| \texttt{sealHash} \| \nu \| \sigma_\nu)$로 대체한다. LDPC decoder와 codeword 유효성 조건은 WIP-2와 동일하게 유지한다.

5. **sealHash 및 블록 검증 갱신** — 블록 헤더에 $PK_i$, $y_i$, $\pi_i$, $\nu$, Codeword, CodeLength, $\sigma_\nu$ 필드를 추가한다. 클라이언트는 고정된 HeaderFields에서 `sealHash`를 재계산하고, $\mathrm{ECRecover}(\sigma_\nu, m_\nu) = a_i$를 확인한 뒤, ECCPoW hash vector를 재생성하여 codeword를 검증한다. 검증 실패 블록은 체인 작업량에 기여하지 않는다.

## 참고문헌

[1] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008.  
[2] H. Park, S. Kim, and H.-N. Lee, "Time-Varying LDPC Code-Based Proof-of-Work for Cryptocurrency Mining," Symmetry, vol. 12, no. 6, 2020.  
[3] H.-N. Lee et al., "Error Correction Code Verifiable Computation Consensus," IEEE, doc. 11048962, 2025.  
[4] S. Micali, M. Rabin, and S. Vadhan, "Verifiable Random Functions," in Proc. FOCS, 1999.  
[5] J. Chen and S. Micali, "Algorand: A Secure and Efficient Distributed Ledger," Theor. Comput. Sci., vol. 777, pp. 155–183, 2019.  
[6] D. Papadopoulos et al., "Making NSEC5 Practical for DNSSEC," 2017; IETF RFC 9381, "Verifiable Random Functions (VRFs)," 2023.  
[7] S. Kim et al., "VRF-PoW: Proof of Work Consensus With Verifiable Random Function," IEEE Xplore, doc. 11362952.  
[8] J. R. Douceur, "The Sybil Attack," in Proc. IPTPS, 2002.  
[9] Ethereum Foundation, go-ethereum crypto/secp256k1 — libsecp256k1 wrapper, https://github.com/ethereum/go-ethereum/tree/master/crypto/secp256k1.  
[10] aergoio, secp256k1-vrf — secp256k1 ECVRF implementation, https://github.com/aergoio/secp256k1-vrf.  
[11] vechain, go-ecvrf — Go ECVRF library, https://github.com/vechain/go-ecvrf.  

## 저작권

Copyright and related rights released under the [GNU Lesser General Public License v3.0](https://www.gnu.org/licenses/lgpl-3.0.html).
