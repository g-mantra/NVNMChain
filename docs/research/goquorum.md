# GoQuorum (Consensys/quorum) — Research Report

**Last verified: 2026-06-24**

## Executive Summary

GoQuorum is an enterprise/permissioned Ethereum client written in Go, originally created by J.P. Morgan and later maintained by ConsenSys. It is a comparatively lightweight **fork of go-ethereum (geth)** that swaps proof-of-work for permissioned consensus (QBFT/IBFT/Raft), adds a public/private dual-state model, and integrates the **Tessera** private transaction manager for on-chain privacy. Its defining characteristic as of mid-2026 is that **it is end-of-life**: the GitHub repository `Consensys/quorum` was **archived (made read-only) on June 5, 2026**, the README explicitly states GoQuorum "is no longer actively maintained or supported by Consensys… We do not recommend using GoQuorum for new projects," and the last functional release (v24.4.1) shipped in **June 2024**. Critically for Ethereum-tooling compatibility, GoQuorum's EVM/chain configuration **caps out at the Berlin hardfork** — it never adopted London (EIP-1559), Shanghai, Cancun, or The Merge. ConsenSys now steers enterprise users to **Hyperledger Besu** (Java, Apache 2.0), with Kaleido as the designated migration partner for legacy Quorum deployments.

---

## Fit Against Requirements

### 1. 1:1 compatibility with the LATEST Ethereum mainnet — **WEAK**
GoQuorum is a geth fork and inherits geth's JSON-RPC surface and Solidity/EVM execution, so a large slice of Ethereum tooling (web3.js, ethers, Solidity, common JSON-RPC calls) works. **But it is far behind mainnet.** Its `params/config.go` `ChainConfig` defines forks only up to **Berlin** (`BerlinBlock`) — there is **no `LondonBlock` (so no EIP-1559 / base-fee / type-2 transactions), no Shanghai, no Cancun (no blobs/EIP-4844, no transient storage), and no Merge/proof-of-stake fields**. The underlying geth base is around the **v1.10.x** line. It also deliberately deviates from stock Ethereum semantics: a **dual public/private state trie**, block validation against a "global public state root," **private transactions** whose payload is replaced by an encrypted-payload hash, and gas pricing disabled by default. Tooling that assumes EIP-1559 fee fields, post-Merge consensus RPC (`engine_*`), or recent opcodes will not work flawlessly. Verdict: **Weak** for "latest mainnet" compatibility — it is frozen several years behind and intentionally non-standard in its privacy/account model.

### 2. Turing-incomplete modules exposed as EVM contracts via precompiles — **PARTIAL**
As a geth fork, GoQuorum supports precompiled contracts exactly the way geth does: precompiles are native Go implementations of the `PrecompiledContract` interface (`RequiredGas(input []byte) uint64` and `Run(input []byte) ([]byte, error)`), registered in per-fork Go maps in `core/vm/contracts.go` (e.g. `PrecompiledContractsBerlin`) keyed by `common.Address`. GoQuorum even ships its own extra precompile (the **Quorum privacy precompile**, `QuorumPrivacyPrecompileContractAddress`), proving the pattern is used for custom native modules. So adding a Turing-incomplete native module and exposing it at a fixed contract address is technically straightforward — implement the interface, add it to the map, recompile. **However**, there is **no plugin/dynamic-registration mechanism for precompiles**: you must fork the source and rebuild your own binary, and because the project is archived you'd be maintaining that fork yourself against an EOL, Berlin-era codebase. (Note: predeploying *contracts* in the genesis file is supported, but that is ordinary EVM bytecode, not a native precompile.) Verdict: **Partial** — the precompile extension model exists and is proven, but only via source fork + recompile on a dead codebase.

