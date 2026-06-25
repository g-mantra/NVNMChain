# ADR-0001: EVM Chain Implementation for a Sovereign Stablecoin L1

- **Status:** Proposed
- **Date:** 2026-06-25
- **Deciders:** Platform / Protocol Engineering
- **Related:** Research reports in this directory (see [README.md](docs/research/README.md)); validation plan in [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md)

---

## 1. Context

We are building a **sovereign Layer-1 blockchain** with an EVM execution environment. It must behave like Ethereum for the entire existing tooling and contract ecosystem, let us add bespoke deterministic native functionality exposed to Solidity, run on a validator set and consensus model we control, and — as a product requirement — let users **pay gas in any stablecoin**.

We need to choose the full-stack implementation (consensus + EVM execution + precompiles) to build on. Twelve implementations were researched in depth; this ADR distils that research into a decision. The per-project evidence lives in the linked reports.

## 2. Requirements

All five requirements **R1–R5 are weighted equally** (20% each) when comparing options.

| # | Requirement | What "good" looks like |
|---|---|---|
| **R1** | **1:1 compatibility with latest Ethereum mainnet** | Tracks the newest fork (Prague/Pectra → Fusaka/Osaka); full standard JSON-RPC; MetaMask / Hardhat / Foundry / ethers / viem work unmodified; minimal, documented deviations. |
| **R2** | **Turing-incomplete native modules as precompiles** | A supported path to add custom deterministic native modules and expose them as EVM contracts at fixed addresses. See the deep-dive: [precompiles-turing-incomplete-modules.md](docs/architecture/precompiles-turing-incomplete-modules.md). |
| **R3** | **Pluggable consensus: BFT, DPoS, PoS, PoA** | A configurable validator/consensus model spanning these families, with fast deterministic finality. |
| **R4** | **Actively maintained** | Current releases, healthy cadence, identifiable maintainer, production track record, viable license. |
| **R5** | **Gas payable in any stablecoin** | Users pay fees in any USD stablecoin with seamless UX, ideally with **no (or optional) native gas token** at the protocol level. Research ([stablecoin-gas-mechanisms.md](docs/architecture/stablecoin-gas-mechanisms.md)) shows only a **protocol-native (enshrined) fee module** scores high here; ERC-4337 / EIP-7702 paymasters keep a native token + relayer infra and are a weaker, app-layer workaround. |

**Hard constraints (gate the decision regardless of requirement scores):**

- **Sovereign L1.** The implementation must launch an L1 with its own validator set — not an L2/L3 rollup that derives ordering from a sequencer and security from another chain.
- **Trust model is open.** Permissionless or permissioned/consortium validator sets are both acceptable; trust model is not a deciding factor.

## 3. Considered Options

**In scope** (sovereign-L1 capable EVM stacks):

1. **Reth SDK + Commonware** — Paradigm's Rust execution client (as a library) + Commonware Simplex BFT consensus, joined by a custom glue layer. Bootstrappable from Stripe's open-source **Tempo** node.
2. **Cosmos EVM** (`cosmos/evm`, ex-Evmos/evmOS lineage) — Cosmos SDK + CometBFT + a Cosmos go-ethereum fork.
3. **Avalanche Subnet-EVM / Coreth** (`ava-labs`) — AvalancheGo VM with Snowman consensus.
4. **Polkadot Frontier** (`polkadot-evm/frontier`) — EVM emulation pallets on Substrate / Polkadot-SDK.
5. **Hyperledger Besu** — Java enterprise Ethereum client.
6. **Berachain BeaconKit** — CometBFT consensus client paired with an unmodified EL.

**Out of scope** (documented for completeness):

