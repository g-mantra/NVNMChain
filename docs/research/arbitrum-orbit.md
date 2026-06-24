# Arbitrum Orbit & Arbitrum Nitro — Full-Stack EVM Chain Implementation Research

**Last verified: 2026-06-24**

## Executive Summary

Arbitrum Nitro is Offchain Labs' production framework for building EVM-compatible Layer-2 / Layer-3 chains; **Arbitrum Orbit** (now generally branded "Arbitrum chains") is the program/tooling that lets anyone deploy a custom chain on the Nitro stack that settles to Ethereum, Arbitrum One/Nova, or another parent chain. Nitro achieves near-perfect Ethereum equivalence through a "Geth sandwich" architecture — a lightly-modified fork of go-ethereum (geth) does EVM execution at the core, with **ArbOS** (Go) wrapped around it to handle L2 concerns (sequencing inputs, cross-chain messaging, the dual L2/L1 gas model, and fee collection). It tracks Ethereum hardforks closely: as of June 2026 the live release on Arbitrum One is **ArbOS 51 "Dia"** (activated 8 Jan 2026), which implements the EVM-relevant parts of Ethereum's **Fusaka** upgrade. Nitro is extensible via two paths: a fixed set of **ArbOS precompiles** (ArbSys, ArbGasInfo, ArbOwner, ArbWasm, etc.) exposed as Solidity-callable contracts at reserved `0x64…` addresses, and **Stylus**, a coequal WASM VM (Rust/C/C++) that runs alongside the EVM. Chain operators *can* add custom Go precompiles, but with significant caveats. Critically for a chain-framework comparison: **Orbit is a rollup/AnyTrust framework, not a sovereign-L1 BFT/PoS/PoA validator framework.** Ordering is done by a (typically single, Offchain-Labs-or-RaaS-operated) **sequencer**; security comes from **fraud proofs (now the permissionless BoLD protocol) settling disputes on Ethereum L1**, or from a **Data Availability Committee (DAC)** in AnyTrust mode. There is no validator consensus (BFT/DPoS/PoA) over transaction ordering. The stack is very actively maintained (latest Nitro **v3.11.0**, 22 Jun 2026) under a **Business Source License 1.1** that converts to Apache 2.0 on 31 Dec 2030.

---

## Fit Against Requirements

### 1. 1:1 compatibility with the latest Ethereum mainnet — **STRONG (with documented L2 deviations)**
Nitro uses a fork of go-ethereum for EVM execution, so opcode/precompile/state-trie behavior and JSON-RPC are Ethereum-equivalent and standard tooling (Hardhat, Foundry, ethers/viem, MetaMask, The Graph, block explorers) works out of the box. It tracks mainnet hardforks within months: ArbOS 51 "Dia" brings the EVM-relevant **Fusaka** changes (Fusaka finalized on mainnet 3 Dec 2025). The deviations are inherent to being an L2 rollup, not equivalence bugs: a **dual gas model** (L2 execution gas + L1 data/posting fee), different block timing/structure (sequencer-produced blocks, batched & Brotli-compressed posting to L1 via EIP-4844 blobs), L2-specific tx types, address aliasing for L1→L2 calls, and the extra ArbOS precompiles. Stylus adds a *second* VM (WASM) but does not change EVM semantics. Verdict: as close to "latest Ethereum mainnet" as an L2 realistically gets — Strong, but it is intentionally an L2-flavored EVM, not byte-identical to an Ethereum execution client.

### 2. Add Turing-incomplete modules exposed as EVM contracts via precompiles — **STRONG**
ArbOS ships a rich set of native precompiles exposed as ordinary Solidity-callable contracts at fixed addresses (`0x64`–`0x73`, plus `0xff`), e.g. ArbSys, ArbGasInfo, ArbOwner, ArbAddressTable, ArbRetryableTx, ArbWasm. Chain operators **can add custom precompiles**: written in Go in Nitro's `/precompiles` directory with a matching Solidity interface, given a reserved (per EIP-7587), non-conflicting address. This is exactly the "native module exposed as an EVM contract" capability the criterion asks for. Caveat that lowers confidence slightly: the simplest supported pattern is read-only (`eth_call`/view) precompiles — adding non-view methods or calling custom precompiles from other contracts requires care because it affects deterministic block validation and fraud-proving, and Offchain Labs recommends going through a RaaS provider and ArbOS-upgrade procedures. Stylus is an additional (Turing-complete) extensibility path. Verdict: Strong — precompiles are first-class and customizable, with operational/security caveats.

