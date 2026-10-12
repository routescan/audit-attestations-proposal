# Audit Coverage Attestations

> **Status: draft, seeking review.** This repository holds a specification only. No code exists yet: development starts only once the [initiative on TheDAO Security Fund](https://initiatives.thedao.fund/initiative/audit-coverage-attestations-bind-every-audit-report-to-the-e) is funded. Every format here is a proposal. Field names, hashes, schemas and endpoints are TBD until reviewed with the participating audit firms and Verifier Alliance members.
>
> Comments are welcome: [open an issue](https://github.com/routescan/audit-attestations-proposal/issues).

Bind every audit report to the exact verified bytecode it covered.

Audit reports are tied to a git commit. Explorers show code tied to an address. Nothing connects the two, so nobody can check whether the code they are about to use is the code an audit covered.

An audit coverage attestation is a signed statement from an audit firm: report R covered commit C, which compiles to bytecode B. It says only what the report covered, never that the code is safe. Firms publish it onchain through the [Ethereum Attestation Service](https://attest.org) (EAS). An open coverage service indexes the attestations, so explorers, wallets and AI agents can check what a report covered before interacting with a contract, and get a verifiable answer.

## Why this matters

Contract verification proves that published source compiles to the deployed bytecode, and every explorer shows it. It does not say whether anyone reviewed that source, or whether a review covered this build. That information sits in PDF reports on the projects' own websites.

[Ackee Blockchain's review of Rekt-listed incidents](https://ackee.xyz/blog/why-we-need-to-start-using-erc-7512-onchain-audit-representation-right-now/) found that 92 of 157 protocols were exploited through unaudited or out-of-scope code, while often carrying "audited by" logos.

With coverage attestations:

- a wallet, an explorer or an agent asks "which reports cover the bytecode this transaction executes?" and gets an answer built from signed attestations it can verify;
- audit firms get their name next to the code they reviewed, and only that code, which matters more as AI-generated reviews multiply;
- coverage never carries over to an upgraded implementation, because the new code has a different bytecode hash.

## How it works

1. The firm binds its issuer key to its domain, with a DNS TXT record and an EAS attestation from the key.
2. The firm runs the issuer tooling on the audited commit's build output. It computes the bytecode hashes of every in-scope contract.
3. The firm signs one EAS attestation per contract with its key. Either the firm submits it to Ethereum mainnet directly, or the Routescan relayer submits it and pays the gas.
4. A trust list decides which attestations count. The default list is an EAS schema governed by Verifier Alliance members, who recognise firms against published criteria. Anyone can deploy another list.
5. The coverage service reads EAS and the trust list, and joins each attestation to [Verifier Alliance](https://verifieralliance.org) verified-contract records by bytecode hash.
6. Explorers, wallets and agents query the API or the MCP tools by bytecode hash or address. Each bytecode gets one of four statuses: coverage present, out of scope, no coverage, unverified.

Anyone will be able to check an attestation without trusting the service. The verification library will read the attestation from EAS through any RPC, recompute the hashes from the compilation artifact, check the issuer's DNS binding and ask a trust list whether the attestation counts.

## Documents

- [SPEC.md](SPEC.md): attestation format, bytecode identity, statuses, issuers and trust lists, threats.
- [TOOLING.md](TOOLING.md): the Hardhat plugin and CLI that audit firms use to publish attestations.
- [API.md](API.md): the coverage service's HTTP API, MCP tools, and the records it writes to the Verifier Alliance database.
- [METRICS.md](METRICS.md): definitions behind the public metrics page and the adoption targets.

## License

[Apache-2.0](LICENSE)