- **OP Stack, Arbitrum Orbit, Polygon CDK** — L2/L3 rollup frameworks: sequencer-based ordering, security inherited from an L1. Fail the sovereign-L1 + pluggable-consensus constraints. (Strong choices if we ever retarget to an L2.)
- **Tempo (Stripe + Paradigm)** — a single, currently permissioned payments network, not a self-deployable framework. Not a deploy target, but its open-source node is the **bootstrap reference** for Option 1 (see §7).
- **Commonware (alone)** — a primitives toolkit that ships no EVM; folded into Option 1.
- **Polygon Edge** (archived Dec 2024) and **GoQuorum** (archived Jun 2026, frozen at the Berlin fork) — end-of-life.

## 4. Decision

**Adopt Option 1 — Reth SDK + Commonware, bootstrapped from Tempo (`tempoxyz/tempo`) — retaining and adapting Tempo's enshrined stablecoin-fee module.**

It is the **highest-scoring option overall** (4.60 vs. 4.00 for the next best — see §5), strong on every requirement and uniquely strong on R5:

- **R5 is already built and forkable.** Tempo runs enshrined, no-native-token, any-USD-stablecoin gas (a `0x76` transaction type carrying a `fee_token`, a FeeManager precompile, and a fixed-rate Fee AMM), open source under Apache-2.0/MIT. We keep and adapt this module rather than build it. This is the single biggest differentiator versus the other top options, which score low on R5.
- **R1 is best-in-class.** Reth is a genuine mainnet client current to the newest fork, with full standard JSON-RPC — flawless tooling parity.
- **R2 is clean.** Custom Turing-incomplete precompiles inject into revm via Reth's EVM-config path and execute inside the deterministic block executor.
- **R3 is covered.** Commonware Simplex gives deterministic sub-second BFT finality; PoS/DPoS/PoA are thin validator-selection layers we write over the validator set.
- **R4 is solid.** Reth and Commonware are both actively maintained; Tempo proves the combination in production.

For the stablecoin-eligibility design, use **Tempo's model** (bespoke TIP-20 token standard, structural USD eligibility, fixed-rate MEV-free AMM) for the fastest fork, or **Celo's `FeeCurrencyDirectory` + per-token oracle + decimal-adapter** design (CIP-64) if we need to support *arbitrary existing ERC-20* stablecoins (real USDC/USDT) rather than a bespoke standard.

**Alternatives, in order of preference:**

- **Cosmos EVM** — *fallback if we cannot staff "owning the stack."* Top-tier on R1–R4 (turnkey, upstream-maintained, best-in-class precompiles), but its low R5 score pulls it to 4.00: it has no EVM-native arbitrary-ERC-20 gas, so R5 would rely on ERC-4337/EIP-7702 paymasters (keeping a native token). Choose only if the seamless no-native-token UX can be sacrificed.
- **Avalanche Subnet-EVM** — also 4.00; strong on R2/R3/R4 with turnkey PoA↔PoS↔DPoS, but trails mainnet (Cancun, no Pectra) and has no native R5 precedent.

This ADR is **Proposed**, pending the [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md) validation and stakeholder sign-off.

## 5. Rationale (scoring)

Scores: 1 (weak) – 5 (strong), averaged across all five requirements with equal (20%) weight.

| Option | R1 | R2 | R3 | R4 | R5 | **Avg** |
|---|:--:|:--:|:--:|:--:|:--:|:--:|
| **Reth + Commonware (Tempo)** | 5.0 | 4.5 | 4.0 | 4.5 | 5.0 | **4.60** |
| **Cosmos EVM** | 4.5 | 5.0 | 4.5 | 4.0 | 2.0 | **4.00** |
| **Avalanche Subnet-EVM** | 3.5 | 5.0 | 4.5 | 4.5 | 2.5 | **4.00** |
| **Polkadot Frontier** | 3.5 | 4.5 | 5.0 | 3.5 | 2.5 | **3.80** |
| **Hyperledger Besu** | 5.0 | 2.0 | 3.5 | 5.0 | 2.0 | **3.50** |
| **Berachain BeaconKit** | 5.0 | 1.5 | 3.0 | 4.5 | 2.0 | **3.20** |