### 3. Consensus support (BFT / DPoS / PoS / PoA) — **WEAK / NOT APPLICABLE (different security model)**
This is the most important honest caveat. Orbit/Nitro chains are **optimistic rollups (or AnyTrust chains)**, not sovereign chains with a validator-set consensus over ordering. Transaction **ordering is done by a sequencer** — on Arbitrum One a single centralized sequencer run by Offchain Labs; on Orbit chains typically a single operator/RaaS sequencer. There is **no BFT, DPoS, or PoA validator consensus producing blocks.** Safety/finality derives from: (a) posting data + assertions to **Ethereum L1 (which itself is PoS)**, and (b) **fraud proofs** — now the permissionless **BoLD** dispute protocol where a single honest party can defend the correct state via interactive bisection adjudicated by an L1 one-step-proof contract; or in **AnyTrust** mode a permissioned **Data Availability Committee (DAC)** vouches for data availability (cheaper, extra trust assumption). So the relevant guarantees are "inherited from Ethereum + fraud proofs / DAC," not classical validator consensus. Decentralization is a roadmap item (BoLD made Arbitrum One Stage 1; sequencer remains centralized; Timeboost adds an MEV auction; multi-party/decentralized sequencing targeted later). Verdict: Weak against the literal "BFT/DPoS/PoS/PoA" criterion — the model is sequencer + fraud-proof/DAC settling to Ethereum PoS, not pluggable validator consensus.

### 4. Actively maintained — **STRONG**
Very actively developed by Offchain Labs with Arbitrum DAO/Foundation governance over protocol upgrades. Latest Nitro release **v3.11.0 (22 Jun 2026)**; recent cadence shows multiple releases per quarter (v3.10.0 May 2026, v3.10.1/.2, v3.11.0-rc series). ArbOS upgrades are frequent and DAO-governed (ArbOS 51 "Dia" Jan 2026; ArbOS 60 in testnet). Licensing is **Business Source License 1.1** (not fully open until the 31 Dec 2030 change date, when it becomes Apache 2.0) — a real consideration for forking/competitive use, mitigated by the Additional Use Grant for chains settling under the Arbitrum Expansion Program. Large ecosystem of live chains. Verdict: Strong on maintenance; note the BUSL restriction.

---

## Architecture (Nitro Stack)

**The "Geth sandwich" / three layers:**
1. **Geth core (bottom):** a fork of go-ethereum performs EVM execution and state management, giving Ethereum-equivalent behavior.
2. **ArbOS (middle):** Arbitrum's L2 "operating system," written in Go. Handles cross-chain messaging, the fee/gas model, L1 data-posting accounting, retryable tickets, deposits/withdrawals, and (since ArbOS 32) the Stylus WASM VM. It is the State Transition Function applied to sequenced inputs.
3. **Node interface (top):** Ethereum-compatible JSON-RPC plus Arbitrum-specific helper methods (NodeInterface pseudo-precompile).

**Data flow:** The **Sequencer** orders incoming transactions and produces L2 blocks/receipts quickly (soft confirmations), then batches them, compresses with **Brotli**, and posts to Ethereum L1 — using **EIP-4844 blobs** (with calldata fallback) for data availability. Validators (now permissionless via BoLD) watch the chain and can post assertions/challenges to L1.

**Fraud proofs:** Deterministic STF means a dispute reduces (via multi-level bisection: block → big-step ~2^20 WASM instructions → single instruction) to a **one-step proof** executed by an L1 contract that acts as referee. Nitro compiles its STF to WASM (and to native for speed; to WAVM for proving) so disputes are provable on Ethereum.

