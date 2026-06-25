# Issue Tree — Fork Tempo into a Sovereign Stablecoin L1

Operationalises [PROPOSAL-0001](PROPOSAL-0001-fork-tempo.md) (phases §7, decision points §11) into an epic → issue hierarchy. Implements [ADR-0001](ADR-0001-evm-chain-implementation.md); gated by [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md).

**Last updated:** 2026-06-25

## How to use this

- **IDs are stable.** Epics map to proposal phases (`P0`–`P7`), plus a Decisions epic (`D`) and a Cross-cutting epic (`X`). Child issues are `P2-03`, `D-01`, etc. Paste into GitHub/Linear/Jira and keep the IDs as references.
- **Estimates** are engineer-weeks (EW), rolled up to match PROPOSAL-0001 §7 ranges.
- **`blocks` / `needs`** capture dependencies; the **critical path** is summarised at the end.
- Labels suggested: `area:consensus` `area:evm` `area:precompiles` `area:fees` `area:tooling` `area:ops` `area:security` `type:decision` `type:spike` `good-first-fork-task`.

---

## EPIC D — Decision points *(blocking gates; resolve before the dependent phases)*

> These are the §11 stakeholder decisions. Each is a short `type:decision` issue that ends in a written ruling recorded back in ADR-0001 / PROPOSAL-0001.

- [ ] **D-01 — Consensus family.** PoA only, or PoS/DPoS? · *needs:* P0 · *blocks:* P2-03, D-03 · *est: decision*
  - **AC:** documented choice + rationale; if PoS/DPoS, the staking→validator-set mapping is sketched.
- [ ] **D-02 — Fee eligibility model.** Tempo TIP-20 model vs. Celo `FeeCurrencyDirectory` (arbitrary ERC-20 + oracles)? · *needs:* P0 · *blocks:* P3-01 · *est: decision*
  - **AC:** choice + rationale; oracle/governance implications noted.
- [ ] **D-03 — Native gas token.** None (stablecoin-only) vs. optional (staking/incentives)? · *needs:* D-01 · *blocks:* P3-02 · *est: decision*
- [ ] **D-04 — Transaction types.** Keep `0x76` only, or also accept standard type-2? *(recommend both)* · *needs:* P0 · *blocks:* P5-01 · *est: decision*
- [ ] **D-05 — Domain precompiles.** Which native modules do we need (e.g. compliance/KYC, oracle, validator-config read)? · *needs:* P0 · *blocks:* P4-02 · *est: decision*
- [ ] **D-06 — Strip scope.** Keep or drop `zones` / `faucet` / `sidecar` and dormant throughput features (subblocks, BAL/TIP-1016)? · *needs:* P0 · *blocks:* P1-04 · *est: decision*

---

## EPIC P0 — Validate (SPIKE-0001) *(timebox 3 weeks)*

> Full detail in [SPIKE-0001](SPIKE-0001-reth-commonware-tempo.md). Tracked here as the gating epic.

- [ ] **P0-01 — Fork, build, single node** *(WS-A)* · *est: 0.5 EW* · **AC:** node produces blocks from genesis.
- [ ] **P0-02 — ≥4-validator devnet** *(WS-A)* · *needs:* P0-01 · *est: 0.5 EW* · **AC:** 4 validators reach consensus; survives a node restart.
- [ ] **P0-03 — R5 proof: stablecoin gas end-to-end** *(WS-B)* · *needs:* P0-01 · *est: 0.75 EW* · **AC:** account with 0 native balance pays gas in a stablecoin via `0x76`; balances + FeeManager state verified.
- [ ] **P0-04 — Tooling matrix** *(WS-C)* · *needs:* P0-02 · *est: 0.5 EW* · **AC:** MetaMask/Foundry/ethers/viem + an indexer work against the devnet.
- [ ] **P0-05 — Custom precompile probe** *(WS-D)* · *needs:* P0-01 · *est: 0.5 EW* · **AC:** trivial precompile callable from Solidity, deterministic.
- [ ] **P0-06 — Validator-selection swap probe** *(WS-D)* · *needs:* P0-02 · *est: 0.5 EW* · **AC:** validator-set membership change takes effect via our own input.
- [ ] **P0-07 — Spike report + GO/NO-GO** · *needs:* P0-01..06 · *est: 0.25 EW* · **AC:** report with E1–E8 evidence; decision recorded.

