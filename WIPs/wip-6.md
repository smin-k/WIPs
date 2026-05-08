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
  - [시스템 모델](#시스템-모델)
  - [ECVRF 구조](#ecvrf-구조)
  - [Verifiable Coin Toss 함수](#verifiable-coin-toss-함수)
  - [블록 생성 및 검증](#블록-생성-및-검증)
  - [p 조정](#p-조정)
* [보안 분석](#보안-분석)
* [구현](#구현)
* [참고문헌](#참고문헌)

## 개요

이 문서는 WorldLand 블록체인을 위한 합의 메커니즘인 Verifiable Coin Toss (VCT) 프로토콜을 설명한다. VCT는 잔액 임계값 기반 채굴 자격 조건, secp256k1 기반 VRF 선발, 계정 바인딩 ECCPoW, Nakamoto 방식의 최대 유효 작업량 포크 선택을 결합한다.

각 블록 높이에서, 부모 상태 잔액이 $S_0$ 이상인 계정은 자신의 계정 키를 사용하여 부모 블록 해시와 목표 높이에 대해 secp256k1 기반 VRF를 평가할 수 있다. VRF 출력이 네트워크 자격 임계값 $p$ 미만이면 해당 계정은 ECCPoW를 시도할 자격을 얻는다. 생성된 블록에는 계정 공개 키, VRF 출력, VRF 증명, ECCPoW nonce, 최종 블록 커밋먼트에 대한 계정 서명이 포함되어야 한다.

VCT는 검증자 레지스트리나 지분 가중 리더 선출을 도입하지 않는다. 단일 계정에 $S_0$ 이상을 보유해도 해당 계정의 자격 확률은 증가하지 않는다. 여러 자격 시도를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

합의 메커니즘의 변경은 핵심 프로토콜을 수정하므로 블록체인의 하드 포크를 유발한다. 업데이트된 프로토콜을 따르는 노드는 이전 프로토콜을 따르는 노드가 채굴한 블록을 더 이상 수락하지 않으며, 그 역도 마찬가지다.

## 동기

PoW(Proof-of-Work) 시스템은 탁월한 검열 저항성을 제공하지만, 모든 노드가 매 블록 경쟁하기 때문에 산업적 규모의 에너지를 소비한다. PoS(Proof-of-Stake) 시스템은 물리적 비용을 자본 가중 투표로 대체하는 대가로 극적인 에너지 절감을 달성한다.

VCT 프로토콜은 다른 위치를 점한다. 스테이킹 컨트랙트나 검증자 레지스트리를 도입하지 않고, 부모 상태의 계정 잔액만을 자격 기준으로 사용한다. VRF 자격 임계값 $p$가 각 블록에서 ECCPoW를 수행하는 계정 수를 제어하며, 최종 블록 경쟁은 여전히 자격을 갖춘 계정들 간의 ECCPoW 연산 경쟁으로 결정된다. $p$는 등록 계정 수가 아니라 실제 블록 생성 속도에 따라 Bitcoin의 난이도 조정과 유사한 방식으로 조정된다.

**초기 네트워크 보안과 잔액 기반 자격 조건.** VRF 기반 블록 선택 체계에서는 계정 수가 직접적인 보안 변수가 된다. 블록당 기대 자격 계정 수는 $p \cdot N$이므로, 참여 장벽이 없다면 공격자는 계정을 대량 생성하는 것만으로 자격 획득 확률을 임의로 높일 수 있다. 계정 생성 비용이 근본적으로 낮은 Ethereum 계정 모델에서 이는 Sybil 공격에 구조적으로 취약한 설계가 된다 [7, 8]. 특히 초기 네트워크 단계에서는 정직한 참여자 수가 적어 공격자의 상대적 계정 비율이 높아지기 쉽다.

잔액 임계값 $S_0$는 이 취약점을 완화하기 위한 ECCPoW 참여 사전 자격(prerequisite eligibility) 설계다. 각 자격 계정은 $S_0$ 이상의 잔액을 보유해야 하므로, 공격자가 $N_A$개의 자격 계정을 확보하려면 최소 $N_A \times S_0$의 자본을 소비해야 한다. 이는 PoS의 지분 가중 리더 선출과 구분된다. PoS에서는 지분 크기가 선출 확률에 직접 비례하지만, VCT에서는 임계값을 충족한 모든 계정이 동일한 확률 $p$로 VRF 자격 기회를 받는다. $S_0$는 지분에 비례한 영향력이 아니라 ECCPoW 참여를 위한 최소 진입 비용으로 설계된다.

위원회도, 비잔틴 합의 프로토콜도, 투표 라운드도 없다. 시스템은 Bitcoin의 단순성을 유지하면서 활성 채굴 참여자 수를 줄이고 잔액 임계값 자격 조건을 도입한다.

## 명세

### 시스템 모델

$a_i$를 WorldLand 계정 주소, $PK_i$를 그 secp256k1 공개 키라 하면 $a_i = \mathrm{Address}(PK_i)$이다. 계정은 부모 상태에서 다음 조건을 만족할 때 블록 높이 $h$에 대해 자격을 갖춘다:

$$\mathrm{Balance}_{h-1}(a_i) \geq S_0$$

자격 조건은 블록 검증 시 부모 상태에서 직접 확인된다.

각 잔액 조건을 충족한 계정은 블록당 하나의 VRF 자격 기회를 받는다. 동일 계정에 $S_0$ 이상을 보유해도 자격 확률이 증가하지 않는다. 여러 자격 기회를 원하는 참여자는 각각 잔액 임계값을 충족하는 여러 계정을 유지해야 한다.

계정의 secp256k1 키 쌍 $(sk_i, PK_i)$이 VRF 키로도 사용되어, 별도의 VRF 키 등록 없이 계정 식별과 VRF 자격 증명이 통합된다.

### ECVRF 구조

VCT는 RFC 9381의 ECVRF 구조를 따르는 secp256k1 기반 ECVRF instantiation을 사용한다. ECVRF는 ECDSA 서명 알고리즘과는 별개의 암호 구조로, Schnorr 방식의 비대화형 영지식 증명을 기반으로 한다. 단, RFC 9381의 표준 ciphersuite는 P-256 및 ED25519 곡선만을 정의하며 secp256k1은 포함되어 있지 않다. 따라서 VCT는 RFC 9381의 구조적 틀을 따르되, hash-to-curve, point encoding, challenge generation, nonce derivation, proof-to-hash 절차를 WorldLand 전용 ciphersuite로 명시해야 한다. 이 ciphersuite를 합의 규칙으로 고정하지 않으면 서로 다른 클라이언트가 동일 입력에 대해 서로 다른 VRF 출력을 생성하여 합의를 위반할 수 있다.

비밀 키 $sk_i$와 메시지 $m$이 주어졌을 때, ECVRF.Prove는 먼저 메시지를 타원 곡선 점으로 해시한 뒤 비밀 키를 곱하여 VRF 점을 생성한다:

$$\Gamma_i = sk_i \cdot H_{\text{curve}}(m)$$

여기서 $H_{\text{curve}}$는 메시지를 secp256k1 곡선 위의 점으로 해시하는 함수(hash-to-curve)이다. VRF 출력 $y_i$는 이 점의 해시값이다:

$$y_i = \mathrm{Hash}(\Gamma_i)$$

VRF 증명 $\pi_i = (c, s)$는 Schnorr 방식의 비대화형 증명으로, $\Gamma_i$가 비밀 키 $sk_i$에 의해 정직하게 생성되었음을 보증한다. 검증자는 $(PK_i,\, m,\, y_i,\, \pi_i)$가 주어지면 $sk_i$를 공개하지 않고도 다음 세 성질을 확인할 수 있다:

1. **유일성(Uniqueness):** $(sk_i, m)$ 쌍에 대해 유효한 VRF 출력은 정확히 하나 존재한다. 동일한 입력에 대해 서로 다른 $y_i$를 주장하는 것은 ECDLP를 푸는 것과 동등하다.
2. **의사난수성(Pseudorandomness):** $sk_i$를 모르는 당사자에게 $y_i$는 증명 $\pi_i$가 공개되기 전까지 계산적으로 균등한 난수와 구분할 수 없다.
3. **검증 가능성(Verifiability):** $PK_i$가 있는 누구나 $\pi_i$로부터 $y_i$가 $sk_i$에 의해 정직하게 생성되었음을 검증할 수 있다.

ECVRF 증명 $\pi_i$는 Schnorr 방식의 구조체이며, ECDSA 서명 $(r, s)$와는 다른 형태이다. ECDSA의 결정론적 nonce(RFC 6979)는 ECDSA 서명의 재현성을 보장하는 기법이지만 VRF를 구성하지 않는다. ECVRF는 독립적인 VRF 구조로, 동일한 secp256k1 키 쌍을 사용하되 별개의 알고리즘 구조를 따른다.

**Ethereum 계정 인프라와의 관계.** WorldLand는 EVM 호환 체인이므로 계정 모델은 Ethereum의 외부 소유 계정(EOA) 구조를 따른다. Ethereum 계정은 secp256k1 개인 키로 트랜잭션과 메시지를 서명하며, 검증자는 서명으로부터 공개 키 또는 주소를 복원하여 계정 소유자의 승인을 확인한다. go-ethereum에 포함된 secp256k1 라이브러리는 ECDSA signing/verification과 RFC 6979 기반 nonce generation을 지원한다 [9]. 이는 Ethereum 계정 생태계가 이미 secp256k1 기반 계정 인증과 결정론적 서명 구현 경험을 갖고 있음을 의미한다.

그러나 RFC 6979 기반 ECDSA는 그 자체로 VRF가 아니다. 정직한 서명자가 동일한 개인 키와 메시지에 대해 재현 가능한 서명을 생성할 수 있더라도, 검증자가 서명자가 올바른 절차를 따랐는지 공개적으로 증명받는 구조가 없다. 즉, VRF가 요구하는 공개 검증 가능한 유일성과 의사난수성은 ECDSA만으로는 보장되지 않는다.

VCT의 핵심은 Ethereum의 ECDSA 서명을 VRF로 재해석하는 것이 아니라, 기존 secp256k1 계정 키 쌍을 그대로 ECVRF 키로 활용하는 것이다. 역할은 명확히 분리된다. ECVRF는 특정 블록 높이에서 해당 계정이 ECCPoW 자격을 얻었는지를 증명하는 데 사용되고, ECDSA는 최종 블록 커밋먼트에 대한 계정 인증에 사용된다. 별도의 VRF 전용 키 등록 없이 계정 식별과 VRF 자격 증명이 통합된다.

### Verifiable Coin Toss 함수

$\mathsf{phash}_{h-1}$을 높이 $h-1$의 부모 블록 해시라 하자. 높이 $h$에서 계정 $a_i$는 다음을 계산한다:

$$(y_i, \pi_i) \leftarrow \mathrm{ECVRF.Prove}\!\left(sk_i,\; \texttt{VCT\_VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h\right)$$

다음 조건을 만족하면 계정 $a_i$는 높이 $h$에 대해 VRF 자격을 얻는다:

$$\frac{y_i}{2^{|y|}} < p_h$$

이 구조는 세 가지 핵심 특성을 가진다:

1. **지역성(Locality).** VRF는 $sk_i$와 공개된 $\mathsf{phash}_{h-1}$만으로 계정 $a_i$의 운영자 혼자 계산할 수 있다. 다른 노드와의 통신이 필요하지 않다.
2. **의사난수성(Pseudorandomness).** $sk_i$를 모르는 당사자에게 VRF 출력은 증명이 공개되기 전까지 난수와 계산적으로 구분할 수 없다. 입력에 $\mathsf{phash}_{h-1} \| h$를 포함함으로써 자격이 현재 체인 상태에 바인딩되고 유리한 입력을 자유롭게 선택하는 것이 제한된다. 단, 블록 헤더 그라인딩은 여전히 프로토콜 수준의 위험으로 남는다.
3. **검증 가능성(Verifiability).** VRF 증명 $\pi_i$는 $PK_i$와 $\mathsf{phash}_{h-1}$이 주어진 모든 검증 노드가 $sk_i$를 공개하지 않고도 자격 주장이 정직하게 생성되었음을 검증할 수 있게 한다.

### 블록 생성 및 검증

계정 $a_i$의 운영자는 다음 채굴 루프를 실행한다:

```
1. Select the current canonical parent block at height h-1; let phash_{h-1} be its hash
2. Check that Balance_{h-1}(a_i) ≥ S_0
3. (y_i, π_i) ← ECVRF.Prove(sk_i, VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h)
4. if y_i / 2^|y| < p_h then              ▷ heads: a_i is VRF-eligible
5.     Begin ECCPoW search over account-bound challenge:
           input = (phash_{h-1}, h, a_i, PK_i, y_i, π_i)
6.     if nonce ν found such that ECCPoW(input, ν, T_h) = valid then
7.         σ_i ← Sign_{sk_i}(VCT_BLOCK ∥ chainId
                              ∥ h ∥ phash_{h-1} ∥ timestamp
                              ∥ stateRoot ∥ txRoot ∥ receiptRoot
                              ∥ gasLimit ∥ gasUsed
                              ∥ a_i ∥ PK_i ∥ y_i ∥ π_i ∥ ν)
8.         Construct block h with fields (ν, a_i, PK_i, y_i, π_i, σ_i)
9.         Broadcast block h
10.    end if
11. else                                   ▷ tails: skip this block
12.    Wait for block h from peers
13. end if
```

**계정 서명의 역할.** 계정 서명 $\sigma_i$는 VCT 블록 생성 과정에서 단순한 인증 수단을 넘어 핵심 보안 요소다. $\sigma_i$는 ECCPoW 탐색을 완료한 직후, 블록 높이·부모 해시·상태 루트·트랜잭션 루트·영수증 루트·가스 필드·타임스탬프·VRF 출력·VRF 증명·ECCPoW nonce를 포함한 전체 블록 헤더 커밋먼트에 대해 계정 개인 키 $sk_i$로 생성된다. 이로써 유효한 블록을 생성하려면 ECCPoW 탐색을 수행한 그 계정의 개인 키로 직접 서명해야 한다.

이 설계는 두 가지 공격을 차단한다. 첫째, 후보 블록 또는 부분적으로 전파된 블록 생성 정보를 복사하더라도, 공격자는 원 계정의 개인 키 없이 다른 블록 커밋먼트에 대한 유효한 $\sigma_i$를 생성할 수 없다. 둘째, 탈취한 VRF 자격 증명과 유효한 ECCPoW nonce $\nu$만으로 블록을 위조하는 자격 증명 재사용 공격을 막는다. $sk_i$에 대한 접근 없이는 올바른 $\sigma_i$를 생성할 수 없다.

결과적으로 VCT의 작업증명 과정은 ECCPoW 연산 완료 후 해당 계정의 개인 키 서명까지 포함해야 비로소 유효한 블록이 완성된다. ECCPoW가 연산 비용을 통해 자격을 증명한다면, $\sigma_i$는 블록 내용 전체에 대한 계정 소유권을 최종 확정한다.

수신 노드는 높이 $h$의 후보 블록을 다음 순서로 검증한다:
```
1. Address(PK_i) = a_i
2. Balance_{h-1}(a_i) ≥ S_0
3. (y_i', valid) ← ECVRF.VerifyAndHash(PK_i,
       VCT_VRF ∥ chainId ∥ phash_{h-1} ∥ h, π_i)
4. Require valid = 1  ▷ π_i가 PK_i와 메시지에 대해 유효한 증명인지 확인
5. Require y_i' = y_i  ▷ 블록에 기재된 y_i가 π_i로부터 재계산된 값과 일치하는지 확인
6. Require y_i' / 2^|y| < p_h  ▷ VRF 자격 임계값 통과 여부
7. ECCPoW(ν, phash_{h-1} ∥ h ∥ a_i ∥ PK_i ∥ y_i ∥ π_i, T_h) = valid
8. σ_i가 a_i의 sk_i 하에서 전체 블록 헤더 커밋먼트에 대해 유효함
```
포크 선택은 누적 체인 작업량 기준 Nakamoto 최장 체인 규칙을 따른다. 블록 $B_h$의 작업량은 $W(B_h) = 1/T_h$이며, 체인 점수는 $W(\text{chain}) = \sum_h W(B_h)$이다. 위 6가지 검증 조건 중 하나라도 실패한 블록은 작업량에 기여하지 않는다. 자격 임계값 $p_h$는 채굴 참여 자격을 제어하는 역할을 하며, 체인 작업량 계산에는 포함되지 않는다.

### p 조정

$p$는 등록 계정 수에서 계산하지 않고, Bitcoin의 난이도 조정과 유사하게 실제 블록 생성 속도를 기반으로 조정된다.

$$p_{e+1} = \mathrm{clip}\!\left(p_e \cdot \frac{\Delta_{\mathrm{observed}}}{\Delta_{\mathrm{target}}},\; \frac{p_e}{r},\; p_e \cdot r\right)$$

여기서:

- $\Delta_{\mathrm{target}}$: 목표 블록 생성 간격
- $\Delta_{\mathrm{observed}}$: 직전 에포크에서 관측된 실제 평균 블록 생성 간격
- $r$: 한 에포크에서 $p$의 최대 변화 배율 (예: $r = 4$)

블록이 너무 자주 생성되면 $p$를 낮춰 참여자 수를 줄이고, 너무 느리게 생성되면 $p$를 높여 더 많은 계정이 ECCPoW를 시도하게 한다. 이 조정은 잔액 조건을 충족한 총 계정 수 $N$을 명시적으로 세지 않아도 작동한다.

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

현재 블록 생성자는 트랜잭션 선택, 순서 변경, 또는 채굴 타이밍 조작을 통해 서로 다른 $\mathsf{phash}_{h-1}$ 후보를 생성하고, 어느 값이 다음 블록에서 자신의 계정에 유리한 VRF 출력을 만드는지 사전에 확인할 수 있다. 이를 부모 해시 그라인딩이라 한다.

ECCPoW 챌린지에 계정 바인딩을 도입함으로써 VRF 자격 증명의 전이적 재사용은 방지하였으나, 부모 해시 선택 자체에서 발생하는 그라인딩은 VCT v1에서 완전히 제거되지 않는다. 공격자가 $G$개의 부모 해시 후보를 탐색할 수 있다면, 특정 공격자 계정이 다음 높이에서 적어도 한 번 VRF 자격을 얻을 확률은:

$$1 - (1-p_h)^G$$

로 증가한다. 따라서 그라인딩 편향은 $p_h$만이 아니라 블록 생성자가 조작 가능한 헤더 필드, 트랜잭션 순서, 타임스탬프 선택 공간($G$)에 의해 결정된다. 완전한 해결을 위해서는 VDF(Verifiable Delay Function) 기반 랜덤 비콘과 같은 추가 메커니즘이 필요하며, 이는 향후 WIP에서 다룰 예정이다.

### 키 보안

계정 개인 키 $sk_i$가 ECVRF 계산과 블록 서명 모두에 사용되므로 핫 키 노출 위험이 존재한다. 채굴 노드 운영 시 적절한 키 관리 절차가 필요하다.

## 구현

VCT 프로토콜은 https://github.com/cryptoecc/WorldLand 에서 관리되는 go-ethereum의 EVM 호환 포크인 WorldLand 클라이언트의 합의 엔진으로 구현될 예정이다.

현재 ECCPoW 구현 대비 필요한 변경 사항은 다음과 같다:

1. **secp256k1 ECVRF ciphersuite 정의** — hash-to-curve 방식, point encoding, scalar encoding, challenge hash, nonce derivation, proof-to-hash, public-key validation 절차를 합의 규칙으로 고정; 클라이언트 간 VRF 출력 불일치 방지
2. **잔액 자격 확인** — 블록 검증 시 부모 상태에서 $\mathrm{Balance}_{h-1}(a_i) \geq S_0$ 직접 확인; 별도 레지스트리 불필요
3. **secp256k1-VRF 통합** — 계정 개인 키를 사용하는 ECVRF 구현; 채굴 루프에서 $\texttt{VCT\_VRF} \| \texttt{chainId} \| \mathsf{phash}_{h-1} \| h$에 대한 블록당 VRF 평가 및 증명 생성
4. **계정 바인딩 ECCPoW 챌린지** — ECCPoW 입력에 $\mathsf{phash}_{h-1}$, $h$, 계정 주소 $a_i$, 공개 키 $PK_i$, VRF 출력 $y_i$, VRF 증명 $\pi_i$ 포함
5. **블록 헤더 확장** — 계정 주소 $a_i$, 공개 키 $PK_i$, VRF 출력 $y_i$, VRF 증명 $\pi_i$, ECCPoW nonce $\nu$, 계정 서명 $\sigma_i$를 위한 추가 필드
6. **검증 로직** — 잔액 확인, 공개 키-주소 검증, VRF 증명 검증(proof-derived $y_i'$ 재계산 및 블록 기재값과 비교), 자격 임계값 확인, 계정 바인딩 ECCPoW 검증, 전체 헤더 커밋먼트에 대한 계정 서명 검증의 8단계 순차 확인; 검증 실패 블록은 체인 작업량에 기여하지 않음
7. **p 조정** — 에포크 단위로 관측된 블록 생성 간격 기반 $p$ 업데이트

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