**L1 contracts:** Inbox/SequencerInbox (data + force-inclusion), Outbox, Rollup core, Bridge, and the BoLD challenge/one-step-proof contracts live on the parent chain (Ethereum for L2; Arbitrum One/Nova for L3).

**Languages:** Go (Nitro node, ArbOS, geth fork, precompiles); Rust (Stylus VM components, fraud-prover tooling, and Stylus user contracts). Stylus user code also in C/C++ or anything compiling to WASM.

**Rollup vs AnyTrust modes:**
- **Rollup:** full transaction data posted to L1 (Ethereum) — maximal trustlessness, higher cost.
- **AnyTrust:** data availability handled by a permissioned **Data Availability Committee (DAC)**; only fallback data hits L1. Lower fees, extra trust assumption (the DAC). Arbitrum Nova is the flagship AnyTrust chain; many gaming/social Orbit chains use AnyTrust.

---

## Ethereum Compatibility & ArbOS Versions

**Equivalence:** EVM opcodes, precompiles, state trie, and JSON-RPC are inherited from the geth fork, so the chain is "Ethereum-equivalent" for tooling purposes.

**Deviations from Ethereum mainnet (by design as an L2):**
- **Dual fee model:** child-chain (L2) EVM gas under EIP-1559-style dynamic pricing **plus** a parent-chain (L1) data fee for posting batches. ArbOS 51 raised the minimum L2 base fee from 0.01 to 0.02 gwei and introduced multi-target gas pricing over multiple adjustment windows.
- **Block/timing:** sequencer-produced blocks; batched, compressed L1 posting; different block-time and gas-accounting semantics.
- **Extra tx types & address aliasing** for L1↔L2 messaging.
- **Stylus:** a second (WASM) VM coexists with the EVM.

**Recent ArbOS releases (named after moons, +10 per major, starting ArbOS 20 "Atlas"):**
- **ArbOS 11** — earlier baseline.
- **ArbOS 20 "Atlas"** — Ethereum **Dencun**/EIP-4844 blob support era.
- **ArbOS 30 (Stylus AIP)** — governance vote to activate Stylus.
- **ArbOS 32 "Bianca"** — Stylus (WASM/Rust) live on mainnet (Stylus shipped to Arbitrum One ~Sep 2024).
- **ArbOS 40 "Callisto"** — tracked Ethereum **Pectra**-era changes.
- **ArbOS 51 "Dia"** — current on Arbitrum One; voted 18 Dec 2025, activated **8 Jan 2026**; min Nitro **v3.9.6**. Implements EVM-relevant parts of Ethereum **Fusaka**, including EIP-7951 (secp256r1 / P-256 precompile), EIP-7825 (32M per-tx gas cap), EIP-7939 (CLZ opcode), EIP-7823 & EIP-7883 (ModExp limits/gas), EIP-2537 (BLS12-381), EIP-7910 (eth_config RPC); plus new multi-target gas pricing, full-block MaxTxGasLimit, and optional native-token mint/burn (disabled on Arbitrum One/Nova).
- **ArbOS 60** — in development on Arbitrum Sepolia testnet (rc activating mid-2026); no Arbitrum One date set as of June 2026. Adds ArbNativeTokenManager (`0x73`) precompile for native gas-token mint/burn (ArbOS 41+).

---

## Precompiles & Extensibility

**Built-in ArbOS precompiles (Solidity-callable, fixed addresses):**