### 3. Consensus support (BFT, DPoS, PoS, PoA) — **PARTIAL**
GoQuorum supports four pluggable consensus protocols:
- **QBFT** — Byzantine fault tolerant, proof-of-authority style; tolerates `f` faulty of `3f+1` validators; immediate finality, no forks. Recommended for new networks; **interoperable with Hyperledger Besu's QBFT**.
- **IBFT (Istanbul BFT)** — also BFT/PoA, same `3f+1` tolerance and immediate finality; supported for existing networks, with a migration path to QBFT.
- **Raft** — **crash-fault-tolerant only** (not Byzantine), tolerates up to ~half of nodes failing but assumes the leader acts correctly; fast/low-latency but **not recommended for production**.
- **Clique (PoA)** — proof-of-authority inherited from geth; subject to forks/reorgs.

So it covers **BFT (QBFT, IBFT)** and **PoA (Clique, and QBFT/IBFT are PoA-style)** strongly, and **crash-fault tolerance (Raft)**. **It does NOT support classic Proof-of-Stake or DPoS** — there is no staking/validator-economics consensus and no Merge/PoS engine. Verdict: **Partial-to-Strong on BFT/PoA, but Weak on PoS/DPoS** → overall **Partial** against the full four-way ask.

### 4. Actively maintained — **WEAK (effectively end-of-life)**
- **Repository archived June 5, 2026** (now read-only) at `github.com/Consensys/quorum`.
- README: *"GoQuorum is no longer actively maintained or supported by Consensys. This repository has been archived and remains available for historical reference only. We do not recommend using GoQuorum for new projects."*
- **Last release: v24.4.1, June 2024** (a trivial "update builder base image" patch). Substantive development effectively stopped well before that; ConsenSys publicly deprioritized Quorum around **2023** in favour of MetaMask, Infura, and **Besu**.
- For legacy/commercial customers, ConsenSys appointed **Kaleido** as the preferred migration partner to keep "Quorum Blockchain Service" running, and points new enterprise users to **Hyperledger Besu**.
Verdict: **Weak** — not actively maintained; archived and EOL.

---

## Architecture

- **Implementation language:** Go. GoQuorum is a fork of go-ethereum and is "updated in line with go-ethereum releases" (though that line stopped at roughly geth v1.10.x / Berlin).
- **Key changes vs. geth:**
  - Replaces PoW with **QBFT / IBFT / Raft** (Clique also available).
  - **Permissioned P2P** — connections restricted to known/permitted nodes.
  - **Dual state model** — the state Patricia trie is split into a **public state trie** and a **private state trie**; block validation uses a global *public* state root.
  - **Private transaction handling** — for private txs, the transaction payload is replaced on the public chain by an **encrypted-payload hash**; the actual payload is exchanged off-chain between the relevant parties via the privacy manager.
  - **Gas pricing disabled by default** (gas accounting still exists; can be enabled).
- **Components:** the GoQuorum node (geth fork) + a **privacy manager / private transaction manager** + supporting tooling (quorum-dev-quickstart, Cakeshop/Quorum Explorer historically).

## Compatibility Detail

- **EVM / hardfork ceiling: Berlin.** `ChainConfig` fork fields present include Homestead, DAO, EIP150/155/158, Byzantium, Constantinople, Petersburg, Istanbul, MuirGlacier, **Berlin** — plus placeholder fields (YoloV3, EWASM, Catalyst) but **no production London/Shanghai/Cancun and no `TerminalTotalDifficulty`/Merge support**.
- **No EIP-1559:** transactions are legacy/Berlin-era; type-2 (dynamic-fee) transactions and base-fee semantics are absent.
- **JSON-RPC:** broadly geth-compatible standard namespaces (`eth`, `net`, `web3`, `admin`, `debug`, `txpool`) plus GoQuorum-specific extensions for privacy/permissioning. Post-Merge `engine_*` consensus API is not applicable.
- **Account model:** standard Ethereum accounts, but with the public/private state split and private-state semantics layered on top — a deviation from vanilla Ethereum that some indexers/explorers handle imperfectly.
- **Tooling caveats:** Solidity and basic web3/ethers calls work; newer toolchains that assume EIP-1559 fees or post-Merge behavior (and modern Foundry workflows) can hit friction.