**Gate:** a GO on P0-07 unlocks P1+. A NO-GO triggers the ADR fallback (Cosmos EVM).

---

## EPIC P1 — Foundation *(4–6 EW)*

- [ ] **P1-01 — Fork & pin.** Fork `tempoxyz/tempo`; pin Tempo commit + Reth rev + `commonware-*` versions. · *needs:* P0-07(GO) · *est: 0.5 EW* · **AC:** reproducible build in our CI off pinned revs.
- [ ] **P1-02 — Rebrand.** Rename `Tempo*` → our chain across crates; package metadata, binaries. · *needs:* P1-01 · *est: 1 EW* · **AC:** builds + runs under new name; no `tempo` identifiers in user-facing surfaces.
- [ ] **P1-03 — Chain identity.** New chain ID, genesis, chainspec (`crates/chainspec`, `primitives`). · *needs:* P1-02 · *est: 1 EW* · **AC:** devnet boots under our chain ID/genesis.
- [ ] **P1-04 — Initial strip.** Remove product code per D-06 (zones/faucet/sidecar as decided); gate/document dormant feature-flag paths. · *needs:* D-06 · *est: 1–1.5 EW* · **AC:** build green; live-vs-dormant code documented.
- [ ] **P1-05 — CI/CD baseline.** Build, lint, unit tests; pinned toolchain; scheduled upstream-bump branch job (see X-01). · *needs:* P1-01 · *est: 1 EW* · **AC:** CI green on every PR; bump-branch job runs weekly.

---

## EPIC P2 — Consensus & validator model (R3) *(8–14 EW)*

- [ ] **P2-01 — Verify Commonware API surface.** Confirm `Supervisor`/`ThresholdSupervisor`, `threshold_simplex` path, `Automaton::genesis` against pinned rev. · *needs:* P1-01 · *est: 0.5 EW* · **AC:** API map documented; any drift from research noted.
- [ ] **P2-02 — Keep & adapt the glue.** Port `crates/consensus` (application/executor/marshal/epoch/dkg) under our brand; keep in-process Engine-API pattern + `NoopEngineApiBuilder`. · *needs:* P1-03 · *est: 2 EW* · **AC:** glue compiles + runs on our chainspec.
- [ ] **P2-03 — Validator-selection module.** Implement the source feeding the `Supervisor` per D-01 — PoA set, or PoS/DPoS staking module ranking validators by stake. · *needs:* D-01, P2-02 · *est: 3–6 EW* · **AC:** active set + weights derive from our policy; documented.
- [ ] **P2-04 — Epoch & DKG/resharing.** Validate per-epoch DKG and resharing on validator-set changes. · *needs:* P2-03 · *est: 2–3 EW* · **AC:** add/remove validator + resharing succeeds on devnet without liveness loss.
- [ ] **P2-05 — Liveness tuning.** Expose & tune engine knobs (`time_to_propose`, `time_to_collect_notarizations`, `proposal_return_budget`, `fcu_heartbeat_interval`, `views_until_leader_skip`). · *needs:* P2-02 · *est: 1 EW* · **AC:** finality measured + tuned for target validator count.
- [ ] **P2-06 — Consensus e2e tests.** Deterministic-runtime + `Pacer` tests; partition/restart/nullify scenarios. · *needs:* P2-04 · *est: 1.5 EW* · **AC:** reproducible passing e2e suite in CI.

---

## EPIC P3 — Token & fee model (R5) *(6–10 EW)*