| Precompile | Address | Purpose |
|---|---|---|
| ArbSys | `0x64` | L2 block info, L1 messaging (sendTxToL1), address aliasing, system calls |
| ArbInfo | `0x65` | Account/contract balance & code lookups |
| ArbAddressTable | `0x66` | Register addresses → compact indices (calldata compression) |
| ArbFunctionTable | `0x68` | Legacy (no longer used) |
| ArbosTest | `0x69` | Testing — burn arbitrary gas |
| ArbOwnerPublic | `0x6b` | Public read of chain-owner/config info |
| ArbGasInfo | `0x6c` | Gas/price info & estimation (L1 & L2 components) |
| ArbAggregator | `0x6d` | Configure tx aggregation/batch poster |
| ArbRetryableTx | `0x6e` | Manage retryable tickets (L1→L2 messaging) |
| ArbStatistics | `0x6f` | Pre-Nitro historical chain stats |
| ArbOwner | `0x70` | Chain administration (owner-only) |
| ArbWasm | `0x71` | Activate/manage Stylus (WASM) contracts |
| ArbWasmCache | `0x72` | Manage Stylus compiled-artifact cache |
| ArbNativeTokenManager | `0x73` | Native gas-token mint/burn (ArbOS 41+) |
| ArbDebug | `0xff` | Debug/test tools (gated by chain param) |
| ArbBLS | — | Disabled (former BLS key registry) |
| NodeInterface | (virtual) | RPC-only helper methods, not a real chain contract |

**Custom precompiles (operator extensibility):** Implemented in **Go** in Nitro's `/precompiles` dir with a matching Solidity interface in `/contracts-local/src/precompiles`. Five supported customization modes: add methods to existing precompiles, create new precompiles, define custom events, customize gas costs, and read/write ArbOS state. Addresses must avoid Ethereum/ArbOS reserved ranges (per EIP-7587). **Major caveat:** the documented-safe pattern works reliably only for **view/`eth_call` precompiles** — adding non-view/pure methods or calling custom precompiles from other contracts can **break deterministic block validation / fraud-proving** unless done carefully and folded into the ArbOS upgrade + WASM-module-root process. Offchain Labs recommends partnering with a RaaS provider and getting audits. This is a genuine "native Turing-incomplete module exposed as an EVM contract" path, with operational complexity.

**Stylus (alternative extensibility, Turing-complete):** A **coequal second VM** (MultiVM) running WASM alongside the EVM, activated with ArbOS 32 "Bianca." Contracts in **Rust, C, C++** (or any WASM-targeting language) are compiled to WASM, then `activate`d via the **ArbWasm** precompile (which validates bytecode, adds gas metering/memory/depth checks, and compiles to native ARM/x86 per node). Fully interoperable with Solidity (bidirectional calls). Uses a finer-grained gas unit called **"ink"** (~1000× smaller than gas). 10–100× faster for compute-heavy workloads (hashing, big-int, crypto). Contracts require periodic re-activation. Stylus is the recommended path for heavy/non-Solidity logic; custom Go precompiles are the path for chain-native/system-level extensions.

---

## Consensus & Security Model

**Honest framing:** There is **no validator consensus (BFT/DPoS/PoA) over transaction ordering.** Orbit/Nitro chains are optimistic rollups (or AnyTrust):

- **Sequencer (ordering):** A single, typically centralized sequencer (Offchain Labs on Arbitrum One; a RaaS/operator on most Orbit chains) orders txs and issues fast soft confirmations. **Force-inclusion** via the L1 SequencerInbox and a **Delay Buffer** mechanism mitigate censorship if the sequencer misbehaves/goes down. **Timeboost** adds a sealed-bid MEV/priority auction on Arbitrum One.
- **Data availability:** **Rollup mode** posts (compressed) data to **Ethereum L1** (EIP-4844 blobs). **AnyTrust mode** uses a permissioned **Data Availability Committee (DAC)** — cheaper, extra trust assumption.
- **Settlement / dispute resolution:** **BoLD** (Bounded Liquidity Delay), a **permissionless** interactive fraud-proof protocol, replaced the old allowlisted challenge system. A single honest party can defend the correct state; disputes bisect down to a one-step proof adjudicated by an **L1 contract on Ethereum (PoS)**. Live on Arbitrum One, Nova, and Sepolia. Posting assertions requires a large bond (≈3,600 ETH on Arbitrum One); watchtower validation is free and permissionless. Challenge window ≈ two challenge periods (~6.4 days each on Arbitrum One) + 2-day Security Council grace.
- **Ultimate trust root:** **Ethereum** — for data availability (Rollup) and as the neutral referee for fraud proofs. So security is "inherited from Ethereum PoS + fraud proofs," not from the chain's own validator set.

