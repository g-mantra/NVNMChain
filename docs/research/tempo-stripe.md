# Tempo (Stripe / Paradigm) — Full-Stack EVM Payments L1

**Last verified: 2026-06-25**

## Executive summary

Tempo is a payments-focused, EVM-compatible Layer-1 blockchain incubated by **Stripe** and **Paradigm**, with a mainnet launch on **March 18, 2026**. Its stack pairs a **Reth-SDK / revm** execution layer with **Commonware's Simplex BFT** consensus, targeting the **Osaka** Ethereum hardfork (the Fusaka-era EVM), sub-second (~0.6s) deterministic finality, and very high throughput. Its defining payments features are protocol-enshrined: **no native gas token** (fees are paid in any USD stablecoin via an enshrined Fee AMM), the **TIP-20** stablecoin token standard, a custom **EIP-2718 transaction type (0x76)** with WebAuthn/P256 smart-account features, and a suite of **protocol-level precompiles/system contracts** (TIP-20 Factory, Fee Manager, Stablecoin DEX, signature-verification precompile TIP-1020, etc.). Crucially, the reader must distinguish **two things**: (a) **Tempo the network** — a single, currently **permissioned** L1 whose validator set is run by Tempo and a small set of named institutions (Visa, Stripe, Standard Chartered's Zodia Custody), which you *join/build on* but do not control; and (b) **the underlying stack** — Reth SDK + revm (Apache/MIT, open) and Commonware (Apache, open Rust library) — which **is reusable** to build your own chain. Tempo's own node software is open-sourced (Apache-2.0/MIT) on GitHub, but it is engineered to run *the Tempo network*, not marketed as a generic "launch-your-own-payments-chain" framework. For someone evaluating implementations to build their own chain, the interesting, reusable artifacts are the components beneath Tempo, not Tempo itself.

---

## Fit against requirements

### 1. 1:1 compatibility with the latest Ethereum mainnet — **Partial-to-Strong (with payments-specific deviations)**
Tempo is built on the **Reth SDK** with a **revm** execution core and explicitly **targets the Osaka hardfork** (the Fusaka-generation EVM — newer than Cancun/Prague), and its docs state "all Ethereum JSON-RPC methods work out of the box" with Solidity, Foundry, Hardhat and standard tooling working directly. That makes baseline tooling compatibility strong. However, it is **deliberately not a drop-in Ethereum clone**: there is **no native gas currency** (you pay fees in USD stablecoins, selected per-transaction), it adds a **custom EIP-2718 transaction type (0x76)** with WebAuthn/P256 auth, batching, fee sponsorship and partitioned/expiring nonces, and it changes block structure (separate gas pools / "payment lanes"). Wallets that assume an ETH-like native balance for gas need adaptation, and the smart-account transaction type is non-standard. Verdict: **Partial** for strict 1:1 parity, **Strong** for "existing EVM contracts and RPC tooling work." MetaMask is listed as a partner, so wallet support exists in practice — but via Tempo-specific integration, not pure vanilla behavior.

