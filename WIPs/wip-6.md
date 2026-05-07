# WIP-6: Verifiable Coin Toss (VCT): Balance-Gated VRF-ECCPoW Consensus

<pre>
  Title: Verifiable Coin Toss (VCT): Balance-Gated VRF-ECCPoW Consensus
  Status: Draft
  Type: Core
  Author: Heungno Lee <@lincolnkerry>
  Created: 2026-05-07
  License: GNU Lesser General Public License v3.0
</pre>

## Table of Contents
* [Abstract](#abstract)
* [Motivation](#motivation)
* [Specification](#specification)
  - [System Model](#system-model)
  - [Verifiable Coin Toss Function](#verifiable-coin-toss-function)
  - [Block Production and Validation](#block-production-and-validation)
  - [Dynamic p Adjustment](#dynamic-p-adjustment)
* [Security Analysis](#security-analysis)
* [Implementation](#implementation)
* [References](#references)

## Abstract

This document describes the Verifiable Coin Toss (VCT) protocol, a consensus mechanism for the WorldLand blockchain that combines balance-threshold account eligibility, Verifiable Random Function (VRF)-based mining selection, account signature binding, and ECCPoW-based block production.

In every block, each balance-qualified account independently evaluates a VRF over the previous block hash to decide whether it earns the right to attempt ECCPoW. With a network-tunable acceptance probability $p$, only an expected $pN = T$ of $N$ eligible accounts become eligible to mine per block, reducing the expected active ECCPoW population from $N$ to $T$ while preserving a Nakamoto-style heaviest-valid-work fork-choice rule. A valid block must also include an account-level signature binding the ECCPoW solution to the balance-qualified account, preventing unauthorized reuse of a copied VRF eligibility proof.

VCT is not stake-weighted PoS. Holding more than the threshold balance $S_0$ in a single account does not increase that account's eligibility probability. Instead, $S_0$ acts as a minimum economic eligibility condition. Each qualified account receives exactly one VRF eligibility trial per block.

Changing the consensus mechanism modifies the core protocol, thus triggering a hard fork in the blockchain. After this update is made, nodes following the updated protocol will no longer accept blocks mined by nodes following the deprecated protocol, and vice versa.

## Motivation

Proof-of-Work (PoW) systems provide unparalleled censorship resistance but consume energy at industrial scale, as all nodes compete in every block. Proof-of-Stake (PoS) systems achieve dramatic energy reduction at the cost of replacing physical-world cost with capital-weighted voting.

The VCT protocol occupies a different position. The threshold balance $S_0$ provides a lightweight economic eligibility condition without requiring funds to be locked in a staking contract. VRF eligibility reduces the number of accounts that perform ECCPoW in each block; the final block-winning step remains a computational race among eligible accounts. For example, if $T$ is set to roughly one percent of the eligible account count, the expected active mining population is reduced by two orders of magnitude.

Unlike a fixed-deposit staking design, the balance-gated design does not require accounts to lock funds in a native validator registry. Eligibility is determined from the account balance at the epoch snapshot. There is no committee, no Byzantine agreement protocol, and no voting round. The system maintains the simplicity of Bitcoin while reducing the active mining population by the factor $p$ and introducing a balance-threshold eligibility condition.

## Specification

### System Model

Let $\mathcal{A}$ denote the set of VRF-registered accounts. Each account $i \in \mathcal{A}$ is controlled by an operator who maintains a long-term VRF key pair $(sk_i^{\mathsf{vrf}}, pk_i^{\mathsf{vrf}})$ and a standard account signing key for the account address $a_i$. The operator binds $pk_i^{\mathsf{vrf}}$ to $a_i$ through a VRF-key registration transaction. Registration does not lock funds.

At each epoch boundary, the protocol checks whether each registered account satisfies the balance threshold:

$$\mathcal{A}_e = \left\{ a_i \in \mathcal{A} : \mathrm{Balance}_e(a_i) \geq S_0 \right\}$$

where $\mathrm{Balance}_e(a_i)$ is the account balance at the epoch snapshot. The set $\mathcal{A}_e$ forms the active eligibility set for epoch $e$.

Each balance-qualified account receives exactly one VRF eligibility trial per block during the epoch. Holding more than $S_0$ in the same account does not increase its eligibility probability. A participant who wants multiple eligibility trials must maintain multiple VRF-registered accounts, each with balance at least $S_0$ at the epoch snapshot.

The protocol maintains a network-tunable parameter $p_e \in (0, 1)$, computed at each epoch boundary from $|\mathcal{A}_e|$ and fixed for the duration of that epoch.

### Verifiable Coin Toss Function

Let $\mathsf{phash}_{h-1}$ denote the block hash of the parent block at height $h-1$. For each account $a_i \in \mathcal{A}_e$, the operator computes:

$$(y_i, \pi_i) \leftarrow \mathrm{VRF}(sk_i^{\mathsf{vrf}},\, \mathsf{phash}_{h-1} \| h)$$

Account $a_i$ is VRF-eligible for height $h$ if:

$$\frac{y_i}{2^{|y|}} < p_e$$

The construction has three essential properties:

1. **Locality.** The VRF is computable by the operator of account $a_i$ alone, requiring only $sk_i^{\mathsf{vrf}}$ and the public $\mathsf{phash}_{h-1}$. No interaction with other nodes is needed.
2. **Pseudorandomness.** To any party not knowing $sk_i^{\mathsf{vrf}}$, the VRF output is computationally indistinguishable from random before the proof is revealed. The input $\mathsf{phash}_{h-1} \| h$ binds eligibility to the current chain state and limits the freedom to choose a favorable VRF input, although block-header grinding remains a protocol-level risk and must be further limited by the header format and fork-choice rule.
3. **Verifiability.** The accompanying VRF proof $\pi_i$ allows any validating node, given $pk_i^{\mathsf{vrf}}$ and $\mathsf{phash}_{h-1}$, to verify that the eligibility claim was generated honestly without revealing $sk_i^{\mathsf{vrf}}$.

### Block Production and Validation

The operator controlling validator account $a_i$ executes the following mining loop:

```
1. Select the current canonical parent block at height h-1; let phash_{h-1} be its hash
2. Check that a_i ∈ A_e and Balance(a_i) ≥ S_0 in the parent state
3. (y_i, π_i) ← VRF(sk_i^vrf, phash_{h-1} ∥ h)
4. if y_i / 2^|y| < p_e then              ▷ heads: a_i is VRF-eligible
5.     Begin ECCPoW search over account-bound challenge:
           input = (phash_{h-1}, h, a_i, pk_i^vrf, y_i, π_i)
6.     if nonce ν found such that ECCPoW(input, ν, T_h) = valid then
7.         σ_i ← Sign_{a_i}(VCT_BLOCK ∥ chainId ∥ h ∥ phash_{h-1}
                             ∥ a_i ∥ pk_i^vrf ∥ y_i ∥ π_i ∥ ν ∥ txRoot)
8.         Construct block h with fields (ν, a_i, pk_i^vrf, y_i, π_i, σ_i)
9.         Broadcast block h
10.    end if
11. else                                   ▷ tails: skip this block
12.    Wait for block h from peers
13. end if
```

The nonce $\nu$ is included in the signed message so that a copied VRF eligibility proof cannot be reused by another miner to produce a valid block without access to the account signing key of $a_i$.

The value of $p_e$ is taken from the epoch snapshot and is fixed for all blocks in epoch $e$. All nodes apply the same deterministic value of $p_e$ when validating eligibility.

A receiving node validates a candidate block at height $h$ by checking, in order:
1. Account $a_i$ is included in the epoch eligibility snapshot $\mathcal{A}_e$
2. Account $a_i$ has balance $\geq S_0$ in the parent state of the candidate block
3. $pk_i^{\mathsf{vrf}}$ is bound to $a_i$ in the VRF-key registry
4. $\mathrm{VerifyVRF}(pk_i^{\mathsf{vrf}},\, \mathsf{phash}_{h-1} \| h,\, y_i,\, \pi_i) = 1$
5. $y_i / 2^{|y|} < p_e$
6. The ECCPoW solution $\nu$ is valid for the account-bound challenge $(\mathsf{phash}_{h-1},\, h,\, a_i,\, pk_i^{\mathsf{vrf}},\, y_i,\, \pi_i)$ at difficulty $T_h$
7. The account signature $\sigma_i$ is valid under $a_i$ over the block commitment including $\nu$

Fork choice follows the heaviest-valid-ECCPoW-chain rule, where a block contributes work only if all seven validation conditions above are satisfied.

### Dynamic p Adjustment

At each epoch boundary, the protocol computes the active eligibility set $\mathcal{A}_e$ from the epoch balance snapshot and sets:

$$p_e \leftarrow \min\!\left(1,\, \frac{T}{|\mathcal{A}_e|}\right)$$

The set $\mathcal{A}_e$ and the value $p_e$ remain fixed throughout epoch $e$. Accounts that newly satisfy the balance threshold during an epoch are included in $\mathcal{A}_e$ only at the next epoch boundary. Accounts whose balance falls below $S_0$ after the snapshot remain eligible for the current epoch under the snapshot rule; immediate-disqualification rules are left for future consideration.

Since $p_e$ is chosen so that $\mathbb{E}[X] = p_e |\mathcal{A}_e| = T$, the probability that no account is eligible in a given block is:

$$\Pr[X = 0] = (1 - p_e)^{|\mathcal{A}_e|} \approx e^{-T}$$

For target values such as $T \in [10^2, 10^3]$, this probability is negligible.

The use of $|\mathcal{A}_e|$ assumes that balance-qualified accounts are economically motivated to remain online and exploit their eligibility. If many eligible accounts are offline or intentionally inactive, the effective number of active mining participants may fall below $T$. The protocol may therefore require reward rules that discourage idle eligible accounts.

## Security Analysis

Let $N_H$ and $N_A$ denote the number of active balance-qualified accounts controlled by the honest network and the adversary, respectively, in the epoch snapshot $\mathcal{A}_e$. Each active account has balance at least $S_0$, has a registered VRF public key, and receives one VRF eligibility trial per block with probability $p_e$.

At block height $h$, the number of eligible honest and adversarial accounts is:

$$X_H \sim \text{Binomial}(N_H, p_e), \qquad X_A \sim \text{Binomial}(N_A, p_e)$$

Conditioned on $(X_H, X_A) = (x_H, x_A)$, the probability that the adversary wins the ECCPoW race is:

$$\Pr[A \text{ wins} \mid x_H, x_A] = \frac{h_A x_A}{h_A x_A + h_H x_H}$$

where $h_A$ and $h_H$ denote the effective ECCPoW throughput per eligible adversarial and honest account. For large $N_H$, $N_A$, the binomial distributions concentrate around their means and $p_e$ cancels:

$$\Pr[A \text{ wins}] \approx \frac{h_A N_A}{h_A N_A + h_H N_H}$$

Under ECCPoW's ASIC resistance ($h_A \approx h_H$), this simplifies to:

$$\Pr[A \text{ wins}] \approx \frac{N_A}{N_A + N_H}$$

VCT is not stake-proportional. Holding more than $S_0$ in one account does not increase that account's eligibility probability. The block-winning probability is driven by the number of balance-qualified accounts and ECCPoW throughput, not by total balance held in a single account.

A sustained majority attack requires the adversary to dominate the eligible ECCPoW capacity:

$$h_A N_A > h_H N_H$$

Registered account count alone is not sufficient for block dominance unless the adversary also has enough ECCPoW throughput to exploit its eligible accounts. Conversely, a hardware advantage increases $h_A$ and lowers the number of balance-qualified accounts required for majority control. If $h_A \approx h_H$, this reduces to $N_A > N_H$. Since each additional adversarial account must maintain balance at least $S_0$, the liquid balance requirement for increasing $N_A$ scales linearly with the number of adversarial accounts.

**Limitations.** The balance-gated design does not provide hardware-level Sybil resistance and does not lock funds. A participant may create multiple eligible accounts by maintaining balance at least $S_0$ in each. VCT should therefore not be interpreted as one-CPU-one-vote consensus. Its Sybil resistance comes from the liquid balance requirement per eligible account, while its energy reduction comes from VRF-gated participation.

The account-level signature prevents unauthorized reuse of a copied VRF eligibility proof, because a valid block must be signed by the balance-qualified account over a commitment that includes the ECCPoW nonce. However, this mechanism does not prevent voluntary delegation: if the account owner shares its private key or provides a remote signing service to another miner, the protocol cannot distinguish this from normal operation.

## Implementation

The VCT protocol is to be implemented as a consensus engine within the WorldLand client, an EVM-compatible fork of go-ethereum maintained at https://github.com/cryptoecc/WorldLand.

The required changes relative to the current ECCPoW implementation are:

1. **VRF key registration** — protocol-level binding between account address $a_i$ and VRF public key $pk_i^{\mathsf{vrf}}$ via a registration transaction
2. **Epoch balance snapshot** — epoch-wise computation of the active eligibility set $\mathcal{A}_e = \{a_i : \mathrm{Balance}_e(a_i) \geq S_0\}$ and the corresponding $p_e = \min(1,\, T / |\mathcal{A}_e|)$
3. **VRF integration** — per-block VRF evaluation over $\mathsf{phash}_{h-1} \| h$ and proof generation in the mining loop
4. **Account-bound ECCPoW challenge** — ECCPoW input includes $\mathsf{phash}_{h-1}$, $h$, account address $a_i$, registered VRF public key, VRF output, and VRF proof
5. **Block header extension** — additional fields for account address $a_i$, VRF public key $pk_i^{\mathsf{vrf}}$, VRF output $y_i$, VRF proof $\pi_i$, ECCPoW nonce $\nu$, and account signature $\sigma_i$
6. **Validation logic** — balance-threshold check, VRF-key registry lookup, VRF proof verification, eligibility check, ECCPoW validation, and account signature verification; blocks failing any check do not contribute to chain work

## References

[1] S. Nakamoto, "Bitcoin: A Peer-to-Peer Electronic Cash System," 2008.  
[2] H. Park, S. Kim, and H.-N. Lee, "Time-Varying LDPC Code-Based Proof-of-Work for Cryptocurrency Mining," Symmetry, vol. 12, no. 6, 2020.  
[3] H.-N. Lee et al., "Error Correction Code Verifiable Computation Consensus," IEEE, doc. 11048962, 2025.  
[4] S. Micali, M. Rabin, and S. Vadhan, "Verifiable Random Functions," in Proc. FOCS, 1999.  
[5] J. Chen and S. Micali, "Algorand: A Secure and Efficient Distributed Ledger," Theor. Comput. Sci., vol. 777, pp. 155–183, 2019.  