**Decentralization status (June 2026):** BoLD gives Arbitrum One **Stage 1**. The **sequencer is still centralized**; a **Security Council** multisig can intervene, which (along with sequencer centralization) keeps it short of Stage 2. Decentralized/multi-party sequencing is on the roadmap (targeted later in 2026 per public statements).

**Implication for the comparison:** If the requirement is a pluggable L1 validator consensus (BFT/DPoS/PoS/PoA), Orbit does **not** provide it. It provides a sequencer + fraud-proof/DAC model anchored to Ethereum. This is a fundamental architectural difference from sovereign-L1 frameworks.

---

## Maintenance, Licensing & Ecosystem

- **Maintainer:** Offchain Labs (core dev) with **Arbitrum DAO / Arbitrum Foundation** governance over protocol/ArbOS upgrades and the Security Council.
- **Latest Nitro release:** **v3.11.0 (22 Jun 2026)**; recent: v3.10.2 (2 Jun 2026), v3.10.1 (19 May 2026), v3.10.0 (11 May 2026), plus v3.11.0-rc series. Cadence: multiple releases per quarter; only some carry ArbOS upgrades.
- **Latest ArbOS:** ArbOS 51 "Dia" live on Arbitrum One (Jan 2026); ArbOS 60 in testnet.
- **License:** **Business Source License 1.1 (BUSL 1.1)**. **Change Date: 31 Dec 2030** (or 4th anniversary of distribution, whichever first), after which it converts to **Apache 2.0**. **Additional Use Grant** permits running nodes on public Arbitrum chains and deploying Nitro as a new chain *provided it settles under the Arbitrum Expansion Program (AEP)* (historically Arbitrum One/Nova; AEP now allows other parent chains under terms). Competitive/commercial production use outside the grant requires a commercial license or is prohibited until the change date — a real consideration for would-be forkers.
- **Major Orbit chains:** Arbitrum Nova (AnyTrust), Xai (gaming), Sanko, Degen Chain, ApeChain/Treasure-class gaming L3s, Pirate Nation, Kinto, Plume (RWA, ~$265M TVL mid-2025), SX Network, Gravity (Conduit-powered AnyTrust). Large RaaS ecosystem (Conduit, Caldera, Gelato, Alchemy, Zeeve, AltLayer) provisions Orbit chains.

---

## Pros / Cons Against the 4 Criteria

**Pros**
- Best-in-class EVM equivalence via the geth fork; tracks Ethereum hardforks (Fusaka) within months; full standard-tooling compatibility.
- Two real extensibility paths: customizable Go precompiles (Turing-incomplete native modules as EVM contracts) **and** Stylus WASM (Rust/C/C++) as a coequal VM.
- Extremely active maintenance, mature, battle-tested at scale; permissionless fraud proofs (BoLD); rich RaaS ecosystem.
- Inherits Ethereum L1 security (strong trust root) in Rollup mode.

**Cons**
- **Not a sovereign-L1 BFT/PoS/PoA framework** — no validator consensus over ordering; relies on a (usually centralized) sequencer + fraud proofs/DAC. Fails the literal "consensus support" criterion.
- Centralized sequencer and Security Council mean it's not yet Stage-2 decentralized; liveness/censorship depend on the operator (mitigated, not eliminated, by force-inclusion).
- L2 deviations (dual gas model, L1 data fees, address aliasing, batch/blob block structure) mean it is Ethereum-*equivalent*, not byte-identical to mainnet.
- **BUSL 1.1** restricts competitive/standalone use until 31 Dec 2030; deploying outside the AEP grant needs a commercial license.
- Custom precompiles that mutate state or are called on-chain require care to avoid breaking fraud-proof determinism — effectively expert/RaaS territory.
- AnyTrust mode trades trustlessness for cost via a permissioned DAC.

**Overall verdict:** Strong as an EVM L2/L3 rollup framework (compatibility, precompiles/Stylus extensibility, maintenance); fundamentally mismatched with a requirement for sovereign validator consensus (BFT/DPoS/PoS/PoA), since its security model is a centralized sequencer + fraud proofs (BoLD)/DAC settling to Ethereum.