### 2. Turing-incomplete modules exposed as EVM contracts via precompiles — **Strong (as a stack capability) / Weak (for third parties on Tempo mainnet)**
This is exactly what Tempo does, and it validates the Reth/revm precompile-injection approach. The `tempo_precompiles` Rust crate registers a rich set of protocol precompiles into revm's `PrecompilesMap` via `extend_tempo_precompiles()`: TIP-20 Factory, Fee Manager, Stablecoin DEX (an enshrined CLOB), Account Keychain, Nonce Manager, Validator Config, Storage Credits, TIP-403 policy registry, Address Registry, Receive Policy Guard, and the **TIP-1020 signature-verification precompile** (secp256k1/P256/WebAuthn at `0x5165...`). These are fixed dollar-cost, deterministic, non-Turing-complete native modules surfaced to Solidity. **But**: they are **compile-time, protocol-level precompiles baked into the node software** — third parties **cannot** register their own precompiles at runtime on Tempo mainnet (you'd have to fork the node and run your own chain). So if the requirement is "the *stack* lets me add precompiles," that is **Strong** (Reth/revm + Tempo's pattern proves it). If the requirement is "*I* can add precompiles to the live Tempo network," that is **Weak** — only the Tempo protocol team can.

### 3. Consensus support (BFT / DPoS / PoS / PoA) — **Partial (it is one specific BFT/permissioned-PoA model, not a configurable menu)**
Tempo uses **Simplex BFT consensus via the Commonware library** over a **bounded validator set**, giving deterministic ~0.6s finality with no reorgs and graceful degradation under adverse networks. The validator set is currently **permissioned/approved** (effectively PoA-style at launch): the testnet ran ~4 validators operated by the Tempo team; mainnet's first external validators are **Visa** (running its node in-house), **Stripe**, and **Standard Chartered's Zodia Custody**, with institutions like UBS, Mastercard and Kalshi as partners. The stated roadmap is to transition to a **neutral, permissionless Proof-of-Stake** network "over time," but validator economics for that phase are **not yet defined** and no timeline is published. So Tempo natively supports **BFT + permissioned PoA today, PoS later** — it does **not** offer DPoS or a pluggable choice of consensus. The underlying **Commonware** library, however, is a modular consensus toolkit ("the anti-framework") that you could use to build different models — that flexibility is a property of the stack, not of the Tempo network. Verdict: **Partial**.

### 4. Actively maintained — **Strong (as a network) / Caveated (as a reusable framework)**
Very actively developed and well funded. Mainnet went live **March 18, 2026**; the GitHub `tempoxyz/tempo` repo (Rust, ~964 stars, Apache-2.0/MIT) shows ongoing network upgrades (T2–T7: storage credits, dynamic base fee, reserve channels, DEX/routing improvements, access-key/policy features). Backing: **$500M Series A at a $5B valuation** (Oct 2025, led by Thrive Capital and Greenoaks; Sequoia, Ribbit, SV Angel). Maintained by the Tempo team with Paradigm; Stripe provides the payments context (it owns stablecoin firm **Bridge** and wallet/auth firm **Privy**). 40+ infrastructure partners (MetaMask, Phantom, wallets, RPC providers). **Caveat for self-builders**: this is a **corporate-backed single network**, not a productized "deploy your own chain" framework. The code is open and re-usable in principle, but maintenance effort is directed at running Tempo, not at supporting downstream forks. Verdict: **Strong** for the network's health; treat reusability as "open-source components you self-integrate."

---

## Architecture

- **Execution layer:** Reth SDK (Paradigm's high-performance Rust Ethereum client) with a **revm** EVM core. Engineers include revm/Reth contributors (e.g. Dragan Rakita). Targets the **Osaka** hardfork. Full EVM + JSON-RPC compatibility claimed.
- **Consensus layer:** **Simplex BFT** from the **Commonware** library — leader-based, notarization + nullification (vote a "null block" on timeout), deterministic finality (~0.6s), no chain reorgs.
- **Performance:** Marketing claims >100,000 TPS and sub-second finality; testnet figures cited around 20k TPS with a roadmap to higher. Treat the 100k figure as a target/benchmark, not an independently verified mainnet number.
- **Block structure:** separate `gasLimit` pools for general EVM execution vs. reserved **payment lanes** / system sub-blocks, so payment traffic can't be starved during congestion.
- **Languages / license:** Rust (primary, ~80%), Solidity (contracts), TypeScript/Go/Python SDKs. **Dual-licensed Apache-2.0 / MIT**. Repos: `tempo`, `tempo-std`, `tempo-go`, `pytempo`, `accounts`, `wallet-rs`, `zones` (private chains anchored to Tempo), `mpp-specs`, `docs`.
- **Self-deployability:** node runs via prebuilt binary, source build, or Docker (testnet chain ID 42431). The software is open, but it is built to operate the Tempo network rather than as a generic chain-launcher.

## Ethereum compatibility detail

- Tracks **Osaka** (Fusaka-era), i.e. ahead of Cancun/Prague — relatively current.
- "All Ethereum JSON-RPC methods work out of the box"; Solidity/Foundry/Hardhat/ethers.js usable.
- **Deviations:** no native gas token; fees paid in USD-denominated **TIP-20** stablecoins via an enshrined **Fee AMM** (fixed-rate swaps, batch/end-of-block settlement for MEV protection); custom tx type **0x76** with WebAuthn/P256 smart accounts, fee sponsorship, batching, expiring/partitioned nonces. These require wallet/integration awareness, so it is not strictly byte-for-byte "vanilla Ethereum."

## Precompiles & system contracts

Protocol-level precompiles registered into revm at fixed addresses (compile-time, via `extend_tempo_precompiles()`):
- **TIP-20 Factory** (deterministic token deploys), **Fee Manager**, **Stablecoin DEX** (enshrined singleton CLOB), **Account Keychain** (session keys + spend limits), **Nonce Manager**, **Validator Config (v1/v2)**, **Storage Credits**, **TIP-403** policy/compliance registry, **Address Registry** (virtual addresses), **Receive Policy Guard**, **TIP-1020 Signature Verification** (secp256k1/P256/WebAuthn, reverts on invalid).
- Predeployed standard utilities: Multicall3, CreateX, Permit2, Arachnid Create2 factory, Safe deployer.
- **Anyone can deploy ordinary Solidity contracts and create TIP-20 tokens permissionlessly via the Factory**, but **only the protocol can add precompiles** — third parties cannot inject their own native modules on Tempo mainnet.

## Consensus & decentralization (candid)

- **Permissioned at launch.** Approved validator set. First external validators: Visa (in-house node), Stripe, Standard Chartered/Zodia Custody; testnet was ~4 Tempo-run validators.
- Validators are currently rewarded in **stablecoin fees**, not a volatile native token.
- Roadmap to **permissionless PoS** is stated but **undated and economically undefined** ("not yet clear how validators will be incentivized").
- Honest read: today this is closer to a **consortium / PoA institutional network** than a decentralized chain. Decentralization is aspirational.

## Maintenance, funding & Stripe context

- **Mainnet live 18 March 2026**, launched alongside the **Machine Payments Protocol (MPP)** — an open standard for AI-agent-to-service payments (a "sessions" primitive for streamed micropayments without per-interaction on-chain txs).
- **$500M Series A, $5B valuation** (Oct 2025; Thrive Capital, Greenoaks, Sequoia, Ribbit, SV Angel).
- Stripe context: acquired **Bridge** (stablecoin orchestration) and **Privy** (wallets/auth); Tempo is the chain-layer of Stripe's stablecoin strategy. Design partners: Visa, Mastercard, Deutsche Bank, Standard Chartered, Revolut, Nubank, Shopify, OpenAI, Anthropic, Ramp, DoorDash, UBS, Klarna, Kalshi.

## The stack vs. the network (key distinction for self-builders)

- **Reusable stack (build your own chain):** **Reth SDK + revm** (Apache/MIT) for execution, and **Commonware** (open Apache Rust library, modular consensus incl. Simplex/Threshold-Simplex) for consensus. Tempo itself is a **reference proof** that this combination yields a fast, precompile-rich EVM L1. You can adopt these libraries directly and design your own validator/consensus/fee model.
- **Not reusable as-is:** **Tempo the network** — a single hosted, permissioned L1 you join or build apps on. Its payments precompiles, no-native-token fee model and validator set are properties of *that* network. Tempo's node repo is open, so a determined team could fork it, but it is not positioned or supported as a turnkey "launch-a-Tempo-clone" framework.

## Pros / Cons against the 4 criteria

**Pros**
- Modern EVM (Osaka), strong baseline tooling/JSON-RPC compatibility.
- Demonstrates exactly the precompile-injection pattern on a Reth/revm base (criterion 2 validated at the stack level).
- Fast deterministic BFT finality via a clean, open consensus library (Commonware).
- Extremely well funded, actively maintained, top-tier institutional partners, open-source (Apache/MIT).

**Cons**
- Payments-specific deviations (no native gas token, custom tx type) break strict 1:1 Ethereum parity.
- Precompiles are protocol-fixed; third parties can't add native modules on the live network.
- Consensus is a single fixed model (Simplex BFT), not a configurable BFT/DPoS/PoS/PoA menu.
- Currently permissioned/consortium; decentralization roadmap is undated and economically undefined.
- It is a **network**, not a self-deploy framework — the reusable value is in the underlying Reth + Commonware components, which you'd integrate yourself.

---

## Sources

- [Tempo — official site](https://tempo.xyz/)
- [Tempo Documentation](https://docs.tempo.xyz/)
- [Tempo docs — Performance](https://docs.tempo.xyz/learn/tempo/performance)
- [Tempo docs — Predeployed Contracts](https://docs.tempo.xyz/quickstart/predeployed-contracts)
- [Tempo Improvement Proposals (TIPs)](https://docs.tempo.xyz/protocol/tips)
- [TIP-1020: Signature Verification Precompile](https://tips.sh/1020)
- [tempo_precompiles — Rust crate docs](https://rustdocs.tempo.xyz/tempo_precompiles/index.html)
- [GitHub — tempoxyz/tempo](https://github.com/tempoxyz/tempo)
- [GitHub — tempoxyz org](https://github.com/tempoxyz)
- [GitHub — tempoxyz/tempo-std](https://github.com/tempoxyz/tempo-std)
- [Commonware — official site](https://commonware.xyz/)
- [CoinDesk — Stripe-led Tempo goes live with AI agent protocol (Mar 18, 2026)](https://www.coindesk.com/tech/2026/03/18/stripe-led-payments-blockchain-tempo-goes-live-with-protocol-for-ai-agents)
- [Ledger Insights — Stripe, Paradigm launch Tempo + MPP](https://www.ledgerinsights.com/stripe-paradigm-launch-tempo-blockchain-alongside-machine-payments-standard/)
- [Ledger Insights — Visa becomes validator on Tempo](https://www.ledgerinsights.com/visa-becomes-validators-on-tempo-blockchain-co-founded-by-stripe/)
- [The Defiant — Tempo goes live on mainnet, unveils MPP](https://thedefiant.io/news/blockchains/tempo-launches-mainnet-unveils-machine-payments-protocol-with-stripe)
- [The Defiant — Tempo launches public testnet](https://thedefiant.io/news/blockchains/payments-focused-tempo-blockchain-launches-public-testnet)
- [CoinGecko — What Is Tempo (stablechain)](https://www.coingecko.com/learn/what-is-tempo-stablechain)
- [DL News — Stripe-backed Tempo public testnet](https://www.dlnews.com/articles/markets/stripe-backed-tempo-blockchain-launches-public-testnet/)
- [crypto.news — Tempo mainnet live for machine payments](https://crypto.news/stripe-and-paradigms-tempo-mainnet-goes-live-for-machine-payments/)
- [Sentora/Medium — Technical observations about Tempo](https://medium.com/sentora/fine-tuning-the-world-computer-on-payments-some-technical-observations-about-tempo-1d1bfa4a70d5)
- [Medium (J. Rodriguez) — Inside Tempo's Machine Payment Protocol](https://jrodthoughts.medium.com/the-architecture-of-autonomous-wealth-inside-tempos-machine-payment-protocol-c14071a420f1)
