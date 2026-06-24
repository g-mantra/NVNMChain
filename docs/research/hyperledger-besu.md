# Hyperledger Besu

> Enterprise-grade, Apache 2.0–licensed Ethereum execution client written in Java, governed by LF Decentralized Trust (formerly Hyperledger Foundation) with ConsenSys as the primary corporate contributor.

**Last verified:** 2026-06-24

---

## Executive summary

Hyperledger Besu is a production Ethereum **execution-layer client** — one of the small handful of clients that actually secure Ethereum mainnet today, alongside Geth, Nethermind, Erigon and Reth. Because it is a real mainnet client, it tracks the live Ethereum hardfork schedule closely: it shipped Pectra (May 2025) and Fusaka (December 3, 2025, including PeerDAS and Blob-Parameter-Only forks) on time, and it exposes the full standard JSON-RPC and Engine API surface, so essentially all existing Ethereum tooling works against it unchanged. Its second life is as the de-facto reference client for **permissioned/consortium EVM networks**: a single codebase runs both public mainnet and private networks, supporting QBFT and IBFT 2.0 (both Byzantine-fault-tolerant PoA) consensus, with Clique (also PoA) block production now deprecated. Besu is actively and heavily maintained (15 releases and 811 commits in 2025; latest stable **26.6.1**, June 17, 2026). The main caveat for the criteria here is extensibility: Besu has a capable **plugin API** for observability, RPC endpoints, transaction selection and storage, but it has **no first-class hook for registering custom precompiles** — custom precompiles require integrating against the `besu-evm` library and building a customized client (the path Linea and Hedera took), i.e. a code-level fork rather than a config/plugin drop-in.

---

## Fit against requirements

### 1. 1:1 compatibility with the latest Ethereum mainnet — **Strong**

Besu is a genuine Ethereum mainnet execution client, not an EVM look-alike. It has implemented the full historical fork sequence through **Cancun/Deneb**, **Prague/Pectra** (live on mainnet May 2025), and **Fusaka** (live on mainnet December 3, 2025, slot 13,164,544), including EIP-7594 PeerDAS and the new Blob-Parameter-Only (BPO) fork mechanism. It implements the complete standard **JSON-RPC** API and the **Engine API** used by consensus clients (so it pairs with any CL such as Teku, Lighthouse, Prysm, Nimbus). It holds roughly **9–16% of Ethereum's execution-client share** (third behind Geth and Nethermind per clientdiversity.org), which is the strongest possible evidence that mainnet tooling works against it. *Caveat:* since Pectra, mainnet requires native crypto (BLS12-381 via `besu-native`); Besu has no pure-Java BLS12-381, so only Linux and macOS are supported for mainnet configurations.

### 2. Custom Turing-incomplete modules exposed as EVM precompiles — **Partial / Weak**

Besu ships a **plugin API** (`org.hyperledger.besu:plugin-api`) with a clean `BesuPlugin` lifecycle (`register` → `start` → `stop`) and a service-locator (`ServiceManager.getService`). Available services include `PicoCLIOptions`, `RpcEndpointService` (expose custom JSON-RPC methods), `BesuEvents`, `BlockchainService`, `MetricsSystem`, `StorageService`, `TransactionSelectionService`, `TransactionPoolValidatorService`, `TransactionSimulationService`, `BlockSimulationService`, `TraceService`, `PermissioningService` and more. This is excellent for monitoring, sequencing, validation, storage backends and custom RPC.

However, **there is no `PrecompiledContractService` or any plugin hook to register a custom precompiled contract or custom EVM opcode at runtime.** Custom precompiles are added through the **`besu-evm` standalone library** (the `PrecompiledContract` interface plus the precompile registry / `MainnetPrecompiledContractRegistries`), which requires writing Java and **building a customized Besu/EVM binary** — a code-level integration or fork, not a config flag or plugin jar. This is precisely the route taken by **Linea** (a Besu-based L2 with custom precompiles) and **Hedera** (which embeds the Besu EVM as a library, HIP-26). For a Turing-incomplete native module that you want callable as a precompile, the model is viable and well-trodden, but it is a *build-and-maintain-a-fork* effort, not a sanctioned drop-in extension point. Verdict leans Partial because the library path is real and proven, but Weak relative to chains (e.g. Avalanche Subnet-EVM) that expose genesis-configurable, register-at-build stateful precompiles as a supported first-class feature.

