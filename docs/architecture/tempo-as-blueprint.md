# Tempo as a Blueprint for a Commonware + Reth Sovereign EVM L1

**Audience:** A team evaluating whether to build its own sovereign EVM L1 on the **Commonware (Simplex BFT) + Reth/revm** stack, using Stripe + Paradigm's **Tempo** as a reference.

**Bottom line up front:** Tempo's entire node — including the consensus↔execution glue you care about — is **open source under Apache-2.0 / MIT** at [github.com/tempoxyz/tempo](https://github.com/tempoxyz/tempo). This is far more than a marketing site. The repo is a complete, production-grade Rust workspace that wires `commonware-consensus` (Simplex) to a custom Reth-SDK node via the Engine-API *data types* (forkchoice + payload), with native-Rust precompiles, a custom EIP-2718 transaction type (`0x76`), and a stablecoin fee model. A third party can read, copy, and adapt this. The main caveats are: it is unaudited, the validator set is centralized (4 team-run validators), and several payments-specific design decisions are coupled tightly to Tempo's product and would need to be stripped out.

---

## Executive summary

- **The glue is public.** The `crates/consensus` crate depends on both `commonware-consensus` and `reth-node-builder`/`reth-evm`/`reth-revm`, and its top comment states the consensus engine is *"modelled after commonware's [alto] toy blockchain."* This is the single most valuable artifact for your project: a real, working example of the exact integration you are contemplating, not just an architecture talk.
- **Integration pattern = Engine API data types, in-process (NOT JSON-RPC).** Consensus drives execution by constructing `ForkchoiceState` / `TempoPayloadAttributes` / `PayloadId` and calling Reth's in-process `ConsensusEngineHandle` / `beacon_engine_handle` directly. The external JSON-RPC Engine API is explicitly disabled (`NoopEngineApiBuilder`). So Tempo reuses Reth's beacon-consensus *machinery and types* but skips the network hop a normal CL↔EL split would impose.
- **Reth is used as a library** via the node builder. Tempo defines a `TempoNode` implementing Reth's `Node`/`NodeTypes` traits, swapping in custom `ExecutorBuilder` (EVM config), `ConsensusBuilder`, `PayloadBuilder`, `PoolBuilder`, and `PayloadValidatorBuilder`. This is the canonical "build your own chain with the Reth SDK" pattern, made concrete.
- **Payments features are mostly native-Rust precompiles**, not Solidity system contracts and not consensus-layer logic. TIP-20 Factory, Fee Manager, Stablecoin DEX, Account Keychain, Signature Verifier (secp256k1/P256/WebAuthn) etc. live in `crates/precompiles` at fixed addresses and are injected into revm via a custom `PrecompilesMap` in `TempoEvmConfig`.
- **Credibility is high, battle-testing is low.** Built by Stripe + Paradigm with Commonware (Patrick O'Grady's team) providing the consensus library. ~$500M raised at ~$5B valuation (Oct 2025), public testnet then mainnet **March 18, 2026**. But: only 4 team-run validators, unaudited at the time the README was written, and only ~3 months of mainnet history as of this writing.

---

## What is public vs proprietary

### Public and reusable (Apache-2.0 / MIT)

Verified by cloning `github.com/tempoxyz/tempo` (commit `9f2041c`, 2026-06-24). The workspace is ~79% Rust, ~15% Solidity. Relevant crates under `crates/`:

| Crate | What it is | Why it matters to you |
| --- | --- | --- |
| `consensus` | The **glue**. Commonware Simplex engine + actors (application, executor, marshal, epoch, dkg, feed, follow). | This is the Commonware↔Reth bridge in full. |
| `node` | `TempoNode` Reth-SDK node definition + builders + `engine.rs`. | Shows how to instantiate Reth as a library and replace its consensus. |
| `evm` | `TempoEvmConfig`, custom revm `Evm`, block assembly/validation, `TempoConsensus`. | Shows custom `ConfigureEvm`, precompile injection, block rules. |
| `revm` | revm extensions/context. | Custom execution context plumbing. |
| `payload` (`tempo-payload*`) | `TempoPayloadBuilder`, `TempoPayloadAttributes`, `TempoBuiltPayload`. | The payload-build path consensus calls into. |
| `precompiles` + `precompiles-macros` | Native-Rust precompiles (TIP-20, Fee Manager, DEX, Keychain, Signature Verifier, Policy Registry…). | Pattern for protocol features as precompiles. |
| `primitives`, `alloy`, `chainspec` | `TempoTxType` (`0x76`), `TempoTxEnvelope`, signatures, chain spec. | Custom EIP-2718 tx type + chain config. |
| `transaction-pool` | `TempoPoolBuilder`, AA tx validation. | Mempool customization for a custom tx type. |
| `contracts` | Precompile **addresses** and Solidity interfaces/libraries. | The on-chain ABI surface. |