---

## Sources

- [ArbOS software releases: Overview — Arbitrum Docs](https://docs.arbitrum.io/run-arbitrum-node/arbos-releases/overview)
- [ArbOS 51 "Dia" — Arbitrum Docs](https://docs.arbitrum.io/run-arbitrum-node/arbos-releases/arbos51)
- [Inside Arbitrum Nitro — Arbitrum Docs](https://docs.arbitrum.io/how-arbitrum-works/inside-arbitrum-nitro)
- [Geth at the core — Arbitrum Docs](https://docs.arbitrum.io/how-arbitrum-works/geth-at-the-core)
- [ArbOS — Arbitrum Docs](https://docs.arbitrum.io/how-arbitrum-works/arbos/introduction)
- [Precompiles reference — Arbitrum Docs](https://docs.arbitrum.io/arbitrum-essentials/precompiles/reference)
- [How to customize your Arbitrum chain's precompiles — Arbitrum Docs](https://docs.arbitrum.io/launch-arbitrum-chain/customize-your-chain/customize-precompile)
- [A gentle introduction to Stylus — Arbitrum Docs](https://docs.arbitrum.io/stylus/gentle-introduction)
- [Stylus Quickstart (Rust) — Arbitrum Docs](https://docs.arbitrum.io/stylus/quickstart)
- [stylus-sdk-c (C/C++ contracts) — GitHub](https://github.com/OffchainLabs/stylus-sdk-c)
- [Overview of BoLD — Arbitrum Docs](https://docs.arbitrum.io/how-arbitrum-works/bold/gentle-introduction)
- [Arbitrum Orbit AnyTrust Chains — Offchain Labs (Medium)](https://medium.com/offchainlabs/arbitrum-orbit-anytrust-chains-465ec2f1a5d8)
- [Overview of Arbitrum chains — Arbitrum Docs](https://docs.arbitrum.io/launch-arbitrum-chain/a-gentle-introduction)
- [Launch a Chain (Orbit) — arbitrum.io](https://arbitrum.io/orbit)
- [Arbitrum Stylus — arbitrum.io](https://arbitrum.io/stylus)
- [The state of Arbitrum's progressive decentralization — Arbitrum DAO Governance Docs](https://docs.arbitrum.foundation/state-of-progressive-decentralization)
- [Network upgrades — Arbitrum DAO Governance Docs](https://docs.arbitrum.foundation/network-upgrades)
- [AIP: Activate Stylus (ArbOS 30) — Arbitrum Forum](https://forum.arbitrum.foundation/t/aip-activate-stylus-and-enable-next-gen-webassembly-smart-contracts-arbos-30/22970)
- [Nitro releases — GitHub](https://github.com/OffchainLabs/nitro/releases)
- [Nitro v3.10.0 release — GitHub](https://github.com/OffchainLabs/nitro/releases/tag/v3.10.0)
- [Nitro repository — GitHub](https://github.com/OffchainLabs/nitro)
- [Nitro LICENSE.md (BUSL 1.1) — GitHub](https://github.com/OffchainLabs/nitro/blob/master/LICENSE.md)
- [Arbitrum chain licensing (AEP) — Arbitrum Docs](https://docs.arbitrum.io/launch-arbitrum-chain/aep-license)
- [Arbitrum 2026 Roadmap — BlockEden](https://blockeden.xyz/blog/2026/02/01/arbitrum-2026-roadmap-arbos-dia-gaming-catalyst-stylus/)
- [arbitrum-orbit chains TVL — DefiLlama](https://defillama.com/chains/arbitrum-orbit)
- [A closer look at Arbitrum Nitro and Orbit Chains — OAK Research](https://oakresearch.io/en/reports/protocols/a-closer-look-at-arbitrum-nitro-orbit-chains)
- [Ethereum L2 Sequencers: Centralized Today, Decentralized Tomorrow — eco.com](https://eco.com/support/en/articles/14798711-ethereum-l2-sequencers-centralized-today-decentralized-tomorrow)
