# Polygon Edge & Polygon CDK — EVM Chain Framework Research

**Last verified: 2026-06-24**

## Executive summary

Polygon offers two generations of "launch your own EVM chain" technology, and the distinction is now the single most important fact about them. **Polygon Edge** is a Go-based framework for standalone, sovereign EVM-compatible L1/L2 networks using IBFT consensus (in PoA or PoS modes). It is **discontinued and archived**: Polygon Labs announced in December 2023 it would stop contributing, the last release (v1.3.2) shipped January 2024, the last code push was August 2024, and the GitHub repository was **archived (read-only) on 4 December 2024**. It is Apache-2.0, so it can be forked, but there is no first-party maintenance and no security support. **Polygon CDK (Chain Development Kit)** is the actively developed successor: an open-source toolkit for launching ZK-powered L2 chains that connect to the **AggLayer** cross-chain interoperability protocol. CDK's flagship execution client, **cdk-erigon**, is under active development (v2.64.2 on 16 Jan 2026; v2.65.0-RC3 on 18 Feb 2026), and CDK now offers multiple stacks (cdk-erigon zkEVM, cdk-opgeth/OP-Stack, sovereign, validium). For the four evaluation criteria, neither product is a strong fit for *latest-mainnet 1:1 equivalence combined with pluggable Turing-incomplete precompile modules*: Edge is dead, and CDK's zkEVM is a near-Type-2 (not Type-1) zkEVM that lags Ethereum's latest hardforks and offers no plugin API for custom precompiles (extension requires forking the Go client).

---

## Fit against requirements