- [ ] **P3-01 — Eligibility implementation.** Implement D-02 choice: keep Tempo TIP-20 path, or build Celo-style `FeeCurrencyDirectory` + oracles + decimal adapters. · *needs:* D-02, P1-03 · *est: 2–4 EW* · **AC:** an eligible stablecoin can pay gas; ineligible rejected.
- [ ] **P3-02 — Native-token behavior.** Implement D-03: keep no-native-token (BALANCE/CALLVALUE = 0) or wire an optional native token. · *needs:* D-03, P3-01 · *est: 1–2 EW* · **AC:** behavior matches decision; interacts correctly with P2-03 staking if applicable.
- [ ] **P3-03 — Fee Manager + Fee AMM.** Keep/adapt FeeManager precompile (`0xfeec…`) + fixed-rate AMM; validator fee-token settlement. · *needs:* P3-01 · *est: 1.5 EW* · **AC:** fees collected + swapped to validator-preferred token at block end; no sandwich MEV.
- [ ] **P3-04 — Fee-module fuzz + hardening.** Fuzz swap math, fee accounting, `0x76` fee fields; de-peg/liquidity edge cases. · *needs:* P3-03 · *est: 1.5 EW* · **AC:** fuzz suite green; de-peg kill-switch + liquidity floors in place.

---

## EPIC P4 — Precompiles (R2) *(6–12 EW)*

- [ ] **P4-01 — Keep core precompiles.** Signature Verifier (secp256k1/P256/WebAuthn), TIP-20 Factory + Stablecoin DEX (if TIP-20 kept), Account Keychain. · *needs:* P1-03 · *est: 1.5 EW* · **AC:** retained precompiles function under our chainspec.
- [ ] **P4-02 — Build domain precompiles.** Implement D-05 modules (e.g. compliance/KYC allowlist, oracle/price read, validator-config read). Read-only stateful where possible. · *needs:* D-05, P4-01 · *est: 3–6 EW* · **AC:** each callable from Solidity; deterministic; gas-bounded; unit + determinism tests.
- [ ] **P4-03 — System-contract split.** Move governance-tunable policy (compliance rules, issuance params, validator-onboarding policy) into upgradeable system contracts. · *needs:* P4-02 · *est: 1.5 EW* · **AC:** policy upgradeable without node upgrade; precompiles hold only fixed logic.
- [ ] **P4-04 — Address map + hardfork gating.** Re-document address map (no Ethereum-reserved/future-EIP collisions); gate every precompile change behind a Commonware-coordinated hardfork. · *needs:* P4-02 · *est: 1 EW* · **AC:** documented address registry; activation height mechanism tested.
- [ ] **P4-05 — Strip unneeded precompiles.** Remove Stripe-/product-specific precompiles we don't use. · *needs:* P4-01 · *est: 0.5 EW* · **AC:** build green; removed surface documented.

---

## EPIC P5 — Tooling, RPC & compatibility (R1) *(4–8 EW)*

- [ ] **P5-01 — Tx-type support.** Implement D-04: accept standard type-2 txs (default fee-token path) and/or `0x76`. · *needs:* D-04, P3-02 · *est: 2–3 EW* · **AC:** unmodified MetaMask sends a type-2 tx; `0x76` advanced flows work.
- [ ] **P5-02 — Signer/SDK support.** Adopt/adapt `tempo-go` / `wallet-rs` / `pytempo` for `0x76` signing. · *needs:* P5-01 · *est: 1–2 EW* · **AC:** an SDK signs + submits `0x76`; documented.
- [ ] **P5-03 — Compatibility matrix.** Full run: MetaMask, Hardhat, Foundry, ethers, viem, Blockscout-style indexer; execution-spec conformance tests. · *needs:* P5-01 · *est: 1.5 EW* · **AC:** matrix green; deviations documented for contract authors.
- [ ] **P5-04 — Hardfork target.** Confirm our EVM fork target (Osaka/Fusaka-era) + `evmVersion` guidance. · *needs:* P1-03 · *est: 0.5 EW* · **AC:** fork documented; contracts compile against it.

---

## EPIC P6 — Governance, ops & public testnet *(8–12 EW)*