### 3. Consensus support (BFT / DPoS / PoS / PoA) — **Strong (for BFT/PoA and PoS-as-client); Weak for DPoS**

- **QBFT** — recommended enterprise consensus; **Byzantine Fault Tolerant** PoA; immediate finality; needs ≥4 validators and ≥2/3 online; supports validator management via smart contract or block header voting.
- **IBFT 2.0** — older **BFT** PoA protocol; same ≥4 validator / ≥2/3 quorum / immediate-finality properties; still supported, but QBFT is recommended for new networks.
- **Clique** — **PoA** (round-robin, not BFT, probabilistic finality). **Block production deprecated from 25.12.0** and Clique fully removed in **26.4.0**; Besu can sync legacy Clique-to-PoS chains but cannot validate or start new Clique networks.
- **PoS** — supported in the sense that mainnet is post-Merge: Besu is the **execution layer** and consensus/staking (Gasper / Casper-FFG + LMD-GHOST) is handled by a paired consensus client via the Engine API. Besu does not itself run a standalone PoS-with-staking consensus for private networks.
- **DPoS** — **not natively supported.** Besu offers BFT-PoA (QBFT/IBFT2) for permissioned networks and acts as the EL under Ethereum PoS; there is no built-in delegated-proof-of-stake engine.

So: BFT ✅ (QBFT, IBFT 2.0), PoA ✅ (QBFT, IBFT 2.0, legacy Clique), PoS ✅ as a mainnet EL, DPoS ❌.

### 4. Actively maintained — **Strong**

Latest stable release **26.6.1 (2026-06-17)**, preceded by 26.6.0 (requires JDK 25), 26.5.0, 26.4.0, 26.2.0, 26.1.0 — a roughly **monthly release cadence**. In 2025 the team shipped **15 releases / 811 commits** and three hardforks (Pectra, Fusaka, BPO1). Governed by **LF Decentralized Trust**, with **ConsenSys** as the dominant corporate contributor; 2026 roadmap includes 70M gas-limit work, Discovery v5 / eth/70, Bonsai Archive with state proofs, and graduating QBFT empty-block features out of experimental.

---

## Architecture

- **Language / license:** Java; **Apache 2.0**. Builds and runs on the JVM; **JDK 25 required from release 26.6.0** onward (previously JDK 21).
- **Dual-mode single codebase:** the same client targets public mainnet/testnets *and* private permissioned consortium networks — a defining design choice that lets enterprises reuse mainnet tooling internally.
- **EVM engine:** full EVM implementation, modularized into the reusable **`besu-evm`** library (usable standalone, e.g. by Hedera). Native precompiles (e.g. altbn128, BLS12-381) via `besu-native`, with pure-Java fallbacks where available.
- **State storage — Bonsai Tries:** default state layout storing trie nodes by location (not hash) and keeping leaf values in a separate **trie log**, reducing disk footprint and improving read performance vs. the classic Forest/hash-keyed layout. Backed by **RocksDB**, using RocksDB snapshots plus in-memory diff layers for concurrent state views. 2025 added History Expiry (prune pre-Merge data) and parallel transaction execution as standard.
- **APIs:** standard Ethereum **JSON-RPC** (HTTP/WS), **GraphQL**, the **Engine API** for consensus-client pairing, and a **Plugin API** for extensions.
- **Performance (2025):** EVM/storage improvements supported doubling Ethereum's block gas target from 30M to 60M; parallel transaction processing became standard.

## Compatibility detail

- Hardfork support is current with mainnet: Cancun/Deneb → **Prague/Pectra** (May 2025) → **Fusaka** (Dec 3, 2025) including **PeerDAS (EIP-7594)** and Fusaka EIPs (e.g. EIP-7825 transaction gas-limit cap, EIP-7934 RLP block-size limit) plus **BPO forks** for blob scaling.
- Standard tooling (Hardhat, Foundry, ethers/web3, MetaMask, The Graph, block explorers, Tenderly-style tracers) works because the JSON-RPC, tracing and Engine API surfaces match mainnet semantics.
- **Native-crypto requirement** since Pectra restricts supported mainnet platforms to Linux and macOS (no pure-Java BLS12-381).