| # | Requirement | Verdict | Rationale |
|---|-------------|---------|-----------|
| 1 | 1:1 compatibility with latest Ethereum mainnet | **Weak (Edge) / Partial (CDK)** | Edge is frozen at a pre-Shanghai-era EVM and unmaintained. CDK's zkEVM is a **near-Type-2** zkEVM (Etrog made it "fully Type 2"), not Type 1; it has documented opcode/precompile deviations (no SELFDESTRUCT, modified BLOCKHASH/NUMBER/DIFFICULTY, some precompiles revert) and tracks Ethereum forks on a delay. cdk-erigon supports up to Cancun/Dencun and is working toward Prague/Pectra, but full Ethereum equivalence is a roadmap goal (the standalone Type-1 prover), not the default chain you launch. |
| 2 | Add Turing-incomplete modules exposed as EVM precompiles | **Weak** | Neither Edge nor cdk-erigon exposes a documented plugin/registration API for custom precompiles. Precompiles are **hardcoded in the Go source** (Edge: `state/runtime/precompiled`; cdk-erigon: Erigon's precompile set + zkEVM ROM). Adding a custom precompile means forking and modifying the client, and on CDK it must also be reflected in the zk prover/ROM — a much heavier lift than a plugin. No Avalanche-style "stateful precompile" SDK exists. |
| 3 | Consensus: BFT, DPoS, PoS, PoA | **Strong (Edge) / Weak (CDK)** | **Edge** ships IBFT (Istanbul BFT) in two modes — **PoA** (default, voted validator set) and **PoS** (staking + epochs), with live switching; IBFT is a genuine BFT protocol. It does not natively offer DPoS. **CDK** chains are ZK rollups/validiums: there is **no decentralized validator consensus** — they run a **single (centralized) sequencer** whose output is secured by ZK validity proofs (or pessimistic proofs) on the settlement layer. So CDK does not satisfy a BFT/PoS/PoA validator-consensus requirement. |
| 4 | Actively maintained | **Weak (Edge) / Strong (CDK)** | **Edge: discontinued + archived (read-only) since 4 Dec 2024**; last release v1.3.2 (Jan 2024). Not maintained by Polygon; community forks only. **CDK/cdk-erigon: actively maintained** (releases through Jan–Feb 2026), with AggLayer v0.3 live (June 2025) and AggLayer "full maturity" targeted for 2026. Note Polygon zkEVM mainnet itself is being wound down/deprecated by 2026 in favor of CDK + AggLayer. |

**Overall:** If you need a sovereign, self-contained PoA/PoS BFT EVM chain you control end-to-end, Edge matches the consensus and architecture model — but it is abandonware. CDK is alive and well-resourced but is a ZK-rollup framework with a centralized sequencer and a not-quite-Ethereum-equivalent zkEVM, and neither product gives you a clean precompile plugin system.

---

## Architecture

### Polygon Edge
- **Type:** Standalone, modular framework for building **sovereign EVM-compatible blockchains** (independent L1s, or sidechains/L2s in the loose 2021-era sense). Each chain runs its own validator set and is self-contained — it does **not** depend on Ethereum for security or settlement.
- **Language:** Go (~97%), with some Solidity for system/staking contracts (~2%).
- **License:** Apache-2.0.
- **Modular design:** libp2p networking, JSON-RPC module, TxPool, blockchain/state modules, and a pluggable consensus module. Provides standard Ethereum JSON-RPC so MetaMask/Hardhat/ethers/web3 tooling connect to it.
- **Custom features:** predeployment of contracts at genesis, EIP-1559-style fees in later versions, bridging (a built-in bridge to Ethereum existed in the broader Edge/"Supernets" line).

### Polygon CDK
- **Type:** Open-source toolkit to launch **ZK-powered Layer 2 chains** that settle to Ethereum and plug into the **AggLayer** for unified liquidity/cross-chain interoperability. This is fundamentally an L2-rollup architecture, not a sovereign L1.
- **Execution stacks (multistack):**
  - **cdk-erigon** — a fork of Erigon optimized for the Polygon Hermez zkEVM; the primary zkEVM execution client (Go). ~10x less disk and far faster sync than the legacy zkNode stack; production peaks cited around 5,500 TPS.
  - **cdk-opgeth** — an OP-Stack (op-geth) based configuration that connects natively to AggLayer (announced 2025, "CDK goes multistack"), supported with partners like Conduit; secured via **pessimistic proofs**.
- **Operating modes:** rollup (data on Ethereum), **validium** (off-chain DA), **private validium** (institutional privacy), and **sovereign** (AggLayer connectivity secured by pessimistic proofs, *no prover required*).
- **Components of a zkEVM CDK chain:** sequencer (orders/executes txs, produces batches), aggregator + prover (generates ZK validity proofs), the L1 settlement contracts, and the AggLayer/Unified Bridge connection.
- **Settlement/proving:** ZK validity proofs (zkEVM chains) or pessimistic proofs (sovereign/OP-Stack chains) submitted to Ethereum.

---

## EVM compatibility / 1:1 mainnet equivalence

### Polygon Edge
- Provided "full compatibility with Ethereum smart contracts and transactions" **for its era** — but it is frozen. The EVM/fork support corresponds to v1.3.2 (early 2024) and earlier; it predates Cancun/Dencun blob mechanics and never tracked subsequent forks. It is **not** suitable for "latest Ethereum mainnet" equivalence and will only fall further behind as an unmaintained codebase.

### Polygon CDK / zkEVM
- **zkEVM type:** The **Etrog** upgrade made Polygon zkEVM a *de facto* **Type 2 zkEVM** (adding precompiles ecAdd, ecMul, ecPairing, SHA-256, modexp). Subsequent upgrades (e.g. **Feijoa**) brought **EIP-4844 / blob** support (Dencun alignment) and fee reductions. cdk-erigon releases reference Cancun support and EIP-2935 historical block hashes, with Prague/Pectra work in CI — i.e. it is tracking but trailing Ethereum mainnet.
- **Type 1 (full equivalence) is a separate prover, not the default chain:** Polygon released an open-source **Type-1 Prover** (Feb 2024) capable of proving *any* EVM chain (even Ethereum mainnet blocks) at ~$0.002–0.003/tx. This is real progress toward full equivalence, but a launched CDK zkEVM chain is the near-Type-2 zkEVM, not a Type-1-equivalent EVM out of the box.
- **Documented EVM deviations (zkEVM-ROM vs EVM):**
  - **SELFDESTRUCT removed**, replaced by **SENDALL**.
  - **EXTCODEHASH** returns the bytecode hash from the zkEVM state tree without the "empty account" check.
  - **DIFFICULTY/PREVRANDAO** returns `0` instead of a random value.
  - **BLOCKHASH** returns all previous block hashes (not just the last 256).
  - **NUMBER** semantics tied to processable transactions.
  - Several **precompiles not implemented** are treated as a **revert** (gas returned, success=0); only the supported set (identity, ecRecover, ecAdd/ecMul/ecPairing, SHA-256, modexp, etc.) functions.
  - Minor **gas-cost** differences in edge cases.
- **JSON-RPC:** Broadly Ethereum-JSON-RPC compatible (standard `eth_*` namespace works with MetaMask/Hardhat/Foundry/ethers/web3), **plus** zkEVM-specific extensions (`zkevm_*` namespace). cdk-erigon exposes standard Erigon RPC plus the zkEVM methods.

**Bottom line on criterion 1:** Edge fails latest-mainnet equivalence (frozen, unmaintained). CDK is *close* and improving but is explicitly a near-Type-2 zkEVM with known opcode/precompile divergences and a fork lag — not a drop-in 1:1 latest-Ethereum equivalent. Most standard tooling works; equivalence-sensitive contracts (SELFDESTRUCT, unsupported precompiles, randomness, BLOCKHASH assumptions) can break.

---

## Precompiles / Turing-incomplete module extensibility

- **No plugin/registration API in either product.** Precompiles are compiled into the Go client.
  - **Edge:** `github.com/0xPolygon/polygon-edge/state/runtime/precompiled` defines a fixed `Precompiled` runtime with `CanRun`/`Run`/`Name`; the set (Blake2f, BN256, modexp, BLS aggregate-sig verification, native transfer, console, base) is hardcoded. To add one you fork Edge and edit the Go source.
  - **cdk-erigon:** inherits Erigon's precompile set; for a zkEVM CDK chain, any new precompile must *also* be implemented in the zkEVM **ROM / prover circuits**, because anything executed must be provable. That makes custom precompiles substantially harder than on a plain Go EVM.
- **No Avalanche-style "stateful precompile" SDK** and no module system comparable to Cosmos SDK / Substrate pallets. This is the key limitation against criterion 2: you *can* add Turing-incomplete native functions via precompiles, but only by **forking and recompiling the client** (and updating the prover on CDK) — there is no supported extension surface.

**Bottom line on criterion 2: Weak.** Technically possible by forking; not supported as a first-class extensibility feature.

---

## Consensus

### Polygon Edge — **IBFT (Istanbul BFT)**, two modes
- **IBFT PoA (default):** dynamic validator set; validators added/removed by on-chain voting; round-robin block proposal; block accepted on **>2/3 supermajority**. This is a genuine BFT (immediate finality) protocol — covers the **BFT** and **PoA** requirements.
- **IBFT PoS:** staking-based validator selection with **epochs** (special end-of-epoch blocks with no transactions); validator set determined by stake. Covers **PoS**.
- **Switchable:** chains can switch between PoA and PoS without resetting the chain.
- **DPoS:** not a distinct native mode (PoS is stake-weighted but Edge does not ship a delegated-voting DPoS module out of the box).

### Polygon CDK — **sequencer model, not validator consensus**
- CDK zkEVM/validium chains run a **single sequencer** (centralized by default) that orders and executes transactions and posts batches; correctness is enforced by **ZK validity proofs** verified on Ethereum (or **pessimistic proofs** for sovereign/OP-Stack chains). 
- There is **no BFT/PoA/PoS validator consensus** among multiple block producers in the default model. Decentralizing the sequencer is a roadmap/AggLayer-level concern, not a turnkey consensus option you select.
- cdk-erigon runs as either **RPC mode** (follows a remote sequencer via data streams) or **sequencer mode**.

**Bottom line on criterion 3:** Edge is **Strong** (real BFT with PoA and PoS, live switching). CDK is **Weak** against a BFT/PoS/PoA-validator requirement (single sequencer + proofs).

---

## Maintenance status (critical)

### Polygon Edge — **DISCONTINUED & ARCHIVED**
- **Dec 2023:** Polygon Labs announced it would **discontinue contributions to Edge** and focus on Polygon CDK.
- **24 Jan 2024:** last release **v1.3.2**.
- **27 Aug 2024:** last commit/push to the repo.
- **4 Dec 2024:** GitHub repo **`0xPolygon/polygon-edge` archived — read-only**.
- **Maintainer today:** none at Polygon Labs. Apache-2.0 means anyone may fork; some ecosystem actors (e.g. providers around Dogechain) maintain private forks, but there is **no canonical maintained upstream and no security support**.

### Polygon CDK / AggLayer — **ACTIVE**
- **cdk-erigon:** v2.64.2 (16 Jan 2026), with v2.65.0-RC2 (4 Feb 2026) and v2.65.0-RC3 (18 Feb 2026) — actively released.
- **AggLayer:** pessimistic proofs live on mainnet (3 Feb 2025); **AggLayer v0.3 live (23 June 2025)** adding execution-proof mode and groundwork for non-CDK chains (e.g. Polygon PoS) to join; **AggLayer full maturity targeted for 2026**.
- **Multistack:** cdk-opgeth (OP-Stack + AggLayer) announced 2025; no "tax" to connect OP-Stack chains to AggLayer (2026).
- **Caveat:** **Polygon zkEVM (the public mainnet) is being deprecated/wound down by 2026** as Polygon consolidates around CDK + AggLayer (and POL/Polygon 2.0). The *framework* (CDK) is alive; the original public zkEVM *network* is sunsetting.

---

## Production chains

- **Polygon Edge (legacy):** **Dogechain** (Dogecoin-adjacent L2-style chain) is the best-known Edge mainnet; various enterprise/appchains used Edge/Supernets in 2022–2023.
- **Polygon CDK:** **OKX X Layer**, **Astar zkEVM**, **Immutable zkEVM** (gaming), **Manta Pacific** (at points in its history), **Katana** (OP-Stack/AggLayer), plus commitments from Flipkart, Gnosis Pay, Palm, Nubank, Wirex, IDEX, GameSwift and others. 190+ dApps/chains cited across the CDK ecosystem.

---

## Edge vs CDK at a glance

| Dimension | Polygon Edge | Polygon CDK |
|-----------|--------------|-------------|
| Model | Sovereign standalone EVM L1/L2 | ZK-powered L2 connected to AggLayer |
| Security | Own validator set (IBFT) | Ethereum settlement + ZK/pessimistic proofs |
| Consensus | IBFT PoA / PoS (switchable, BFT) | Single sequencer (no validator consensus) |
| Language | Go | Go (cdk-erigon), Go/op-geth (cdk-opgeth) |
| EVM target | Frozen, pre-Dencun | Near-Type-2 zkEVM, tracking Cancun→Prague |
| Precompile extension | Fork Go source (hardcoded) | Fork Go source + update prover/ROM |
| License | Apache-2.0 | Open source (Apache/GPL components) |
| Maintenance | **Archived (read-only), Dec 2024** | **Active (2026)** |
| Interop | Standalone | AggLayer unified bridge / liquidity |

---

## Sources

- [An important update about Polygon Edge (Polygon blog)](https://polygon.technology/blog/polygon-labs-to-focus-contributions-on-polygon-cdk-discontinues-contributions-for-edge)
- [Polygon Labs discontinues Edge contributions in favor of CDK (The Block)](https://www.theblock.co/post/267883/polygon-labs-to-discontinue-edge-development-in-favor-of-expanding-cdk-use)
- [Polygon Stops Work on 'Edge,' Used to Build Dogechain (CoinDesk)](https://www.coindesk.com/tech/2023/12/15/polygon-stops-work-on-edge-used-to-build-dogechain-as-focus-turns-to-zk)
- [Polygon halts development on Edge, pivots to CDK (CryptoSlate)](https://cryptoslate.com/polygon-halts-development-on-edge-pivoting-to-polygon-cdk/)
- [0xPolygon/polygon-edge GitHub (archived repo)](https://github.com/0xPolygon/polygon-edge)
- [0xPolygon/polygon-edge releases](https://github.com/0xPolygon/polygon-edge/releases)
- [Polygon Edge precompiled package (pkg.go.dev)](https://pkg.go.dev/github.com/0xPolygon/polygon-edge/state/runtime/precompiled)
- [Polygon Edge consensus docs — IBFT PoA](https://polygon-edge-v063.evmbuilder.com/docs/consensus/poa/)
- [Polygon Edge consensus docs — PoS concepts](https://polygon-edge-v063.evmbuilder.com/docs/consensus/pos-concepts/)
- [0xPolygon/cdk-erigon GitHub](https://github.com/0xPolygon/cdk-erigon)
- [0xPolygon/cdk-erigon releases](https://github.com/0xPolygon/cdk-erigon/releases)
- [Eggfruit Upgrade: cdk-erigon sequencer live (Polygon blog)](https://polygon.technology/blog/eggfruit-upgrade-incoming-polygon-zkevm-mainnet-beta-will-see-the-cdk-erigon-sequencer-go-live)
- [Erigon client comes to Polygon CDK (Polygon blog)](https://polygon.technology/blog/erigon-client-comes-to-polygon-cdk)
- [AggLayer CDK Erigon: enterprise-ready ZK blockchains (agglayer.dev)](https://www.agglayer.dev/blogs/agglayer-cdk-erigon-the-stack-powering-enterprise-ready-zk-blockchains)
- [CDK Goes Multistack: starting with OP Stack (Polygon blog)](https://polygon.technology/blog/cdk-goes-multistack-aggregate-everything-starting-with-op-stack)
- [Polygon CDK docs (docs.polygon.technology)](https://docs.polygon.technology/chain-development/cdk)
- [Polygon zkEVM EVM differences (Polygon Knowledge Layer)](https://docs.polygon.technology/zkEVM/spec/evm-differences/)
- [Polygon zkEVM vs Ethereum differences (Alchemy Docs)](https://docs.alchemy.com/reference/polygon-zkevm-and-ethereum-differences)
- [Polygon releases Type 1 Prover (CoinDesk)](https://www.coindesk.com/tech/2024/02/08/polygon-releases-type-1-prover-claiming-milestone-set-by-ethereums-vitalik-buterin)
- [Polygon zkEVM Type 1 prover reaches status (Blockworks)](https://blockworks.co/news/polygon-zkevm-type-1-prover)
- [The Etrog Upgrade makes Polygon zkEVM fully Type 2 (Polygon blog)](https://polygon.technology/blog/the-etrog-upgrade-is-on-testnet-making-polygon-zkevm-fully-type-2)
- [Etrog upgrade live on mainnet (Polygon blog)](https://polygon.technology/blog/polygon-zkevm-the-etrog-upgrade-is-live-on-mainnet)
- [EIP-4844 Is Coming to Polygon (Polygon blog)](https://polygon.technology/blog/eip-4844-is-coming-to-polygon)
- [Pessimistic Proofs live on AggLayer mainnet (agglayer.dev)](https://www.agglayer.dev/blogs/major-development-upgrade-for-a-multistack-future-pessimistic-proofs-live-on-agglayer-mainnet)
- [AggLayer v0.3 Is Live (agglayer.dev)](https://www.agglayer.dev/blogs/agglayer-v0-3-is-live)
- [Polygon CDK chains list (Flagship.fyi)](https://flagship.fyi/outposts/polygon/the-polygon-cdk-boom-all-the-latest-chains-using-polygon-cdk/)
- [Polygon Ecosystem — CDK Providers](https://ecosystem.polygon.technology/spn/cdk/)