## Precompiles / Extensibility

- Precompiles follow the geth model: native Go types implementing `PrecompiledContract` (`RequiredGas` + `Run`), registered in per-fork maps in `core/vm/contracts.go` keyed by address.
- Standard Ethereum precompiles present through **Berlin** (ecrecover, sha256, ripemd160, identity, modexp w/ EIP-2565, bn256 add/mul/pairing, blake2f) and BLS12-381 set.
- GoQuorum adds the **Quorum privacy precompile** as a custom native contract — direct evidence that adding a Turing-incomplete native module at a fixed address is a supported pattern.
- **Extension mechanism = fork + recompile.** No dynamic plugin registration for precompiles; on an archived codebase this means self-maintaining a fork.

## Consensus

| Protocol | Class | Fault tolerance | Finality | Status |
|---|---|---|---|---|
| **QBFT** | BFT / PoA | `f` of `3f+1` Byzantine | Immediate, no forks | Recommended; Besu-interoperable |
| **IBFT** | BFT / PoA | `f` of `3f+1` Byzantine | Immediate, no forks | Existing networks; migrate to QBFT |
| **Raft** | Crash-fault-tolerant | ~half can fail (leader trusted) | Fast | Not recommended for production |
| **Clique** | PoA | leader/validator rotation | Forks/reorgs possible | Inherited from geth |

- **BFT:** Yes (QBFT, IBFT). **PoA:** Yes (Clique; QBFT/IBFT are PoA-style). **CFT:** Yes (Raft).
- **PoS / DPoS:** **Not supported.** No staking-based or delegated consensus; no Merge/PoS engine.
- v22.4+ introduced "transitions" to change network/consensus config more flexibly; existing IBFT→QBFT and Raft→other migrations are documented.

## Privacy (Tessera / Private Transactions)

- **Tessera** is the production private transaction manager (Java, **Apache 2.0**, separate from the Go node). It encrypts, distributes, and stores private payloads off the public chain.
- Privacy features: **private transactions**, **counterparty protection**, **private state validation (PSV)**, **privacy enhancements**, and **privacy marker transactions (PMTs)** that keep the internal private tx and its receipt off public view behind a public marker tx.
- This privacy model is GoQuorum's main differentiator versus stock geth and a key historical reason for its enterprise adoption.

## Maintenance Status (the headline)