R5 scores reflect the stablecoin-gas analysis: **5.0** for an enshrined, no-native-token model with a production precedent (Reth+Commonware/Tempo); **~2.0–2.5** for stacks that can only meet R5 via ERC-4337/EIP-7702 paymasters or unprecedented custom client work.

With all five requirements weighted equally, **Reth+Commonware leads outright at 4.60.** Cosmos EVM and Avalanche tie for second at 4.00 — both excellent on R1–R4 but held back by weak R5. The deciding margin is almost entirely R5: the enshrined-fee-module path is the only one that delivers seamless any-stablecoin gas without a native token, and its main cost (owning the integration) is contained because Tempo's open-source `crates/consensus` *is* the consensus↔execution glue, forkable today (see Appendix A).

## 6. Per-option analysis

### 6.1 Reth SDK + Commonware (chosen)
**Architecture.** Reth (revm-based) embedded as a library for execution + Commonware Simplex BFT for consensus, joined by a custom glue layer — the Tempo architecture. Verified from `tempoxyz/tempo` source (commit `9f2041c`):
- Reth is built via the SDK node builder with custom `ExecutorBuilder`, `ConsensusBuilder` (replaces Ethereum PoS), `PayloadBuilder`, `PoolBuilder`.
- Consensus drives execution **in-process** using Engine-API data types (`ForkchoiceState`, `PayloadAttributes`, `ConsensusEngineHandle.new_payload`); the external Engine-API RPC is disabled (`NoopEngineApiBuilder`).
- An `application` actor implements Commonware's `Automaton` (propose/verify payloads); an `executor` actor applies finalized forkchoice. Finalized, totally-ordered blocks arrive via Commonware's `marshal`; finality is single-shot (no reorgs) via BLS12-381 threshold certificates.
- Custom precompiles inject via `ConfigureEvm` → `EvmFactory` → revm `PrecompilesMap`, inside the deterministic `BlockExecutor`.

**Pros.** Best mainnet fidelity (R1) and clean revm precompiles (R2); enshrined any-stablecoin gas with a forkable precedent (R5); sub-second deterministic BFT (R3); full sovereignty, no SDK lock-in; the glue is open source (Apache-2.0/MIT), not greenfield.

**Cons / risks.** You **own the stack forever** — every Reth and Commonware upgrade is your merge. Commonware consensus is self-described **ALPHA** (fast-churning APIs; Minimmit, its faster successor, is spec-only). Tempo is **unaudited** (per its own note), runs ~4 validators, and its performance figures (~0.6s finality, ~16k RPS) are vendor/secondary. Tempo is payments-coupled — you keep the fee mechanism but strip Stripe-specific code and still build staking, genesis, governance, DKG reconfiguration, and bridging/DA yourself. Requires a strong Rust platform team.

### 6.2 Cosmos EVM (fallback)
**Architecture.** Cosmos SDK app-chain + CometBFT + a Cosmos go-ethereum fork; native SDK modules surfaced via stateful precompiles.

**Pros.** Best-in-class R2 (staking, bank, distribution, gov, IBC, ERC-20 as precompiles; custom precompiles via a documented keeper interface). R3 strong (CometBFT BFT, ~2s finality; native PoS/DPoS). R1 good (Prague/Pectra, EIP-1559/2930, EIP-7702; near-complete JSON-RPC). R4 active (v0.7.0, May 2026), with a large sovereign-L1 ecosystem and IBC as a bonus.

**Cons / risks.** **Weakest on R5 (scored 2.0):** `x/feemarket` charges gas in the native denom; "fee abstraction" is a Cosmos-layer IBC swap; `x/erc20` only wraps tokens — there is no precedent for charging EVM gas directly in arbitrary ERC-20 stablecoins, so R5 falls back to paymasters (keeping a native token). Also: still **pre-v1.0**; the precompile surface produced two critical CVEs in ~12 months (incl. an ICS-20 nested-execution bug tied to a ~$7M loss elsewhere); minor R1 deviations (non-burning base fee, no EIP-4844 blobs, dual `0x`/bech32 accounts); turnkey PoA only via manual permissioning.

