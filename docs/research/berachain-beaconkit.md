# BeaconKit (Berachain) — EVM Chain Implementation Research

**Project:** `berachain/beacon-kit` — "A modular framework for building EVM consensus clients"
**Maintainer:** Berachain / Berachain Foundation
**Language:** Go · **License:** BUSL-1.1
**Latest stable release:** v1.4.1 (2026-06-23)
**Last verified:** 2026-06-24

---

## Executive Summary

BeaconKit is Berachain's Go-based consensus-client framework that wraps the **CometBFT** BFT consensus engine around a **standard, unmodified Ethereum execution client** connected over the **standard Ethereum Engine API** — exactly the two-client (consensus layer + execution layer) split that Ethereum itself adopted post-Merge. This architecture is the heart of its design philosophy: by delegating all transaction execution to an off-the-shelf EL (Geth, Reth, Erigon, Nethermind, Besu, etc.), BeaconKit aims to be **"EVM-identical,"** meaning the EVM behaves bit-for-bit like Ethereum mainnet and all existing Ethereum tooling works unchanged. This was a deliberate reversal of Berachain's earlier **Polaris** design, which embedded the EVM inside a Cosmos SDK app and relied on custom (stateful) precompiles — an approach Berachain abandoned precisely because it diverged from Ethereum and created an execution bottleneck. The trade-off is the crux of this evaluation: BeaconKit gains near-perfect Ethereum compatibility (criterion 1) at the explicit cost of dropping custom precompiles (criterion 2). Berachain instead exposes its native Proof-of-Liquidity logic as **ordinary Solidity system contracts** on the EVM, not precompiles. It is production-grade: it has powered Berachain mainnet since 6 Feb 2025, tracks Ethereum hardforks closely (Bectra ≈ Pectra live June 2025; Fulu/Fusaka in v1.4.1, June 2026), and ships on a roughly monthly cadence.

---

## Fit Against Requirements

| # | Criterion | Verdict | One-line rationale |
|---|-----------|---------|--------------------|
| 1 | 1:1 compatibility with latest Ethereum mainnet / all tooling works | **Strong** | Pairs with unmodified EL via Engine API → "EVM-identical"; tracks Ethereum hardforks (Pectra/Fusaka). |
| 2 | Add Turing-incomplete modules exposed as EVM precompiles | **Weak** | Custom precompiles were *deliberately removed*; native logic ships as Solidity system contracts, not precompiles. |
| 3 | Consensus: BFT, DPoS, PoS, PoA | **Partial** | Strong BFT (CometBFT) + a bespoke PoS variant (Proof-of-Liquidity). DPoS/PoA not first-class configurable knobs. |
| 4 | Actively maintained | **Strong** | v1.4.1 (2026-06-23), ~monthly cadence, backed by Berachain Foundation, drives a live L1 mainnet. |

### Criterion 1 — Ethereum compatibility: **Strong**
The two-client architecture means execution is handled by a standard Ethereum client. Berachain describes itself as an **"EVM-identical"** L1, and its docs state it is **"100% EVM-compatible, operating on a lightly modified Reth."** It keeps pace with Ethereum upgrades on a ~6-month cadence: **Bectra** (Berachain's port of Ethereum's **Pectra / Prague-Electra**) went live on mainnet **4 June 2025**, making Berachain the first EVM-identical L1 to ship Pectra execution-layer features (e.g. EIP-7702 smart accounts) after Ethereum itself. The latest **v1.4.1 (June 2026)** is the "mainnet-ready Fusaka (Fulu)" release. Because the EVM is genuinely standard, the full Ethereum tooling stack (Hardhat, Foundry, ethers/viem, wallets, indexers, JSON-RPC) works without modification. Caveat: it ships only the *execution-layer* parts of each fork; consensus-layer EIPs are intentionally omitted because consensus is CometBFT, not Ethereum's beacon chain.