- **Archived June 5, 2026; read-only.** Last release **v24.4.1 (June 2024)**, a cosmetic build patch. Real development effectively ceased years earlier.
- ConsenSys deprioritized Quorum (~2023) toward MetaMask/Infura and **Hyperledger Besu**.
- **Migration path:** ConsenSys/Kaleido recommend moving to **Hyperledger Besu**, the actively maintained enterprise Ethereum client. Besu (Java, Apache 2.0) tracks mainnet hardforks, supports QBFT (interoperable with GoQuorum's QBFT), and is the industry-preferred client. APIs/config are similar but **not a drop-in replacement**.

## License

- **GoQuorum node:** inherits go-ethereum licensing — **GNU LGPL v3** for the library and **GNU GPL v3** for the full `geth`/node binaries.
- **Tessera:** **Apache License 2.0**.

## Notable Enterprise Deployments (historical)

- **komgo** — commodity trade-finance network backed by ~15 major global banks/trading houses, built on Quorum.
- **Project Ubin** (Monetary Authority of Singapore) — interbank RTGS settlement PoC using Quorum with transaction privacy and settlement finality.
- **Quorum Blockchain Service (QBS)** — managed offering (Azure Marketplace via the quorum-dev-quickstart), now continued through Kaleido for legacy customers.

## Relationship to Hyperledger Besu

- Both are/were ConsenSys enterprise Ethereum clients. **GoQuorum**: Go, geth fork, LGPL/GPL, Berlin-era, EOL/archived. **Besu**: Java, Apache 2.0, actively maintained, mainnet-current.
- **QBFT is interoperable across GoQuorum and Besu**, easing migration of the consensus layer.
- ConsenSys's clear current recommendation: **use Besu for new enterprise EVM chains**; GoQuorum exists only for historical reference and legacy maintenance.

---

## Overall Verdict

GoQuorum is a **dead-end choice for a new full-stack EVM chain in 2026.** Its consensus story (QBFT/IBFT BFT + Raft CFT + Clique PoA) is solid and its privacy model is genuinely differentiated, and its geth lineage means the precompile extension pattern is proven. But it **fails the two most important criteria**: it is **not maintained** (archived June 2026, last real release June 2024) and it is **not compatible with modern Ethereum** (frozen at Berlin — no EIP-1559, no Shanghai/Cancun, no PoS). For the stated goals (latest-mainnet tooling compatibility, extensible precompiles on a living codebase, broad consensus including PoS/DPoS, active maintenance), **Hyperledger Besu is the strictly better successor**, and is what ConsenSys itself now recommends.

---

## Sources

- [Consensys/quorum — GitHub repository (archived notice + README maintenance statement)](https://github.com/Consensys/quorum)
- [Consensys/quorum — Releases (v24.4.1, June 2024)](https://github.com/Consensys/quorum/releases)
- [Consensys/quorum — v24.4.1 release tag](https://github.com/Consensys/quorum/releases/tag/v24.4.1)
- [Consensys/quorum — params/config.go (ChainConfig fork fields, Berlin ceiling)](https://github.com/Consensys/quorum/blob/master/params/config.go)
- [Consensys/quorum — core/vm/contracts.go (precompiles, Quorum privacy precompile)](https://github.com/Consensys/quorum/blob/master/core/vm/contracts.go)
- [GoQuorum docs — Welcome / Overview](https://docs.goquorum.consensys.io/)
- [GoQuorum docs — Architecture (public/private state, geth fork)](https://github.com/Consensys/doc.goquorum/blob/main/docs/concepts/architecture.md)
- [GoQuorum docs — Consensus protocols (comparing PoA: QBFT/IBFT/Raft/Clique)](https://raw.githubusercontent.com/Consensys/doc.goquorum/main/docs/concepts/consensus/comparing-poa.md)
- [GoQuorum docs — Support plans / maintenance](https://docs.goquorum.consensys.io/support)
- [GoQuorum docs — Privacy concepts](https://github.com/Consensys/doc.goquorum/blob/main/docs/concepts/privacy-index.md)
- [GoQuorum docs — Predeploy a contract in the genesis file](https://docs.goquorum.consensys.io/configure-and-manage/configure/genesis-file/contracts-in-genesis/)
- [Consensys Tessera docs (Apache 2.0 private transaction manager)](https://docs.tessera.consensys.net/)
- [Kaleido — Migrating from Quorum to Hyperledger Besu (Consensys deprioritization, Kaleido migration partner)](https://www.kaleido.io/blockchain-blog/migrating-from-quorum-to-hyperledger-besu)
- [Web3 Labs — A comparison of Ethereum clients (Besu vs GoQuorum, languages/licenses)](https://blog.web3labs.com/a-comparison-of-ethereum-clients/)
- [Consensys/quorum Discussion #1293 — How Besu compares to GoQuorum](https://github.com/Consensys/quorum/discussions/1293)
- [Consensys — A Year in Review: Consensys Quorum Highlights](https://consensys.io/blog/a-year-in-review-consensys-quorum-highlights)
- [Consensys — komgo case study](https://consensys.io/blockchain-use-cases/finance/komgo)
- [Consensys — Project Ubin case study](https://consensys.io/blockchain-use-cases/finance/project-ubin)
- [Consensys/quorum PR #1249 — Upgrade Go-Ethereum release v1.10.0](https://github.com/Consensys/quorum/pull/1249)
