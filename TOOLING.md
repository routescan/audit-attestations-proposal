# Audit Coverage Attestations: issuer tooling

> **Status: draft.** Nothing described here exists yet. Development starts only once the initiative is funded. Package names, commands and flags are TBD.

Audit firms publish attestations with two tools built on the verification library:

- a **CLI** that reads Foundry build output (`out/` and `out/build-info/`, written with `forge build --build-info`) or any standard-JSON compiler input and output;
- a **Hardhat plugin** that reads Hardhat's build info and exposes the same commands as Hardhat tasks.

Both compute hashes with the verification library, so the issuer and every verifier use the same code. Examples below use `aca` as the CLI name.

## 1. Commands

| Command | Does | Key needed |
| :--- | :--- | :--- |
| `aca hash` | Prints both compiled hashes for each contract in the build output | No |
| `aca attest` | Builds, signs and publishes one attestation per in-scope contract | Issuer key |
| `aca bind` | Prints the DNS TXT record for a key and publishes its `IssuerDomainBinding` | Issuer key |
| `aca rotate` | Publishes an `IssuerKeySuccession` from the old key to a new key that already has its binding | Old key |
| `aca compromise` | Publishes an `IssuerKeyCompromise` with a `since` time | Compromised key |
| `aca revoke` | Revokes attestations by UID | Issuer key |
| `aca history` | Lists every onchain action of a key, for the review after a [key compromise](SPEC.md#key-compromise) | No |
| `aca verify` | Checks an attestation against a compilation artifact, the issuer's DNS binding and a trust list | No |

## 2. Attesting a report

`aca attest` takes:

| Input | Becomes |
| :--- | :--- |
| The report file | `reportHash`, keccak256 of the file's bytes |
| `--report-uri` (optional) | `reportURI` |
| `--report-date` | `reportDate` |
| `--repo` (optional) | `sourceRepo`. Never filled from the git remote, so a private repository is not disclosed by default. |
| A scope file | The in-scope contracts, by fully qualified name, each `full` or `partial` |
| `--supersedes`, `--notes-uri` (optional) | `supersedes`, `notesURI` |

`sourceCommit` is the commit the build was made from. The CLI refuses to attest when:

- the working tree has uncommitted changes, or `HEAD` is not the named commit;
- a source file in the build info differs from the same path at that commit;
- a contract in the scope file has no bytecode in the build output, such as an interface or an abstract contract.

Source files that are not in the repository, such as packages under `node_modules`, are listed with their hashes before signing.

Contracts not named in the scope file are not attested. The CLI prints every attestation it will sign and asks for confirmation; `--yes` skips the prompt in pipelines.

## 3. Signing and submission

The CLI signs with the EAS SDK (see [SPEC §2](SPEC.md#signing-and-submission)). The key comes from an encrypted keystore file or a hardware wallet. For other signers, the CLI writes the EIP-712 typed data to a file and reads the signatures back.

`--submit` chooses where the signed attestations go:

| Value | Effect |
| :--- | :--- |
| `direct` | Submits to EAS on Ethereum mainnet in one transaction. The firm pays the gas. |
| `relayer` | Sends them to the Routescan relayer, which pays the gas. |
| `none` | Writes the signed attestations to a file, for submission later by any party. |

After submission, the CLI writes a manifest listing each contract, its hashes and its attestation UID, for inclusion in the published report.

## 4. Audit-report pipelines

Pipelines such as Solodit, Code4rena, Sherlock and Cantina call the verification library and the attestation builder as a JavaScript API, without the CLI. The API takes the same inputs and returns the typed data to sign and, after signing, the manifest.

## 5. Open points

- Supported Hardhat versions.
- Supported hardware wallets.
- The scope file format.