### Criterion 2 — Turing-incomplete modules via precompiles: **Weak**
This is the central tension. BeaconKit's defining decision was to **remove the Polaris-style custom precompiles** to achieve EVM-identical behavior. Polaris (Berachain V1) bundled the EVM into a Cosmos SDK app and used stateful precompiles to expose native modules — but this diverged from canonical Ethereum and tightly coupled consensus and execution. BeaconKit reverses that: by running an *unmodified* EL, you cannot inject custom precompiles without forking the EL and thereby breaking the "EVM-identical" guarantee. Berachain's own native functionality — **Proof-of-Liquidity** (BGT, BeraChef, BlockRewardController/Distributor, Reward Vaults, BeaconDeposit) — is therefore implemented as **ordinary Solidity system contracts deployed on the EVM**, with published contract addresses, *not* as precompiles. The only EL-level customization Berachain itself makes is a small "lightly modified Reth" (Bera-Reth) for protocol mechanics like the mandatory per-block EVM-inflation withdrawal and deposit handling — a fork they accept for themselves but which runs against the spirit of "unmodified EL." Net: if your requirement is *custom precompiles*, BeaconKit explicitly does not offer them; it offers system contracts as the sanctioned alternative.

### Criterion 3 — Consensus (BFT / DPoS / PoS / PoA): **Partial**
- **BFT:** Strong. CometBFT (Tendermint) provides classic BFT with **single-slot / instant finality** (blocks final once 2/3+ validators commit), versus Ethereum's ~13-minute finality.
- **PoS:** Yes, but bespoke. Berachain runs **Proof-of-Liquidity (PoL)**, a heavily modified PoS where validators stake **BERA** to enter the active set and block-production frequency is BERA-weighted, while emissions (WBERA/BGT) are directed to reward vaults. Active set was 69 validators in V1, raised toward a soft cap (~256) with an activation bond in V2.
- **DPoS / PoA:** Not first-class. BeaconKit is a *framework* (modular block builders, DA, etc.), and CometBFT can in principle be configured for permissioned/PoA-style validator sets, but Berachain ships neither a documented PoA mode nor a delegated-PoS variant as a supported configuration. Treat DPoS/PoA as "achievable by forking the framework," not turnkey.

### Criterion 4 — Active maintenance: **Strong**
Maintained by Berachain / the Berachain Foundation as the consensus layer of a live, top-tier L1. Recent releases: v1.4.1 (2026-06-23), v1.4.0-rc3 (2026-05-12), v1.3.9 (2026-04-21), v1.3.8 (2026-04-07), v1.3.7 (2026-03-11), v1.3.6 (2026-01-27), v1.3.5 (2026-01-06) — a roughly monthly cadence with security patches in between. It powers Berachain mainnet (live since 6 Feb 2025) and tracks Ethereum's roadmap on a ~6-month upgrade rhythm.

---

## Architecture

BeaconKit implements the post-Merge Ethereum two-client model:

```
            ┌──────────────────────────┐         ┌────────────────────────────┐
            │  BeaconKit (Consensus)   │         │  Execution Layer (EL)       │
            │  - CometBFT (BFT)        │ Engine  │  Geth / Reth / Erigon /     │
            │  - Cosmos-SDK-derived    │◄──API──►│  Nethermind / Besu          │
            │    middleware            │  (JWT)  │  (standard, EVM-identical)  │
            │  - Beacon/staking logic  │         │  Berachain runs "Bera-Reth" │
            └──────────────────────────┘         └────────────────────────────┘
```

