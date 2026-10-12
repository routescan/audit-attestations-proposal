# Audit Coverage Attestations: API, MCP tools and data

> **Status: draft.** Nothing described here exists yet. Development starts only once the initiative is funded. Paths, parameters, field names and table layouts are TBD.

The coverage service is a standalone Apache-2.0 product that anyone can run, on any Ethereum mainnet RPC and with no Routescan credentials.

Statuses, counting and trust lists are defined in [SPEC.md](SPEC.md). This document covers how callers reach them and where the records live.

## 1. HTTP API

All endpoints are `GET` unless stated, and return JSON.

| Endpoint | Answers |
| :--- | :--- |
| `/v1/bytecodes/{hash}` | Status and attestations for a `compiledRuntimeCodeHash` |
| `/v1/bytecodes/creation/{hash}` | The same, by `compiledCreationCodeHash`, for checks before deployment |
| `/v1/chains/{chainId}/addresses/{address}` | Status for the bytecode stored at that address (see [SPEC §4](SPEC.md#4-resolving-executing-code)) |
| `POST /v1/addresses` | The same, for up to 100 `{chainId, address}` pairs |
| `/v1/attestations/{uid}` | One attestation, with its counted state |
| `/v1/issuers/{address}` | An issuer key: domain, binding state, recognition, succession chain and compromise |

Each instance applies one trust list, set in its configuration (see [SPEC §6](SPEC.md#trust-lists)). Routescan's instance applies the default list.

### Status response

| Field | Meaning |
| :--- | :--- |
| `status` | `present`, `out_of_scope`, `none` or `unverified` |
| `label` | The display label from [SPEC §5](SPEC.md#5-status-vocabulary) |
| `caveat` | The fixed caveat: coverage is not a statement about safety, configuration or ownership |
| `compiledRuntimeCodeHash` | The bytecode the status refers to |
| `deployment` | For an address query: chain, address and the matching `verified_contracts` row |
| `linkedLibraries` | Fully qualified name and address of each linked library |
| `attestations` | Every matching attestation, counted or not |
| `trustList` | The address of the list the instance applies |
| `indexedTo` | The last Ethereum mainnet block the indexer has processed |

Each entry in `attestations` carries the attestation's fields from [SPEC §2](SPEC.md#2-attestation), plus:

| Field | Meaning |
| :--- | :--- |
| `uid` | The EAS UID |
| `issuer` | The attester key |
| `domain` | The issuer's domain (see [SPEC §6](SPEC.md#domain-binding)) |
| `domainConfirmed` | Whether the domain is confirmed |
| `domainCheckedAt` | When the indexer last saw the TXT record |
| `recognised` | Whether the trust list recognises the issuer |
| `counted` | Whether the trust list counts this attestation |
| `attestedAt` | The EAS timestamp |
| `revokedAt` | The EAS revocation time, or null |

An address with no verified source returns `unverified` and an empty `attestations` list, even if attestations exist for some bytecode hash.

## 2. MCP tools

The tools are served on mcp.routescan.io and published as a standalone MCP server that wraps the same API.

| Tool | Input | Output |
| :--- | :--- | :--- |
| `get_bytecode_coverage` | `compiledRuntimeCodeHash` | The status response |
| `get_address_coverage` | `chainId`, `address` | The status response for the bytecode stored at that address |
| `get_issuer` | `address` | The issuer response |
| `check_transaction_coverage` | `chainId`, `from`, `to`, `data`, `value` | See below |

`check_transaction_coverage` simulates the transaction and reports in three groups:

- **Target:** the status of the code at `to`.
- **Delegated code:** the status of every code the target runs by `DELEGATECALL`, including a proxy's implementation and linked libraries.
- **Reported separately:** other contracts the transaction calls, such as tokens and routers, and the sender's own account if it has delegated code. Each has its own status, and none of them changes the target's.

The tool returns statuses only. Whether to sign is the caller's decision.

The standalone server needs an RPC that supports transaction tracing on the chain being simulated. The simulation method is TBD.

## 3. Records in the Verifier Alliance database

The indexer writes coverage records into the Verifier Alliance database, next to the verification records they join to. The tables below are a proposal for the [database-specs](https://github.com/verifier-alliance/database-specs) repository and need the Alliance's agreement.

### What goes into the Alliance database

Only facts read from EAS on Ethereum mainnet. Anyone can rebuild them from chain data, so a row written by Routescan can be checked by any member.

`coverage_attestations`, one row per `CoverageAttestation`:

| Column | Type | Notes |
| :--- | :--- | :--- |
| `uid` | `bytea` | Primary key, the EAS UID |
| `attester` | `bytea` | The issuer key |
| `compiled_runtime_code_hash_keccak` | `bytea` | Indexed |
| `compiled_creation_code_hash_keccak` | `bytea` | Indexed |
| `fully_qualified_name` | `varchar` | |
| `report_hash` | `bytea` | Indexed |
| `report_uri` | `varchar` | Null when empty |
| `source_commit` | `bytea` | |
| `source_repo` | `varchar` | Null when empty |
| `coverage_type` | `smallint` | 0 full, 1 partial |
| `report_date` | `timestamptz` | |
| `supersedes` | `bytea` | Null when zero |
| `notes_uri` | `varchar` | Null when empty |
| `attested_at` | `timestamptz` | EAS timestamp |
| `revoked_at` | `timestamptz` | Null unless revoked |
| `block_number`, `transaction_hash`, `log_index` | | Where the attestation was read |

`issuer_key_events`, one row per key-history attestation (`IssuerDomainBinding`, `IssuerKeySuccession`, `IssuerKeyCompromise`), with the same EAS columns plus `kind`, `domain`, `successor` and `since`.

The hashes are stored as keccak values, not as foreign keys into `code`. An attestation can exist before any Alliance member has verified the code, and the row must still be stored. The join to verification records goes through the indexed `code.code_hash_keccak`:

```
coverage_attestations.compiled_runtime_code_hash_keccak
  = code.code_hash_keccak
code.code_hash
  = compiled_contracts.runtime_code_hash
compiled_contracts.id
  = verified_contracts.compilation_id   (runtime_match and runtime_metadata_match true)
verified_contracts.deployment_id
  = contract_deployments.id             (chain_id, address)
```

Every step uses an existing index in the Alliance schema.

### What stays in the service

State that is not a fact read from Ethereum mainnet stays in the service's own store:

- DNS observations: each TXT record seen and when;
- the default list's evaluated state, and statuses cached per bytecode;
- simulation results.

Anyone running the service rebuilds this state from the Alliance tables, EAS and DNS.

### For the Alliance to decide

- Whether the tables live in `database-specs` and its migrations, or in a separate schema in the same database.
- Write access: whether one indexer writes for everyone, or each member may write rows it read from chain.
- Which non-Routescan Alliance endpoint exposes the rows. The milestone requires at least 100 records readable there.
- Whether the Alliance's data exports include the tables.
