# Audit Coverage Attestations: metrics

> **Status: draft.** Nothing described here exists yet. Development starts only once the initiative is funded. The definitions below are proposals for review. The measurement methods in [section 4](#4-open-points) are still open.

The public metrics page reports adoption from onchain data. Each figure on the page links to the query that produces it, so anyone can reproduce it.

## 1. Terms

**Counted attestation.** An attestation the default list counts, at the time of measurement (see [SPEC §5](SPEC.md#5-status-vocabulary)).

**Recognised issuer.** An issuer key on the default list at the time of measurement. Keys in one succession chain are one issuer, shown with the chain's domain.

**Covered bytecode.** A `compiledRuntimeCodeHash` with at least 1 counted attestation, so its status is `present` or `out_of_scope`. The page shows both statuses separately.

**Report date.** The attestation's `reportDate` (see [SPEC §2](SPEC.md#2-attestation)), as stated by the issuer. The page links each report date to the report at its `reportURI`.

**Publication lag.** The attestation's EAS timestamp minus its `reportDate`. An attestation whose `reportDate` is later than its EAS timestamp is excluded from lag figures, since only public reports are attested.

**Executing code.** For an address, its own code, or for a proxy its current implementation at the time of measurement, found by Routescan's proxy detection.

**Main contract.** For one protocol on one chain, the contract that holds the largest share of the protocol's TVL on that chain.

**Value held onchain.** For an address on Ethereum mainnet, its native balance plus its balances of priced ERC-20 tokens, in USD at the time of measurement. A token contract's balance of its own token is excluded.

## 2. Figures on the page

| Figure | Definition |
| :--- | :--- |
| Attestations | Published, counted and revoked, in total and per issuer. Each is listed with its issuer address and report hash. |
| Issuers | All issuers and recognised issuers, with domains |
| Covered bytecodes | Distinct covered bytecodes that match at least 1 exact-match verified deployment, per chain |
| Publication lag | Distribution of publication lag, and the attestations published within 30 days of their report date |
| Value held by covered contracts | Sum of value held onchain by every Ethereum mainnet address whose executing code is a covered bytecode |
| Covered TVL | Sum of TVL over (protocol, chain) pairs whose main contract's executing code is a covered bytecode |

A deployment matches a bytecode as defined in [SPEC §3](SPEC.md#3-bytecode-identity). Each address counts once, however many attestations cover its code.

## 3. Milestone targets

| Target | Figure |
| :--- | :--- |
| At least 8 recognised firms have published attestations | Issuers |
| At least 100 distinct bytecodes on Ethereum mainnet or an Ethereum L2 carry a recognised attestation | Covered bytecodes |
| At least 25 of those attestations published within 30 days of their report date | Publication lag |
| Protocols whose main contract is covered secure at least $500 million in combined TVL | Covered TVL |

## 4. Open points

- **TVL source.** Which data source gives protocol TVL, and how TVL is attributed to individual contracts to find the main contract.
- **Price source.** Which ERC-20 tokens are priced, and from where.
- **Chains.** Which chains count as Ethereum L2s, and whether value held is measured beyond Ethereum mainnet.
- **Report date.** Whether `reportDate` can be checked independently of the issuer's statement.
- **Cadence.** How often figures are recomputed, and whether past snapshots are kept.