- **Consensus client (BeaconKit):** Wraps **CometBFT** with a modular middleware layer. Built in **Go**, drawing on the **Cosmos SDK / CometBFT** ecosystem (Berachain's lineage is Cosmos-based), but BeaconKit deliberately *decouples* execution from the SDK app — execution lives entirely in the EL.
- **Execution client (EL):** A **standard Ethereum execution client**. The Engine API lets any compliant EL plug in (Geth, Reth, Erigon, Nethermind, Besu, ethereumjs are commonly cited). Berachain's own production client is **Bera-Reth**, a lightly modified Reth fork that adds Berachain-specific mechanics (notably the mandatory first-withdrawal **EVM inflation** mechanism and deposit processing).
- **Engine API:** Standard Ethereum JSON-RPC Engine API over JWT — the same interface Ethereum CL/EL pairs use. This is what makes the EL swappable and the EVM identical.
- **Notable performance features:** Single-slot finality; **optimistic payload building** (claimed to reduce block times by up to ~40%); modular block builders / DA layers.

## Ethereum Compatibility & Hardfork Tracking

- Positioned as **"EVM-identical"** — strictly stronger marketing than "EVM-compatible": the goal is bytecode-for-bytecode identical EVM behavior so no contract rewrites are needed and all tooling works as-is.
- **Hardfork timeline:** Cancun-era support at launch → **Bectra** (= Ethereum **Pectra / Prague-Electra**, including EIP-7702 smart accounts, validator-exit / deposit EIPs) live **4 June 2025** → **Fulu / Fusaka** EL features in **v1.4.1 (June 2026)**.
- **Cadence:** ~6-month protocol upgrades, mirroring Ethereum's EL forks. Consensus-layer EIPs from each Ethereum fork are intentionally excluded (consensus is CometBFT/PoL, not the beacon chain).
- **Tooling implication:** Hardhat, Foundry, viem/ethers, MetaMask, block explorers, subgraph/indexers, and standard JSON-RPC all work because the EL is genuine Ethereum software.

## Precompiles vs System Contracts (the key trade-off)

- **Polaris (V1):** EVM embedded in a Cosmos SDK app; native modules exposed via **custom stateful precompiles** (e.g. the historical "Berachef" precompile). Tightly coupled consensus+execution; diverged from canonical Ethereum; hit an execution bottleneck.
- **BeaconKit (V2):** Custom precompiles **removed** to stay EVM-identical. You cannot add custom precompiles without forking the EL and forfeiting the identical-EVM guarantee.
- **How native functionality is exposed now:** As **regular Solidity smart/system contracts** on the EVM with documented addresses — BGT (soulbound governance token), BeraChef (reward allocation/vault whitelisting), BlockRewardController/Distributor, Reward Vaults, BeaconDeposit. These are normal contracts callable by any tooling, not precompiles.
- **Implication for a "Turing-incomplete precompile module" requirement:** Not supported in the intended sense. The sanctioned path is system contracts (full EVM, Turing-complete Solidity) or — if you truly need precompiles — forking the EL (Bera-Reth itself is such a fork), which contradicts criterion 1.

## Consensus, Staking & Proof-of-Liquidity

- **Engine:** CometBFT BFT, instant/single-slot finality, timeout-based rounds with proposer rotation on failure.
- **Validators:** Stake **BERA** to join the active set; block-production probability is BERA-weighted. Active set ~69 (V1) → soft cap ~256 (V2) with an activation bond.
- **Proof-of-Liquidity:** Berachain's modified PoS. Block emissions (WBERA, and **BGT** governance/reward token) are not paid directly to validators; validators *direct* emissions to whitelisted Reward Vaults, aligning consensus rewards with on-chain liquidity/economic activity. Tri-token model historically described as **BERA** (gas/stake), **BGT** (soulbound governance/rewards), **HONEY** (stablecoin); V2 docs also reference WBERA/sWBERA emission mechanics.
- **DPoS/PoA:** Not provided as supported, configurable consensus variants.

## Maintenance & Production Status

- **Repo:** github.com/berachain/beacon-kit · **Language:** Go · **License:** BUSL-1.1.
- **Latest stable:** v1.4.1 (2026-06-23, "mainnet-ready Fusaka/Fulu"). Recent line: v1.3.5 → v1.3.9 (Jan–Apr 2026, including CometBFT security/DoS fixes), v1.4.0-rc3 (May 2026), v1.4.1 (Jun 2026).
- **Cadence:** ~monthly releases; ~6-month protocol upgrades tracking Ethereum.
- **Production:** Berachain mainnet live since **6 Feb 2025**; Bectra (Pectra) upgrade **4 June 2025**. Maintained by Berachain / Berachain Foundation.

## Pros / Cons Summary

**Pros**
- Genuinely EVM-identical via unmodified-EL + Engine API → best-in-class Ethereum tooling compatibility (criterion 1).
- Instant finality (CometBFT) vs Ethereum's probabilistic ~13-min finality.
- Multiple EL choices (Geth/Reth/Erigon/Nethermind/Besu) → client diversity and resilience.
- Actively maintained, production-proven on a major L1, close Ethereum-fork tracking.

**Cons**
- **No custom precompiles** — the explicit cost of EVM-identity (criterion 2 fails); native logic must be Solidity system contracts.
- Berachain's own production EL is a Reth *fork* (Bera-Reth) for EVM-inflation/deposit mechanics — so "unmodified EL" is the framework's ideal more than Berachain's literal practice.
- DPoS/PoA not turnkey; consensus is opinionated (CometBFT + PoL).
- BUSL-1.1 (source-available, not OSI open source) may constrain some commercial reuse.
- Consensus-layer Ethereum EIPs are intentionally absent (acceptable given CometBFT, but means it is *not* identical at the CL).

---

## Sources

- [GitHub — berachain/beacon-kit](https://github.com/berachain/beacon-kit)
- [beacon-kit README (main)](https://github.com/berachain/beacon-kit/blob/main/README.md)
- [beacon-kit Releases](https://github.com/berachain/beacon-kit/releases)
- [GitHub API — beacon-kit releases (dates/tags)](https://api.github.com/repos/berachain/beacon-kit/releases)
- [What is BeaconKit? — Berachain Docs](https://docs.berachain.com/learn/what-is-beaconkit)
- [BeaconKit Consensus Layer — Berachain Docs](https://docs.berachain.com/nodes/beaconkit-consensus)
- [EVM Execution Layer — Berachain Docs](https://docs.berachain.com/nodes/evm-execution)
- [Proof-of-Liquidity Overview — Berachain Docs](https://docs.berachain.com/learn/pol/)
- [Berachain V2: explanation of the changes (Berachain Blog)](https://blog.berachain.com/blog/berachain-v2-an-explanation-of-the-changes-how-it-affects-pol-dynamics-and-why-its-an-important-next-step)
- [Berachain's V2 Upgrade Goes Live (blocmates)](https://www.blocmates.com/news-posts/berachain-s-v2-upgrade-goes-live)
- [Berachain Taps Ethereum's Pectra Playbook With 'Bectra' (CoinDesk)](https://www.coindesk.com/tech/2025/06/04/berachain-taps-ethereums-pectra-playbook-with-bectra-upgrade)
- [Bectra Is Live (Berachain Blog)](https://blog.berachain.com/blog/bectra-is-live-programmable-wallets-on-demand-exits-and-more-on-berachain)
- [Berachain Technical Deep Dive (Simply Staking)](https://simplystaking.com/berachain-technical-deep-dive)
- [Berachain Beyond the Basics: Proof of Liquidity (Figment)](https://figment.io/insights/berachain-beyond-the-basics-exploring-proof-of-liquidity-consensus/)
- [What Is Berachain and Proof of Liquidity (CoinGecko)](https://www.coingecko.com/learn/what-is-berachain-crypto-proof-of-liquidity)
- [Berachain mainnet launch Feb 6 2025 (OKX)](https://www.okx.com/en-us/learn/what-is-bera-chain-mainnet)
- [Polaris Architecture (legacy docs)](https://polaris.berachain.dev/docs/architecture)

---

*Last verified: 2026-06-24*
