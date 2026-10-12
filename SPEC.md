# Audit Coverage Attestations: specification

> **Status: draft.** Formats and field names are TBD and will be finalised with the reviewing audit firms and Verifier Alliance members.

## 1. Scope

An attestation asserts one checkable fact: report R, issued by firm F, covered source commit C, which compiles to bytecode hash B, with contract N in scope. Most audit reports already name the commit and the contracts in scope, and the attestation adds the bytecode they compile to.

An attestation does not assert that a contract is safe, secure or audited. It says nothing about:

- ownership, admin keys or timelocks
- immutable values, constructor arguments or configuration
- economic parameters
- any bytecode without a published, exact-match verified source

A deployment of covered code with a malicious configuration still shows coverage present. Every response carries a fixed caveat that says so.

This specification does not cover:

- proxy resolution or upgrade tracking (see [section 4](#4-resolving-executing-code))
- developer-declared audit links or any other self-reported input
- rating audit firms or protocols
- hosting reports

The words `audited`, `safe` and `secure` never appear in a status, field or label defined by this specification. A CI lint enforces this.

## 2. Attestation

One attestation covers one compiled contract from one compiler run and points at one report. A report that covers ten contracts produces ten attestations with the same `reportHash`.

Attestations are published onchain on Ethereum mainnet through EAS, as revocable attestations. EAS supplies a global UID, an onchain timestamp and revocation.

```solidity
struct CoverageAttestation {
  bytes32 reportHash;            // keccak256 of the report file
  string  reportURI;             // optional; ipfs:// preferred
  bytes32 sourceCommit;          // git commit hash, left-padded with zero bytes
  string  sourceRepo;            // optional
  bytes32 compiledRuntimeCodeHash;  // see section 3
  bytes32 compiledCreationCodeHash; // see section 3
  string  fullyQualifiedName;    // e.g. src/Vault.sol:Vault
  uint8   coverageType;          // 0 full, 1 partial
  uint64  reportDate;            // when the report was made public, unix seconds
  bytes32 supersedes;            // UID of the attestation this replaces, or zero
  string  notesURI;
}
```

The EIP-712 typed data, the scope semantics and the EAS schema string are TBD. The EAS schema will be registered on Ethereum mainnet.

### Public reports and client consent

A firm attests only a report that is public, and only with its client's consent to publishing the attestation. Consent is a condition the firm accepts by signing; the protocol cannot check it.

`reportURI` and `sourceRepo` are optional, and an empty string means absent. Many audited repositories are private, and the attestation does not need them: `reportHash` pins the report, and the verified source comes from the Verifier Alliance.

### Key history

Three further schemas record what an issuer key states about itself: its domain, its successor and its compromise. Every trust list reads the same key history (see [section 6](#6-issuers)). The schemas will be registered on Ethereum mainnet.

| Schema | Fields | Attester | `recipient` | `refUID` | Revocable |
| :--- | :--- | :--- | :--- | :--- | :--- |
| `IssuerDomainBinding` | `string domain` | The issuer key | Zero | Zero | Yes |
| `IssuerKeySuccession` | `address successor` | The old key | `successor` | The successor's `IssuerDomainBinding` UID | No |
| `IssuerKeyCompromise` | `uint64 since`, `string notesURI` | The compromised key | Zero | Zero | No |

The three schemas share one resolver, which cannot be upgraded. It rejects:

- an `IssuerDomainBinding` from a key that already has a non-revoked one;
- a second `IssuerKeySuccession` from the same key;
- an `IssuerKeySuccession` whose `refUID` is not an `IssuerDomainBinding` from `successor`;
- any key-history attestation from a key that has published an `IssuerKeyCompromise`.

`CoverageAttestation` has no resolver.

### Signing and submission

The firm signs a delegated EAS attestation with its issuer key. The attester recorded onchain is always that key, whoever submits the transaction.

- The firm can submit the signed attestation to EAS directly.
- During the grant, a Routescan relayer submits it and pays the gas for DNS-bound issuers. The relayer is rate-limited and runs under a published policy.

Nothing depends on the relayer after the grant.

## 3. Bytecode identity

An attestation identifies code by what the compiler emits for the audited commit, not by what is deployed. Deployed code differs from compiler output only in regions filled at deployment: immutable values and linked library addresses. Those values are unknown at audit time and outside the claim (see [section 1](#1-scope)).

| Hash | Input | Use |
| :--- | :--- | :--- |
| `compiledRuntimeCodeHash` | Runtime code as emitted by the compiler, with deploy-time regions zeroed | Matching deployed contracts |
| `compiledCreationCodeHash` | Creation code as emitted by the compiler, with deploy-time regions zeroed, without constructor arguments | Checks before deployment |

Both hashes are keccak256 of the code that the [Verifier Alliance database](https://github.com/verifier-alliance/database-specs) stores in `compiled_contracts`, where library and immutable regions are zero. The hashes equal `code.code_hash_keccak` for that compilation's `creation_code_hash` and `runtime_code_hash`.

The CBOR metadata stays in both hashes. Code whose metadata differs was built from different sources or settings, and does not match.

A deployed contract matches an attestation when a `verified_contracts` row links its deployment to a compilation whose runtime code hash is `compiledRuntimeCodeHash`, with `runtime_match` and `runtime_metadata_match` both true. That row's transformations already reconcile the deploy-time regions with the onchain code. No hash is computed from onchain bytecode.

Issuer tooling computes the hashes from the audited commit's build output. The verification library recomputes them from a compilation artifact and checks an attestation against them. EOF is out of scope.

Because coverage binds to compiler output, one attestation covers the same build across deployments, factory-deployed instances and chains, wherever an Alliance explorer has an exact-match verification of it. An upgraded implementation has no coverage until a report covers its bytecode.

## 4. Resolving executing code

The service answers by bytecode hash. It does not resolve proxies or track upgrades. Explorers, wallets and agents decide which code executes, using their own proxy detection or a transaction simulation.

- A query by chain and address answers for the bytecode stored at that address. For a proxy, that is the proxy's own code, not its implementation.
- The transaction-simulation MCP tool finds the code a transaction executes by simulating it (see [API.md](API.md#2-mcp-tools)).
- Routescan's explorer resolves proxies with its own detection, and shows proxies whose current implementation is not covered.

## 5. Status vocabulary

| Status | Label | Meaning |
| :--- | :--- | :--- |
| `present` | Coverage present | A counted attestation covers this bytecode in full. |
| `out_of_scope` | Out of scope | A counted attestation covers this bytecode only in part (`coverageType` partial): some of its code is outside the report's scope. |
| `none` | No coverage | Verified source, no counted attestation. |
| `unverified` | Unverified | No exact-match verified source. Coverage is not evaluated. |

A trust list decides which attestations are counted (see [section 6](#trust-lists)). Statuses use the list the service instance is configured with. Whether an attestation counts depends only on onchain data. DNS decides only which domain is displayed.

When several attestations match, the strongest status applies: `present`, then `out_of_scope`.

A status covers the queried bytecode only, not the libraries it links. A contract can show `present` while a linked library has no coverage. Library code runs by delegation, so a caller that needs coverage of everything that executes checks each linked library as well. Responses list the linked libraries by fully qualified name and address, as recorded in the matching `verified_contracts` row.

Only counted attestations contribute to `present` or `out_of_scope`. A revoked attestation is not counted unless a reinstatement ignores the revocation (see [Key compromise](#key-compromise)). Attestations that are not counted are still returned, each with its issuer's domain and recognition state.

## 6. Issuers

Anyone can publish an attestation. The protocol has no allowlist.

### Domain binding

A firm binds each issuer key to a domain in both directions:

- a DNS TXT record at `_audit-issuer.<domain>`, one per key: `v=aca1; key=eip155:1:<EIP-55 address>`;
- an `IssuerDomainBinding` attestation from that key (see [section 2](#key-history)).

A key's domain is displayed only when both agree. Every attestation is displayed with its issuer's domain, so impersonation is visible.

- Tags are separated by `;`, with surrounding whitespace ignored. Unknown tags are ignored. A record with an unknown `v` value is invalid.
- The address must carry a valid EIP-55 checksum, or the record is invalid.
- `domain` is the exact name the record sits under, subdomains included: lowercase ASCII, internationalised names in `xn--` form, no trailing dot.
- DNSSEC is recommended, not required.
- A key's domain is taken from the last key in its succession chain (see [Key rotation](#key-rotation-and-revocation)). It shows as unconfirmed unless that key's record and binding agree.
- The verification library checks DNS at verification time. The indexer rechecks periodically and records when it last saw each record.
- Removing a record does not revoke attestations. Revocation happens only through EAS.

### Key rotation and revocation

A planned rotation adds a TXT record and an `IssuerDomainBinding` for the new key. The old key then publishes an `IssuerKeySuccession` referring to that binding, and the old key's record is removed. Attestations from the old key keep their domain through the succession. A key can be rotated again in the same way, which extends its succession chain. After a compromise, the old key cannot be trusted, and the firm follows [Key compromise](#key-compromise) instead.

A firm can revoke its own attestations. Revoked attestations stay visible in history.

### Trust lists

A trust list is a contract that answers, for any attestation UID, whether that attestation is counted. It reads the attestation and the key history from EAS. Anyone can deploy a trust list. A coverage service instance applies one list, set in its configuration. A caller that wants another list runs its own instance, or checks attestations against that list with the verification library.

### Default list

The default list is governed by Routescan and at least 2 other Verifier Alliance members. It is an EAS schema whose resolver holds the list state and implements the trust-list interface. Members act through non-revocable `ListAction` attestations, signed with their own keys and submitted directly or as delegated attestations.

| Field | Type | Meaning |
| :--- | :--- | :--- |
| `action` | `uint8` | 0 add, 1 remove, 2 suspend, 3 lift, 4 reinstate |
| `issuer` | `address` | The issuer key acted on |
| `newKey` | `address` | The replacement key for a reinstatement, otherwise zero |
| `time` | `uint64` | `since` for a suspension, `cutoff` for a reinstatement, otherwise zero |
| `excluded` | `bytes32[]` | For a reinstatement, the UIDs it excludes in ascending order, otherwise empty |

"2 members acting together" means `ListAction`s with identical fields from 2 distinct member keys. The first is pending until a second matches it, and expires after 14 days. A second member confirms a suspension the same way, within 72 hours. The resolver rejects actions from keys that are not members, `excluded` lists out of order, and actions that break the rules below.

The members are the owners of an upgrade Safe, which is the admin of the resolver's upgradeable proxy. The Safe's threshold is at least 3, and Routescan holds one owner key, so an upgrade needs at least 2 members other than Routescan.

The binding proves that a key and a domain belong together. Recognition adds that the firm behind them meets the criteria below. It says nothing about audit quality. Recognition binds to an issuer key, not to a domain.

| Action | Rule |
| :--- | :--- |
| Add an issuer | 2 members acting together |
| Remove an issuer | 2 members acting together. Takes effect 14 days after the second member's `ListAction`. |
| Suspend on suspected key compromise | Any single member, with immediate effect, naming a time `since`. Lapses after 72 hours unless a second member confirms it. Once lapsed, suspending that key again needs 2 members. An `IssuerKeyCompromise` suspends the key until reinstatement. |
| Lift a suspension | 2 members acting together. An `IssuerKeyCompromise` cannot be lifted. |
| Reinstate an issuer under a new key | 2 members acting together (see [Key compromise](#key-compromise)) |
| Recognise a successor | Automatic, 72 hours after an `IssuerKeySuccession` from a recognised key, unless in that time the key is suspended or a removal of it is pending |

Suspending a key also suspends every successor it named at or after the suspension's `since` time. Routescan cannot add, remove or reinstate an issuer alone.

Recognition criteria are objective and published:

- a named legal entity;
- a key bound to the firm's domain;
- coverage attestations from that key for at least 5 distinct reports, each public at its `reportURI`.

A firm applies before it is recognised: its attestations are already published and returned as unrecognised, so members check the reports onchain. Members check the legal entity.

### Key compromise

| Step | Who | Action |
| :--- | :--- | :--- |
| 1. Suspend | The firm, through `IssuerKeyCompromise`, or any single member | Nothing from the key counts |
| 2. Confirm | A second member, within 72 hours | A matching `ListAction`, needed only for a member's suspension |
| 3. New key | The firm | TXT record and `IssuerDomainBinding` for the new key; the old key's record is removed |
| 4. Review | The firm | The issuer CLI lists every onchain action of the old key. The firm picks a cutoff and the earlier attestations it does not vouch for. |
| 5. Reinstate | 2 members acting together | A reinstate `ListAction` from each |

A reinstatement names the compromised key as `issuer`, its replacement as `newKey`, a `cutoff` as `time`, and the attestation UIDs before `cutoff` that the firm does not vouch for as `excluded`. `cutoff` is an EAS time. Once it takes effect:

- the new key is recognised, and the old key never is again;
- the old key's attestations before `cutoff`, other than `excluded`, count again and are displayed with the new key's domain;
- the old key's actions at or after `cutoff` are ignored: attestations, revocations, successions and bindings.

A later reinstatement for the same `oldKey` replaces an earlier one. Between suspension and reinstatement, the firm's attestations are not counted, so a contract with no other counted attestation shows `none`.

## 7. Threats

| Threat | Handling |
| :--- | :--- |
| A fake issuer attests a scam contract | Its attestations never count toward `present`. They are shown with the issuer's domain and marked unrecognised. The default list needs 2 members to recognise an issuer. |
| A backdated report | The EAS timestamp records publication time. `reportHash` pins the report file. |
| A silent upgrade | The new implementation has a different bytecode hash and no coverage until a report covers it. |
| Issuer key compromise | Immediate suspension by the firm or any single member, so nothing from the key counts. 2 members reinstate the firm under a new key with a cutoff: the old key's earlier attestations count again, and its later actions are ignored, including any succession the attacker published. |
| A stolen key revokes a firm's attestations | Revocations at or after the reinstatement cutoff are ignored. |
| Same build with different immutables or linked libraries | The compiled hashes match, since deploy-time values are outside the claim. A status does not extend to linked libraries (see [section 5](#5-status-vocabulary)). |
| A compromised member key | A single member's suspension lapses after 72 hours and cannot be renewed alone. Recognising, removing or reinstating an issuer needs a second member key. The other members remove the key as an owner of the upgrade Safe. |
| DNS hijack of a firm's domain | Recognition binds to the key. A new key published under the hijacked domain shows the domain but stays unrecognised until 2 members act. |
| Scope gaming | One attestation per contract. Partial coverage is `out_of_scope`, never `present`. |
| Routescan as gatekeeper | Apache-2.0 service that runs with no Routescan credentials. Whether an attestation counts depends only on onchain data, and Verifier Alliance data is public, so anyone can rebuild the index. Any trust list can replace the default one (see [Trust lists](#trust-lists)). The default list needs 2 members, and a handover clause covers operations. The control points are the upgrade Safe and the npm publisher of the verification library. |
| Relayer outage or withdrawal | Firms submit their signed attestations to EAS directly. |

## 8. Relation to ERC-7512

[ERC-7512](https://eips.ethereum.org/EIPS/eip-7512) (2023) defined a signed audit summary but has not left Draft. It binds each summary to one deployment address, and there is no shared place to publish, index or display it.

This specification keeps the signed-summary idea and changes four things:

- It binds to bytecode hashes instead of a `(chainId, address)` deployment.
- It publishes through EAS, with revocation.
- It binds issuer keys to domains.
- It narrows the claim to which report covered which code.

Draft field mapping:

| ERC-7512 | This spec | Notes |
| :--- | :--- | :--- |
| `auditHash` | `reportHash` | Same meaning |
| `auditUri` | `reportURI` | Same meaning |
| `issuedAt` | `reportDate` | EAS timestamp records publication time separately |
| `auditor` (name, uri, authors) | EAS attester and DNS-bound domain | Identity comes from the key and domain binding |
| `auditedContract` (chainId, deployment) | `compiledRuntimeCodeHash`, `compiledCreationCodeHash`, `fullyQualifiedName` | Compiler output instead of address |
| `ercs` | none | TBD |

## 9. Conformance

The verification library will be published on npm. It will ship:

- test vectors for at least 20 verified Ethereum mainnet contracts, at least 2 of them Vyper;
- CI that runs in public;
- the default list's address, so checking whether an attestation counts takes one call to the list;
- a check that reports any change to the default list's implementation.

Before mainnet deployment, a firm with no relationship to Routescan reviews the hash computation, the verification code, the key-history resolver and the default list's resolver. The review is published in full, and every high or critical finding is resolved, with fix commits linked.

## 10. Review

This specification is open for review by Verifier Alliance members and audit firms. Comments go in [GitHub issues](https://github.com/routescan/audit-attestations-proposal/issues), one topic per issue.