Other open repos in the `tempoxyz` org (all Apache-2.0 unless noted): `zones` (private subchains), `tempo-go`, `pytempo`, `accounts`, `wallet-rs`, `mpp-specs`, `tempo-std` (Solidity), `docs`. The 40+ `tips/tip-*.md` protocol specs are in-repo.

**Commonware itself is open source** ([commonware.xyz](https://commonware.xyz/), [github.com/commonwarexyz](https://github.com/commonwarexyz)) under Apache-2.0/MIT: `consensus::simplex`, `consensus::threshold_simplex`, `cryptography::bls12381` (with DKG/resharing), `p2p`, `storage`, `runtime`, etc. **Alto** ([github.com/commonwarexyz/alto](https://github.com/commonwarexyz/alto)) is the open "minimal and wicked fast" reference chain that Tempo's engine is modelled on — but Alto is a *non-EVM toy chain* (opaque blobs), so it teaches you Commonware wiring, not EVM execution.

### Proprietary / not in the public repo (inference where noted)

- **Audit reports and bug-bounty.** The README states Tempo "is still undergoing audit and does not have an active bug bounty." So the open code is *not* a vetted reference yet.
- **Validator operations / key management / genesis ceremony details.** The code is open, but the production validator set is team-run (4 nodes) and operational runbooks are not the deliverable here.
- **Stripe-side integration, KYC/compliance backends, the hosted fee-payer/indexer services** are products, not in the chain repo (some clients exist, e.g. `faucet` crate, `tempo-sidecar`).
- **Future privacy token standard / on-chain FX** are listed as "coming soon" — design not fully public. *(Inference: partially prototyped in the `zones` repo.)*

**Net:** Unlike most "open" L1s that publish only a forked client, Tempo has published the actual sovereign-L1-on-Reth glue. For your purpose this is close to a worked reference implementation. The gap is assurance (audits) and the need to delete payments-specific coupling.

---

## Reference architecture (as published)

### Component split

- **Execution layer:** Reth used as a library (Reth SDK / node builder), revm under the hood, targeting the **Osaka** EVM hardfork. `TempoNode` implements Reth's `Node`/`NodeTypes`.
- **Consensus layer:** Commonware **Simplex** BFT (leader-based, view/round model, notarization + nullification, deterministic finality, no reorgs). Tempo also pulls in Commonware **BLS12-381 + DKG** (the `dkg` actor and `epoch` manager in the consensus crate) — i.e. it runs an on-chain DKG ceremony per epoch to manage the validator threshold scheme.
- **Bridge:** an in-process actor system in `crates/consensus`. The consensus `Engine` (`consensus/engine.rs`) holds an `Arc<TempoFullNode>` (the Reth node) and spawns actors that translate Simplex decisions into Reth Engine-API calls.

### Transaction → consensus → execution → state-commitment flow (from the code)

1. **Mempool.** Transactions (including the custom `0x76` Tempo tx type) enter Reth's pool via `TempoPoolBuilder` (`crates/transaction-pool`), with extra stateless/stateful validation for account-abstraction fields (`valid_after`, authorizations).
2. **Leader proposes (build).** When a node is Simplex leader for a view, the `application` actor's `handle_propose` → `propose` (`crates/consensus/src/consensus/application/actor.rs`) runs:
   - resolves the parent block (via the `marshal` syncer / Reth provider),
   - builds `TempoPayloadAttributes` (proposer pubkey, ms-precision timestamp, `consensus_context` = epoch/view/parent_view/proposer, optional DKG outcome in `extra_data`, a payload-build time budget),
   - calls `executor.canonicalize_and_build(parent_height, parent_digest, attrs)`, which sets forkchoice to the parent and **kicks off a Reth payload build job**, then awaits the built payload.
   - The resulting Reth execution block is wrapped into a Commonware `Block` (with an optional Block Access List) and broadcast / persisted via the marshal.
3. **Replicas verify.** Non-leaders run `handle_verify` → `verify_block`, which re-executes/validates the proposed block against the execution layer using `beacon_engine_handle` (new-payload semantics) before voting in Simplex.
4. **Simplex agreement.** Standard Simplex notarize→finalize over the proposed digest. On timeout, validators nullify (vote a null block referencing last notarized state) rather than stalling.
5. **Finalize → canonicalize.** The `executor` actor (`crates/consensus/src/executor/actor.rs`) — described in its own header as *"Drives the actual execution forwarding blocks and setting forkchoice state"* — receives finalized blocks from the marshal and issues **forkchoice updates** (`ForkchoiceState { head, safe, finalized }`) to Reth via `submit_forkchoice_update`. It carefully keeps `finalized <= head`, tracks consensus vs execution finalized heights, backfills gaps on startup, and sends periodic FCU heartbeats.
6. **State commitment.** Reth/revm commits the block; state root and canonical chain advance inside Reth. Older blocks are served from Reth's DB; only recently finalized blocks are retained in the consensus marshal's prunable archive (`finalized_blocks_retention`).

### The key integration insight: Engine-API *types*, in-process, no JSON-RPC

Tempo does **not** run a standard split CL/EL over the JSON-RPC Engine API. Instead:

- It reuses the Engine API **data model** — `alloy_rpc_types_engine::{ForkchoiceState, PayloadId}`, `TempoPayloadAttributes`, `ConsensusEngineHandle`, `beacon_engine_handle`, `PayloadKind` — so it inherits Reth's beacon-consensus block-production and forkchoice plumbing.
- But the calls are **in-process Rust** against an `Arc<TempoFullNode>`, and the external Engine API server is disabled via Reth's `NoopEngineApiBuilder` (in `crates/node/src/node.rs`, `TempoAddOns`). This removes the network round-trip and gives consensus direct, typed control of execution.

This is the most transferable architectural decision in the whole project: **embed Reth and speak Engine-API semantics in-process rather than bolting a consensus client onto a stock Reth over HTTP.**

### Performance claims (as published, not independently benchmarked here)

- Simplex BFT, **deterministic sub-second finality** ("~0.6s" finality; docs/marketing also cite block times of ~0.5s). No chain reorgs.
- Reth execution cited at up to ~16k RPS. (These are vendor/secondary-source figures — see "Limits" below.)

---

## Lessons / blueprint for the glue

Concrete, code-grounded takeaways for building your own Commonware + Reth L1:

1. **Use Reth as a library through the node builder, not a fork.** Implement `Node`/`NodeTypes` and supply custom builders: `ExecutorBuilder` (your `ConfigureEvm`), `ConsensusBuilder` (replaces PoW/PoS validation rules), `PayloadBuilder`, `PoolBuilder`, `PayloadValidatorBuilder`. Tempo's `crates/node/src/node.rs` is a complete worked example. This keeps you on upstream Reth and lets you `git`-track it.

2. **Bridge with Engine-API data types in-process.** Hold an `Arc<FullNode>` in your consensus engine. Drive block building with `PayloadAttributes` + `PayloadId`, and advance the canonical chain with `ForkchoiceState` forkchoice-updates. Disable the external Engine API (`NoopEngineApiBuilder`) if your consensus lives in the same process. This is the cleanest way to get sub-second finality without a JSON-RPC hop.

3. **Model your consensus engine on Alto, then layer Reth in.** Tempo literally did this ("modelled after commonware's alto"). Start from Alto to learn Commonware's actor/runtime model (`application`, `marshal`, `p2p`, `storage`, `runtime` with its `Pacer` for deterministic tests), then replace Alto's opaque-blob execution with Reth payload build/verify calls. Note Tempo's pattern of `Pacer::pace` calls so the Commonware *deterministic test runtime* can drive the real (tokio) execution layer in e2e tests — important for testability.

4. **Separate "propose/build" from "finalize/canonicalize."** Tempo splits these into two actors: the `application` actor builds & verifies payloads during the view; the `executor` actor applies finalized forkchoice updates. Mirror this separation — it keeps liveness timeouts (time-to-propose, proposal-return budget) decoupled from execution commitment.

5. **Implement protocol features as native-Rust precompiles, injected via a custom `PrecompilesMap`.** Tempo's `TempoEvmConfig` exposes `type Precompiles = PrecompilesMap` and installs precompiles at fixed addresses (e.g. Fee Manager `0xfeec…`, TIP-20 Factory `0x20FC…`, Stablecoin DEX `0xdec0…`, Signature Verifier `0x5165…`, Account Keychain `0xAAAA…`). Precompiles are cheaper and more controllable than Solidity system contracts and avoid consensus-layer special-casing. Use this when you have enshrined protocol logic.

6. **Add a custom transaction type via a typed EIP-2718 envelope.** `TempoTxType` / `TempoTxEnvelope` define type `0x76` (`TEMPO_TX_TYPE_ID = 0x76`) carrying account-abstraction fields (fee token, WebAuthn/P256 auth, batched calls, scheduling). The pattern: define the envelope in a `primitives`/`alloy` crate, teach the pool and EVM to decode/validate it. Note this is a *non-standard* type byte — wallets/tooling must support it.

7. **Decouple fees from a native token if that fits your product.** Tempo has **no native gas token**: `BALANCE`/`SELFBALANCE`/`CALLVALUE` return 0; gas is paid in stablecoins; a Fee Manager precompile batches fee→validator-preferred-token swaps at block end (avoiding sandwich MEV). This is implemented at the EVM/precompile layer, not consensus. Instructive even if you keep a native token: it shows how far you can bend revm's fee assumptions through the EVM config.

8. **Tune for sub-second finality explicitly.** The engine `Builder` exposes a rich set of liveness knobs: `time_to_propose`, `time_to_collect_notarizations`, `time_to_retry_nullify_broadcast`, `proposal_return_budget`, `fcu_heartbeat_interval`, `views_until_leader_skip`, plus Commonware's buffered BLS signature verification. Plan to expose and tune these; finality numbers depend heavily on them and on validator-set size.

9. **PoA / small validator set with epoch DKG.** Tempo runs a small permissioned validator set with a per-epoch DKG (`dkg` + `epoch` actors) producing a threshold scheme. If you start permissioned, this is a ready pattern; if you intend to decentralize, budget for the validator-onboarding and DKG-resharing work, which is non-trivial.

10. **Block Access Lists + parallel execution.** Tempo's blocks carry an optional Block Access List (BAL) and it advertises parallel transaction execution / dedicated "payment lanes." If throughput matters, study the `evm/block.rs` and BAL handling — but note several sub-features (subblocks, TIP-1016) are gated/disabled by feature flags and hardfork (T4+), so the public code has both live and dormant paths.

---

## Limits of Tempo as guidance

- **Unaudited reference.** Per the repo's own SECURITY note, Tempo was undergoing audit with no active bounty. Do not treat the glue code as security-vetted; you inherit its bugs if you copy it.
- **Centralized validator set.** 4 team-run (rotating) validators. The published finality/throughput characteristics reflect a small, well-connected, trusted set — *not* a large adversarial network. Your decentralization roadmap is your own problem; Tempo doesn't yet demonstrate a solution at scale.
- **Performance figures are vendor/secondary.** "~0.6s finality," "~0.5s block time," "16k RPS" come from Tempo/Commonware materials and third-party write-ups, not from an independent benchmark in this research. Treat as targets/best-case, contingent on validator count, hardware, network, and the timeout knobs above.
- **Heavy payments coupling.** Large parts of the value (precompiles, fee AMM, TIP-20/403, no-native-token EVM tweaks, account keychain) are product decisions you'd rip out or replace. The genuinely reusable glue is `crates/consensus` + the `node`/`evm`/`payload` builder wiring; the rest is example-by-analogy.
- **Moving target / dormant code.** The repo carries feature flags and hardfork-gated paths (subblocks, BAL, TIP-1016) that are off in production. Reading it, you must distinguish live from experimental code. Pin to a commit.
- **Reth SDK is still maturing.** The "Reth as a library" surface (node builder traits) evolves; Tempo tracks a specific Reth version. Expect to chase upstream changes. The Reth SDK docs and Alto/Tempo are your best references because formal tutorials for this exact pattern are thin.
- **Commonware "anti-framework" learning curve.** Commonware is deliberately low-level primitives, not a batteries-included framework. The actor/runtime/codec model (and its deterministic test runtime) is powerful but has a real ramp-up cost. Alto + Tempo's consensus crate are the practical teachers.

---

## Team, credibility & timeline

- **Builders:** Stripe + Paradigm (Reth is Paradigm's client). Consensus library by **Commonware**, founded by **Patrick O'Grady** (ex-Ava Labs/Avalanche engineering lead). Commonware raised ~$9M (Haun Ventures, Dragonfly).
- **Funding:** Tempo raised ~**$500M at ~$5B valuation** (~Oct 2025); round associated with Thrive Capital and Greenoaks, with Sequoia, Ribbit, SV Angel participating (per secondary sources).
- **Timeline:** Public testnet ("Moderato", chain ID 42431) in late 2025; **mainnet launched March 18, 2026**. Advisory unit for stablecoin adoption launched ~April 2026.
- **Partners (design/ecosystem, per press):** Visa, Mastercard, UBS, Deutsche Bank, Standard Chartered, Revolut, Nubank, Klarna, Kalshi, Shopify, DoorDash, Ramp, OpenAI, Anthropic; 40+ infra partners (MetaMask, Phantom, LayerZero, Across, etc.). Treat partner lists as marketing-sourced.

---

## Sources

Primary (code/docs):
- [github.com/tempoxyz/tempo](https://github.com/tempoxyz/tempo) — the node, Apache-2.0/MIT (inspected at commit `9f2041c`, 2026-06-24): `crates/consensus`, `crates/node`, `crates/evm`, `crates/payload`, `crates/precompiles`, `crates/primitives`, `crates/contracts`.
- [tempo/README.md](https://github.com/tempoxyz/tempo/blob/main/README.md)
- [github.com/tempoxyz](https://github.com/tempoxyz) — org repos (zones, accounts, wallet-rs, mpp-specs, etc.)
- [docs.tempo.xyz](https://docs.tempo.xyz/) — protocol docs
- [docs.tempo.xyz — Performance](https://docs.tempo.xyz/learn/tempo/performance)
- [docs.tempo.xyz — EVM compatibility](https://docs.tempo.xyz/quickstart/evm-compatibility)
- [docs.tempo.xyz — Fee AMM](https://docs.tempo.xyz/protocol/fees/fee-amm)
- [github.com/commonwarexyz/alto](https://github.com/commonwarexyz/alto) — reference chain Tempo's engine is modelled on
- [commonware.xyz](https://commonware.xyz/) and [Commonware: the Anti-Framework](https://commonware.xyz/blogs/commonware-the-anti-framework), [Threshold Simplex](https://commonware.xyz/blogs/threshold-simplex), [Buffered Signatures](https://commonware.xyz/blogs/buffered-signatures)
- [Patrick O'Grady](https://patrickogrady.xyz/)
- [Reth (paradigmxyz/reth)](https://github.com/paradigmxyz/reth)

Secondary (context/timeline/figures — treat as non-authoritative):
- [The Block — Tempo public testnet](https://www.theblock.co/post/381930/stripe-and-paradigms-payments-focused-blockchain-tempo-launches-public-testnet)
- [Yahoo/CoinDesk — Tempo opens with Mastercard, UBS](https://finance.yahoo.com/news/stripes-tempo-payments-blockchain-opens-220205103.html)
- [Fortune — Tempo advisory unit](https://fortune.com/2026/04/21/stripe-and-paradigm-tempo-advisory-stablecoin-adoption/)
- [Ledger Insights — Tempo + Machine Payments Protocol](https://www.ledgerinsights.com/stripe-paradigm-launch-tempo-blockchain-alongside-machine-payments-standard/)
- [Chainstack — Tempo stablecoin infrastructure](https://chainstack.com/tempo-blockchain-stablecoin-payments-infrastructure/)
- [The Block — Commonware $9M raise](https://www.theblock.co/post/330443/former-ava-labs-engineering-lead-raises-9-million-to-build-anti-framework-blockchain-developer-tools)
- [Blockworks — Commonware anti-framework](https://blockworks.co/news/commonware-optimized-blockchain-development)

---

**Last verified: 2026-06-25**
