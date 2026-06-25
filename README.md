# nvnm_chain — Architecture & Decision Record

A sovereign **stablecoin EVM L1**. The implementation stack is settled:

> **Reth (execution, as a library) + Commonware Simplex BFT (consensus), bootstrapped by forking Stripe's open-source [Tempo](https://github.com/tempoxyz/tempo) node** — retaining its enshrined any-stablecoin fee model. See [ADR-0001](ADR-0001-evm-chain-implementation.md) for the decision and rationale.

## Decision & implementation docs (start here)

| Doc | Purpose |
|---|---|
| **[ADR-0001 — EVM Chain Implementation](ADR-0001-evm-chain-implementation.md)** | The decision: requirements, scoring of all candidates, chosen stack, consequences. |
| **[PROPOSAL-0001 — Fork Tempo](PROPOSAL-0001-fork-tempo.md)** | How we build it: keep/adapt/strip/build by crate, phases, effort. |
| **[SPIKE-0001 — Validation](SPIKE-0001-reth-commonware-tempo.md)** | 3-week timeboxed spike with a go/no-go gate (run before committing). |
| **[ISSUE-TREE — Fork Tempo](ISSUE-TREE-fork-tempo.md)** | Epic → issue breakdown with IDs, dependencies, estimates, acceptance criteria. |

## Architecture references (`docs/architecture/`)

The design deep-dives behind the chosen stack:

- [Commonware ↔ Reth glue mechanics](docs/architecture/commonware-reth-glue.md) — how consensus drives execution.
- [Tempo as a blueprint](docs/architecture/tempo-as-blueprint.md) — the open-source reference node, what to fork.
- [Stablecoin gas mechanisms](docs/architecture/stablecoin-gas-mechanisms.md) — paying gas in any stablecoin (enshrined vs. paymaster).
- [Turing-incomplete modules via precompiles](docs/architecture/precompiles-turing-incomplete-modules.md) — adding native modules as EVM contracts.

## Background research (`docs/research/`)

The full comparison that produced the decision — 12 EVM-stack evaluations plus the index:

- **[Comparison index & matrix](docs/research/README.md)** — the scored comparison of all candidates and why each was chosen or ruled out.

## Repository layout

```
.
├── README.md                              # this file
├── ADR-0001-evm-chain-implementation.md   # the decision
├── PROPOSAL-0001-fork-tempo.md            # the build plan
├── SPIKE-0001-reth-commonware-tempo.md    # the validation spike
├── ISSUE-TREE-fork-tempo.md               # execution breakdown
└── docs/
    ├── architecture/                      # design deep-dives for the chosen stack
    └── research/                          # candidate evaluations + comparison index
```

## Status

ADR-0001 is **Proposed**. Next step: run **SPIKE-0001**, resolve the open decision points in [ISSUE-TREE Epic D](ISSUE-TREE-fork-tempo.md#epic-d--decision-points-blocking-gates-resolve-before-the-dependent-phases), then move the ADR to **Accepted** and execute PROPOSAL-0001.