- [ ] **P6-01 — Governance surface.** On-chain governance for system-contract params + hardfork-gated upgrades. · *needs:* P4-03 · *est: 2–3 EW* · **AC:** a parameter change + a gated upgrade execute via governance on devnet.
- [ ] **P6-02 — Ops tooling.** Validator runbooks, key management, monitoring/alerting, genesis-ceremony procedure. · *needs:* P2-04 · *est: 2–3 EW* · **AC:** a third party can run a validator from docs.
- [ ] **P6-03 — Public multi-org testnet.** Launch with external validators. · *needs:* P3-04, P4-04, P5-03, P6-02 · *est: 3 EW* · **AC:** testnet live with ≥1 external validator; uptime tracked.
- [ ] **P6-04 — Decentralization roadmap.** Plan + exercise validator-set growth and DKG resharing at larger N. · *needs:* P6-03 · *est: 1.5 EW* · **AC:** documented roadmap; resharing tested at target N.

---

## EPIC P7 — Audit & mainnet readiness *(10–16 EW + external audit)*

- [ ] **P7-01 — Audit prep.** Freeze scope; threat model; docs for auditors (glue, fee module, every precompile, consensus integration). · *needs:* P6-03 · *est: 2 EW* · **AC:** audit-ready package delivered.
- [ ] **P7-02 — External audit.** Independent audit (calendar + budget item). · *needs:* P7-01 · *est: external* · **AC:** report received.
- [ ] **P7-03 — Remediation.** Fix findings; re-review criticals. · *needs:* P7-02 · *est: 3–6 EW* · **AC:** all critical/high closed.
- [ ] **P7-04 — Validator-set expansion.** Grow beyond the small set toward target decentralization. · *needs:* P6-04 · *est: 2 EW* · **AC:** target validator count live on testnet.
- [ ] **P7-05 — Mainnet genesis.** Final chainspec, genesis ceremony, launch. · *needs:* P7-03, P7-04 · *est: 2 EW* · **AC:** mainnet producing blocks; monitoring green.

---

## EPIC X — Cross-cutting *(spans all phases)*

- [ ] **X-01 — Upstream tracking.** Thin-fork discipline; scheduled Reth/Commonware/Tempo bump branch; triage breakage. · *est: ongoing*
- [ ] **X-02 — Security baseline.** Dependency scanning, fuzz harnesses, determinism test gate in CI. · *est: ongoing*
- [ ] **X-03 — Documentation.** Developer docs, address registry, deviation notes, validator docs. · *est: ongoing*
- [ ] **X-04 — Performance benchmarking.** Benchmark our own validator set/hardware; don't assume vendor finality/RPS figures. · *est: ongoing*

---

## Dependencies & critical path

```
P0 ──► D-01..D-06 (decisions)
P0 ──► P1 ──► P2 ──► P3 ──► P4 ──► P5 ──► P6 ──► P7
            (D-01)  (D-02/03) (D-05)  (D-04)
```

**Critical path (longest chain):** P0 → P1 → **P2 (consensus/validators)** → P3 → P4 → P5 → P6 → P7-02 (external audit) → P7-05 (mainnet). P2 and P7 carry the most risk/variance; P3/P4 can partly overlap once P2 stabilises.

## Rollup

| Epic | Scope | Est (EW) |
|---|---|---|
| D | Decisions | — (gating) |
| P0 | Validate (SPIKE) | ~3.5 (3-wk timebox) |
| P1 | Foundation | 4–6 |
| P2 | Consensus & validators | 8–14 |
| P3 | Token & fee model | 6–10 |
| P4 | Precompiles | 6–12 |
| P5 | Tooling & RPC | 4–8 |
| P6 | Governance, ops, testnet | 8–12 |
| P7 | Audit & mainnet | 10–16 + external audit |
| X | Cross-cutting | ongoing |
| **Total build** | **to mainnet** | **~46–78 EW** + external audit calendar/budget |

Matches PROPOSAL-0001 §7. P0–P5 (a credible internal testnet) is the bulk of the value and lands first; P6–P7 is decentralization + assurance.