### 6.3 Avalanche Subnet-EVM
**Pros.** Excellent R2 (native-Go stateful precompiles + `precompilegen` + the `precompile-evm` repo, no fork needed). Best turnkey R3: leaderless Snowman BFT with sub-second finality, and on-chain ValidatorManager switching PoA↔PoS↔DPoS post-Etna. Actively maintained (consolidated into the AvalancheGo monorepo Dec 2025).

**Cons.** Weakest viable R1: targets **Cancun**, no Pectra/Prague yet (devs must pin `evmVersion=cancun`). No native R5 precedent. Avalanche-specific networking/consensus assumptions.

### 6.4 Polkadot Frontier
**Pros.** Best R3 (Aura/PoA, BABE+GRANDPA/PoS, NPoS/DPoS, PoW); mature pallets-as-precompiles (R2); reaches Prague in the interpreter.
**Cons.** EVM **emulation** (account-model/gas/block-synthesis deviations) → weaker R1; Parity's strategic EVM investment is shifting to REVM + `pallet-revive` (Polkadot Hub), an R4 strategic risk; no native R5 model.

### 6.5 Hyperledger Besu
**Pros.** Genuine mainnet client current to Fusaka/PeerDAS (R1 = 5); QBFT/IBFT2 BFT-PoA with immediate finality; impeccable maintenance (R4 = 5); strong permissioned heritage (eNaira, mBridge, Linea, Hedera).
**Cons.** **No plugin hook for custom precompiles** — requires a maintained fork (R2 weak); no DPoS, PoS only as a mainnet execution layer (R3 limited); no native R5 model.

### 6.6 Berachain BeaconKit
**Pros.** "EVM-identical" via an unmodified EL over the Engine API (R1 = 5); instant single-slot BFT finality; production (Berachain).
**Cons.** Custom precompiles **deliberately removed** to preserve EVM-identity — native logic must live in ordinary system contracts (R2 very weak); no DPoS/PoA flexibility (R3 limited); BUSL-1.1 license; no native R5 model.

## 7. Consequences

### Positive
- R5 satisfied natively with a forkable production precedent; seamless any-stablecoin gas, no native token required.
- Best mainnet fidelity and clean revm custom precompiles (R1 + R2).
- Sub-second deterministic BFT finality; full sovereignty; no SDK lock-in.
- The consensus↔execution glue is forkable today, not greenfield.

### Negative / to manage
- **We own the stack forever** — staff a dedicated Rust platform team; every Reth/Commonware bump is our merge.
- **Immature dependencies/reference** — Commonware consensus is ALPHA; Tempo is unaudited and lightly validated. Treat as a blueprint requiring our own audit and a validator-set expansion beyond ~4.
- We still **build** staking/validator-selection (PoS/DPoS/PoA), genesis, governance/upgrades, DKG reconfiguration, and bridging/DA.
- The fee-eligibility model (Tempo TIP-20 vs. Celo `FeeCurrencyDirectory`) must be decided up front; each carries different governance/oracle surface.

## 8. Risks & mitigations (chosen option)

| Risk | Severity | Mitigation |
|---|---|---|
| Commonware consensus is ALPHA; API churn | High | Pin `commonware-*` to Tempo's revisions; controlled upgrade cadence; design for Simplex (not Minimmit); contribute upstream. |
| Tempo reference is unaudited / lightly validated | High | Independent audit of the forked glue, fee module, and all custom precompiles before mainnet; grow the validator set well beyond 4. |
| Perpetual ownership of the integration | Med–High | Dedicated Rust platform team; CI tracking upstream Reth/Commonware; budget recurring merge effort. |
| Fee-module correctness (R5 is value-bearing) | High | Audit FeeManager / Fee-AMM swap math and `0x76` handling; fuzz fee accounting; cap/whitelist eligible tokens initially. |
| Stablecoin de-peg / fee-AMM liquidity risk | Medium | Conservative peg bands; per-token liquidity floors; governance kill-switch for a failing fee token. |
| Custom precompile vulnerability | Medium | Mandatory audit per precompile; conservative gas; determinism tests inside `BlockExecutor`. |