## Precompiles & extensibility detail

- **Plugin API** (jar-based, `org.hyperledger.besu:plugin-api`, served from the Hyperledger/Besu Maven repo; `besu-evm` is published as an API dependency). Strong for: custom RPC methods (`RpcEndpointService`), CLI options (`PicoCLIOptions`), event streaming (`BesuEvents`), transaction selection/sequencing (`TransactionSelectionService`), tx-pool validation, custom storage (`StorageService`), tracing, metrics, permissioning.
- **No runtime/plugin precompile registration.** To add a **custom precompile** you implement the `PrecompiledContract` interface and register it in the precompile registry inside the `besu-evm` library, then **build a customized client** — used in production by **Linea** and by **Hedera** (Besu EVM as embedded library). There is **no genesis-config switch** for custom precompiles (contrast with Avalanche Subnet-EVM's genesis-enabled stateful precompiles).
- Implication for "Turing-incomplete native modules as precompiles": feasible and proven, but it is a **maintain-your-own-build** path, not a supported configuration knob.

## Consensus detail

| Protocol | Family | Finality | Status | Notes |
|---|---|---|---|---|
| QBFT | BFT (PoA) | Immediate | **Recommended** | ≥4 validators, ≥2/3 online; on-chain or header-vote validator mgmt |
| IBFT 2.0 | BFT (PoA) | Immediate | Supported | Predecessor to QBFT; same fault model |
| Clique | PoA (non-BFT) | Probabilistic | **Deprecated** | Block production deprecated 25.12.0; removed 26.4.0; sync-only for legacy chains |
| Ethereum PoS | PoS (Gasper) | — | Supported as EL | Besu = execution layer; CL handles staking via Engine API |
| DPoS | — | — | **Not supported** | No native delegated-PoS engine |

## Maintenance detail

- **Latest:** 26.6.1 (2026-06-17, optional update). **26.6.0** (2026-06): recommended, requires **JDK 25**.
- **Cadence:** ~monthly; **15 releases / 811 commits in 2025**.
- **Governance:** LF Decentralized Trust project; ConsenSys-led contributor base; active Discord/community; recent open-sourcing of the Fleet plugin and a Gradle meta-plugin.
- **2026 goals:** 70M gas target, Discovery v5 + eth/70, world-state download optimization, production-ready Bonsai Archive with state proofs, QBFT empty-block GA, FOCIL / encrypted-mempool research.

## Privacy & notable deployments

- **Privacy:** historically via **Tessera** (private transactions / privacy groups) — **deprecated from Besu 24.12.0**; the privacy plugin service is likewise deprecated. Permissioning (node/account allowlists, on-chain permissioning) remains a core feature for consortium networks.
- **Production / enterprise deployments:**
  - **eNaira** — Nigeria's CBDC (first African CBDC), launched on Besu in 2021, still operational.
  - **mBridge** — multi-CBDC cross-border platform (Hong Kong, Thailand, China mainland, UAE); UAE Digital Dirham pilot launched Nov 2025.
  - **Linea** — ConsenSys zkEVM L2, built on a Besu-derived client.
  - **Hedera** — embeds the Besu EVM as a library for its Smart Contract Service (HIP-26).
  - Broad use across LF Decentralized Trust consortium/enterprise networks and platforms (Kaleido, SettleMint, ChainLaunch, etc.).

---

## Pros / cons against the four criteria

**Pros**
- Real mainnet client with up-to-the-minute fork support (Fusaka/PeerDAS) and full standard RPC/Engine API → mainnet tooling "just works."
- Strong, immediate-finality **BFT-PoA** options (QBFT, IBFT 2.0) purpose-built for permissioned EVM networks.
- Very actively maintained under a neutral foundation; predictable monthly cadence; Apache 2.0.
- Single codebase spans public and private networks; reusable `besu-evm` library; mature storage (Bonsai) and performance work.
- Proven at sovereign/CBDC scale (eNaira, mBridge) and as an L2 base (Linea).

**Cons**
- **No first-class custom-precompile/extension hook** — custom precompiles require building/maintaining a customized client against `besu-evm` (fork-grade effort).
- **No DPoS**; PoS only as a mainnet execution layer, not as a standalone staking consensus for private chains.
- Clique block production deprecated/removed (minor — QBFT/IBFT2 are better anyway).
- Java/JVM and **JDK 25** requirement; mainnet now needs native crypto (Linux/macOS only).
- Tessera privacy deprecated — no maintained native confidential-transaction layer going forward.

---

## Sources

- [Besu documentation (home)](https://besu.hyperledger.org/) / [docs.besu-eth.org](https://docs.besu-eth.org/)
- [Besu GitHub releases](https://github.com/hyperledger/besu/releases) and [latest release API (26.6.1)](https://api.github.com/repos/besu-eth/besu/releases/latest)
- [Besu CHANGELOG (main)](https://raw.githubusercontent.com/hyperledger/besu/main/CHANGELOG.md)
- [Besu repository (besu-eth/besu)](https://github.com/besu-eth/besu)
- [Plugin API interfaces | Besu docs](https://docs.besu-eth.org/public-networks/reference/plugin-api-interfaces)
- [Plugins concept | Besu docs](https://docs.besu-eth.org/public-networks/concepts/plugins)
- [Plugin Services | LF Decentralized Trust wiki](https://lf-hyperledger.atlassian.net/wiki/display/BESU/Plugin+Services)
- [An introduction to Besu's Plugin API | 41North](https://41north.dev/blog/besu/an-introduction-to-hyperledger-besus-plugin-api/)
- [Refactor EVM into a standalone library | LF DT wiki](https://lf-hyperledger.atlassian.net/wiki/spaces/BESU/pages/22154967/Refactor+EVM+into+a+stand+alone+library)
- [HIP-26: Migrate Hedera Smart Contract Service EVM to Besu EVM](https://hips.hedera.com/hip/hip-26)
- [QBFT | Besu docs](https://besu.hyperledger.org/private-networks/how-to/configure/consensus/qbft)
- [IBFT 2.0 | Besu docs](https://besu.hyperledger.org/private-networks/how-to/configure/consensus/ibft)
- [Clique | Besu docs](https://besu.hyperledger.org/private-networks/how-to/configure/consensus/clique)
- [Proof of authority consensus | Besu docs](https://besu.hyperledger.org/private-networks/concepts/poa)
- [Data storage formats (Bonsai) | Besu docs](https://besu.hyperledger.org/public-networks/concepts/data-storage-formats)
- [A Guide To Bonsai Tries | ConsenSys](https://consensys.io/blog/bonsai-tries-guide)
- [Introducing Parallel Transaction Execution in Besu | LF DT](https://www.lfdecentralizedtrust.org/blog/introducing-parallel-transaction-execution-in-hyperledger-besu)
- [A Deep Dive into Besu Milestones: 2025 Highlights and 2026 Goals | LF DT](https://www.lfdecentralizedtrust.org/blog/a-deep-dive-into-besu-milestones-2025-highlights-and-2026-goals)
- [Fusaka Mainnet Announcement | Ethereum Foundation](https://blog.ethereum.org/2025/11/06/fusaka-mainnet-announcement)
- [Ethereum Fusaka locked in for Dec 3 | CoinDesk](https://www.coindesk.com/tech/2025/10/30/ethereum-developers-lock-in-fusaka-upgrade-for-dec-3-with-peerdas-rollout)
- [Prague-Electra (Pectra) | ethereum.org](https://ethereum.org/roadmap/pectra/)
- [Client Diversity](https://clientdiversity.org/)
- [Private transactions (Deprecated) | Besu docs](https://besu.hyperledger.org/private-networks/concepts/privacy/private-transactions)
- [Run Tessera with Besu | Besu docs](https://besu.hyperledger.org/private-networks/how-to/use-privacy/tessera)
- [What Is Hyperledger Besu? Enterprise Ethereum Guide (2026) | ChainLaunch](https://chainlaunch.dev/blog/what-is-hyperledger-besu)
- [Besu | LF Decentralized Trust](https://www.lfdecentralizedtrust.org/projects/besu)
- [Hyperledger CBDC ebook | Linux Foundation](https://www.linuxfoundation.org/hubfs/Hyperledger_CBDC%20ebook_V2.pdf)
- [plugin-api Javadoc](https://javadoc.io/doc/org.hyperledger.besu/plugin-api/latest/index.html)
- [Consensys PluginsAPIDemo](https://github.com/Consensys/PluginsAPIDemo)