## 9. Validation & next steps

1. **Run [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md)** (3-week timebox, go/no-go gate): fork `tempoxyz/tempo`, stand up a ≥4-node devnet, and prove (a) newest-fork behavior + the standard-tooling matrix, (b) **end-to-end stablecoin gas via the `0x76` tx + FeeManager** (the R5 proof), (c) Commonware Simplex finality timing. This converts the §5 effort estimate into a measured number.
2. **Decide the fee-eligibility model** (Tempo TIP-20 vs. Celo `FeeCurrencyDirectory`) and whether to run **no native token** or an optional one for staking/incentives.
3. **Prototype one custom domain precompile** via `ConfigureEvm`/`PrecompilesMap`; commission a security review of the precompile + fee-module pattern.
4. **Design the staking/validator-selection module** feeding Commonware's `Supervisor`; confirm the consensus family (PoS/DPoS/PoA) and validator-set policy.
5. **Contingency:** if the spike shows the ownership/maturity burden is unacceptable, fall back to Cosmos EVM and accept ERC-4337/EIP-7702 paymasters for R5.
6. Move this ADR from **Proposed** to **Accepted**.

## 10. References

Detailed, source-cited research reports (verified 2026-06-24/25):

- **Chosen stack:** [Reth SDK](docs/research/reth-sdk.md) · [Commonware](docs/research/commonware.md) · [Tempo (Stripe)](docs/research/tempo-stripe.md) · [Tempo as a blueprint](docs/architecture/tempo-as-blueprint.md) · [Commonware↔Reth glue mechanics](docs/architecture/commonware-reth-glue.md)
- **R5 — stablecoin gas:** [Stablecoin gas mechanisms](docs/architecture/stablecoin-gas-mechanisms.md) (Tempo enshrined model, Celo fee-currencies, Cosmos fee-abstraction, ERC-4337/7702 paymasters)
- **R2 — precompiles:** [Turing-incomplete modules via precompiles](docs/architecture/precompiles-turing-incomplete-modules.md) (concept, gas/determinism rules, per-stack mechanics, precompile vs. system-contract vs. alt-VM)
- **Alternatives:** [Cosmos EVM](docs/research/cosmos-evm.md) · [Avalanche Subnet-EVM](docs/research/avalanche-subnet-evm.md) · [Polkadot Frontier](docs/research/polkadot-frontier.md) · [Hyperledger Besu](docs/research/hyperledger-besu.md) · [Berachain BeaconKit](docs/research/berachain-beaconkit.md)
- **Out of scope:** [OP Stack](docs/research/op-stack.md) · [Arbitrum Orbit](docs/research/arbitrum-orbit.md) · [Polygon Edge / CDK](docs/research/polygon-edge-cdk.md) · [GoQuorum](docs/research/goquorum.md)
- **Comparison index:** [README.md](docs/research/README.md) · **Validation plan:** [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md)

---

## Appendix A — Implementation path for the chosen stack

### A.1 Reference architecture (from `tempoxyz/tempo`)

```
                          ┌──────────────────────────────────────────┐
                          │            Sovereign L1 node               │
   p2p (Commonware) ───►  │  ┌───────────────┐     ┌────────────────┐ │
                          │  │  Commonware    │     │  application    │ │  Automaton:
   validator set  ───►    │  │  Simplex BFT   │◄───►│  actor (glue)   │ │  propose / verify
   (Supervisor)           │  │  engine        │     └───────┬─────────┘ │  payloads
                          │  └──────┬─────────┘             │           │
                          │         │ marshal: finalized,   │ Engine-API│
                          │         │ totally-ordered blocks│ data types│
                          │         ▼                       ▼ (in-proc) │
                          │  ┌───────────────┐     ┌────────────────┐  │
                          │  │  executor      │────►│  Reth (library) │  │
                          │  │  actor (glue)  │ FCU │  revm + custom  │──┼──►  eth_* JSON-RPC
                          │  └───────────────┘     │  PrecompilesMap │  │     (tooling)
                          │                        │  mempool/state  │  │
                          │                        └────────────────┘  │
                          └──────────────────────────────────────────┘
```

- **Commonware owns:** leader election, ordering, BFT voting/finality, threshold certs, block dissemination/backfill (`marshal`), runtime.
- **Reth owns:** mempool, payload building, revm execution, state + state root, DB, `eth_*` RPC.
- **We own (the glue):** the `application` + `executor` actors — our fork of Tempo's `crates/consensus`.

### A.2 Fork → keep fee module → extend

1. **Fork** `tempoxyz/tempo` (Apache-2.0/MIT). Pin Reth and `commonware-*` to Tempo's revisions; plan a controlled upgrade cadence.
2. **Keep & adapt the fee module (R5).** Retain the `0x76` `fee_token` envelope + FeeManager precompile + fixed-rate Fee AMM. Decide eligibility:
   - **Tempo model** — bespoke TIP-20 + structural USD eligibility + fixed-rate (MEV-free, no-oracle) AMM. Lowest fork friction; ties you to TIP-20/pathUSD.
   - **Celo model** — governance-curated `FeeCurrencyDirectory` + per-token oracles + decimal adapters over arbitrary existing ERC-20s. More flexible; oracle + governance surface.
   Decide **native gas token: none or optional** — none for the seamless stablecoin-only UX, or optional if needed for staking/MEV/incentives. Strip only the Stripe/payments-product code, not the fee mechanism.
3. **Extend** with: a **staking/validator-selection module** feeding Commonware's `Supervisor` (this is where PoS/DPoS/PoA is chosen — a thin layer, not a consensus change); **custom precompiles** for our modules via `ConfigureEvm`/`PrecompilesMap` (interface, gas/determinism rules, and the precompile-vs-system-contract call are in [precompiles-turing-incomplete-modules.md](docs/architecture/precompiles-turing-incomplete-modules.md)); **genesis, chain config, governance/upgrades, bridging/DA**.

### A.3 Build-vs-free checklist & effort

| Component | Source | Effort |
|---|---|---|
| EVM execution, mempool, payload builder, state/state-root, `eth_*` RPC | **Free** (Reth) | — |
| BFT consensus, finality, threshold certs, dissemination/backfill, runtime | **Free** (Commonware) | — |
| Consensus↔execution glue (`application` + `executor` actors) | **Fork** (Tempo `crates/consensus`) | M (adapt) vs XL (greenfield) |
| Stablecoin fee module (R5): `0x76` tx + FeeManager + Fee AMM | **Fork & adapt** (Tempo; or re-implement Celo `FeeCurrencyDirectory`) | M (Tempo model) – L (Celo-style arbitrary ERC-20 + oracles) |
| Block type/codec, genesis | Fork + adapt | S–M |
| Staking / validator-selection (PoS/DPoS/PoA) feeding `Supervisor` | **Build** | M–L |
| DKG / resharing validator reconfiguration | Pattern from Tempo (`OnchainDkgOutcome` + validator-config precompile) | M |
| Custom domain precompiles | **Build** (reuse pattern) | M + audit |
| Governance / upgrades, bridging, DA | **Build** | L |

### A.4 Sequencing note

Run Tempo **as-is** first, then strip — do not refactor before it builds and a multi-node devnet produces blocks. Verify version-sensitive API names against current source on day one (the `Supervisor`/`ThresholdSupervisor` traits, the `threshold_simplex` module path, and `Automaton::genesis` were flagged as drifting in the glue research).
