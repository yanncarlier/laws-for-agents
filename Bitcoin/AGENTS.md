# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.
> **Domain note:** This repository handles Bitcoin — private keys, transaction construction, and/or consensus-relevant logic. Bugs here can mean irreversible loss of funds, not just a bad user experience. Treat every rule below as load-bearing, not stylistic.
> **Written for:** an application/wallet/service built *on* Bitcoin (using a library such as `rust-bitcoin`, BDK, `bitcoinjs-lib`, `btcd`, or similar). See §13 if this repository is actually **Bitcoin Core** itself (C++ node/consensus code) — the rules differ meaningfully.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Language | `<Rust / Go / TypeScript / Python / C++>` |
| Bitcoin library | `<rust-bitcoin + BDK / btcsuite (btcd, btcwallet) / bitcoinjs-lib / python-bitcoinlib / libbitcoin>` |
| Node backend | `<Bitcoin Core RPC / Electrum / Esplora / Neutrino (SPV)>` |
| Network(s) | `<mainnet / testnet4 / signet / regtest>` — state which are in scope for CI vs. manual testing |
| Key management | `<software HD wallet / hardware wallet (HWI) / MPC / multisig via miniscript>` |
| Script types in use | `<P2WPKH / P2TR (Taproot) / P2WSH / miniscript / Lightning>` |
| Audit status | `<unaudited / audited by X on <date> / in progress>` |

Replace the `<...>` placeholders before relying on this file. **If `Audit status` says "unaudited," this software does not custody real user funds on mainnet, regardless of how confident anyone is in the code.**

---

## 2. Setup

```bash
# language-specific dependency install, e.g.:
# cargo build          (Rust)
# go build ./...        (Go)
# pnpm install --frozen-lockfile   (TypeScript)

cp .env.example .env
```

### Local Bitcoin node for development
Development and integration tests run against a local `bitcoind` in **regtest** mode — never against mainnet, and never against a shared/public testnet as the default dev loop.

```bash
bitcoind -regtest -daemon -fallbackfee=0.0002
bitcoin-cli -regtest createwallet "dev"
bitcoin-cli -regtest generatetoaddress 101 "$(bitcoin-cli -regtest getnewaddress)"
```

Never commit `.env`, a wallet descriptor, an xprv/xpub meant to be private, a seed phrase, or an RPC cookie/credential. If a secret is accidentally committed, treat it as **compromised and rotate it immediately** — any funds ever associated with an exposed key or seed must be considered unsafe to reuse, even after the commit is removed from history.

---

## 3. Commands

Fill in the actual project scripts; keep this table in sync with `package.json`/`Makefile`/`justfile`.

| Task | Command |
| --- | --- |
| Build | `<cargo build / go build ./... / pnpm build>` |
| Unit tests | `<cargo test / go test ./... / pnpm test>` |
| Single test | `<cargo test tx_builder::tests::rejects_dust>` |
| Integration tests (regtest) | `<pnpm test:integration>` — requires local `bitcoind -regtest` |
| Lint | `<cargo clippy -- -D warnings / golangci-lint run / eslint>` |
| Format | `<cargo fmt / gofmt -l . / prettier --write>` |
| Format check (CI) | `<cargo fmt --check / gofmt -l . / prettier --check>` |
| Type check (if applicable) | `<tsc --noEmit>` |
| Full pre-push gate | `<pnpm verify>` — wraps build → lint → unit tests → integration tests |

**The full gate must pass before any code is considered done — no exceptions for "just a small change."**

---

## 4. Directory structure

```
.
├── src/ (or cmd/, internal/)
│   ├── wallet/            # key derivation, descriptor handling, signing
│   ├── tx/                # transaction construction, coin selection, fee logic
│   ├── script/             # script/miniscript templates, address types
│   ├── rpc/ (or node/)     # Bitcoin Core RPC client / Electrum / Esplora client
│   ├── psbt/                # PSBT creation, parsing, (partial-)signing
│   └── net/                # p2p or external network-facing code, if any
├── test/
│   ├── unit/                # pure logic, no node required
│   ├── integration/          # exercises a real regtest bitcoind
│   └── vectors/               # BIP test vectors (BIP32/39/173/174/340/341, etc.)
├── scripts/                  # dev tooling: regtest setup, fee estimation checks
└── AGENTS.md
```

Test vectors from the relevant BIPs are checked in and run in CI, not just eyeballed once — see §7.

---

## 5. Style

- Follow the idiomatic style/lint config for the project's language (`rustfmt`/`clippy`, `gofmt`/`golangci-lint`, `eslint`/`prettier`) — don't hand-format.
- Amounts are represented in **satoshis as integers** internally. Never do arithmetic on BTC as a floating-point number anywhere money is actually moved — floating point loses precision at exactly the boundary that matters. Convert to a display-only BTC string only at the UI/formatting edge.
- Be explicit and unambiguous about units in every function signature and variable name that touches value or fees: `amountSats`, `feeRateSatPerVb`, not `amount`, `fee`.
- Byte order matters and is a classic source of bugs: txids are commonly displayed reversed (big-endian display of a little-endian internal hash). Name variables to make the representation explicit (`txidBytesLE` vs. `txidHexDisplay`) rather than relying on context.
- Naming and file layout otherwise follow the language's own convention (Rust: `snake_case` functions, `PascalCase` types; Go: exported `PascalCase`, package-private `camelCase`; TypeScript: `camelCase`/`PascalCase` per the existing config).
- No magic numbers for consensus-relevant constants (dust limits, sequence values, timelock encodings) — name them and cite the BIP or Bitcoin Core source they come from.

---

## 6. Security — non-negotiable rules

Read this section fully before touching key handling, transaction construction, or anything that moves funds.

### Private keys and seeds
- **Never log a private key, xprv, seed phrase, or PSBT containing private key material.** Configure logging/telemetry redaction for these explicitly rather than assuming default log scrubbing catches them.
- Never transmit a private key or seed over the network in any form, including to "your own" backend for convenience — signing happens client-side / in the secure environment that holds the key, full stop.
- Prefer hardware wallets or well-audited signing devices/HSMs for anything beyond a hot wallet holding trivial amounts. Document explicitly, per environment, what holds the signing key and what the threat model assumes.
- Use vetted key-derivation and encoding: BIP32 (HD derivation), BIP39 (mnemonic), BIP43/44/49/84/86 (derivation paths per script type), SLIP-0010 where relevant. Do not hand-roll key derivation or encoding.
- Wipe sensitive material (seeds, private keys, mnemonics) from memory as soon as it's no longer needed, in languages where that's meaningful (e.g. zeroize buffers in Rust/Go rather than relying on GC).
- Any randomness used for key generation must come from a cryptographically secure source (the OS CSPRNG via the language's standard crypto library) — never `Math.random()` or an unseeded/weak PRNG, ever, for anything touching keys or nonces.

### Nonce reuse and signature safety
- **Never reuse an ECDSA nonce (`k`) across signatures**, and never let user-supplied or externally influenced data determine the nonce. Nonce reuse or predictable nonces directly leaks the private key. Use RFC 6979 deterministic nonces (the library default in every well-audited Bitcoin signing library) — don't override this with custom nonce generation.
- For Taproot/Schnorr (BIP340) signing, use the audited library implementation as-is; do not hand-implement the signing algorithm.

### Transaction construction
- **Validate every amount and change output before broadcasting**, and hard-fail (don't silently "fix") if: the fee is implausibly high relative to the amount being sent, a change output would create unexpected dust, or computed inputs/outputs don't balance to the expected fee.
- Guard against unintentional **fee sweeps**: if the input value minus output value drastically exceeds the intended fee, treat that as a bug to fix, not a value to broadcast. This is the single most common catastrophic bug class in wallet software — always show/compute the fee explicitly rather than deriving it implicitly and trusting the difference.
- Respect the network's dust limit (currently 546 sats for a standard P2PKH-sized output; varies by output type). Never create an output that risks being unspendable or non-relayable due to dust rules.
- When constructing a transaction that spends multiple UTXOs, be deliberate about coin selection and its privacy implications (avoid unnecessary address reuse and unintentional UTXO clustering that links unrelated funds) — document the coin-selection algorithm in use.
- **RBF (Replace-By-Fee) and locktime/sequence values must be set deliberately, not left at a library default without review** — signal RBF intentionally if the product needs fee-bumping, and understand the finality implications either way.
- PSBTs (BIP174/BIP370) are the preferred format for any transaction that's constructed in one context and signed in another (e.g. hardware wallet, multisig cosigner, air-gapped signer). Never invent a custom unsigned-transaction interchange format when PSBT already covers the need.

### Address and script correctness
- Validate address format and network (mainnet/testnet/signet/regtest) explicitly before use — sending to a mainnet-looking address parsed under testnet rules (or vice versa) is a real, funds-losing bug class. Fail loudly on a network mismatch; never silently coerce.
- For Taproot outputs, be careful with key-path vs. script-path spending assumptions, and verify script trees / merkle proofs are correctly constructed if using script-path spends — a subtle miniscript/script-tree bug can lock funds unspendably or, worse, spendably by an unintended party.
- When accepting a `descriptor` (output descriptor) from external input, validate it fully (correct checksum, expected script type, expected derivation) before trusting it to generate addresses users will send funds to.

### External data and untrusted input
- Treat data from any external RPC/API (Electrum server, Esplora/block explorer, third-party fee estimator) as **untrusted input**, not ground truth — a malicious or compromised server can lie about balances, fee rates, or transaction confirmation status. Where possible, verify against a trusted full node (SPV proof, or your own `bitcoind`) rather than trusting a single third-party source for anything security-relevant.
- Validate all data parsed from a raw transaction, block, or PSBT (lengths, script sizes, witness structure) before acting on it — a hostile or malformed input should be rejected cleanly, not cause a panic/crash or, worse, be silently misinterpreted.
- Don't trust user-facing "confirmations" count from a single unverified source when the product's guarantees depend on it — document the confirmation policy and reorg-handling assumptions explicitly (see below).

### Reorgs and confirmation policy
- Explicitly define and document how many confirmations the product requires before treating a payment as final, and handle chain reorganizations: a transaction can become unconfirmed after appearing confirmed. Code that marks something "paid" or "settled" must account for this, not assume monotonic confirmation counts.
- Handle double-spend and mempool-eviction scenarios for zero-conf or low-conf flows explicitly if the product relies on them; document the accepted risk if it does.

### General
- **No custom cryptographic primitives.** Use the language's standard, audited Bitcoin library for all key derivation, signing, script construction, and encoding (Base58Check, Bech32/Bech32m, PSBT serialization). Do not hand-roll any of these, even "just for a small helper."
- Pin exact versions of Bitcoin-related dependencies (see §9) — a library version bump can silently change signing or serialization behavior in security-relevant ways.
- **Any software that custodies real user funds on mainnet must go through an external security audit before launch, and after any change to key handling or transaction-construction logic.** No amount of internal review or test coverage substitutes for this.

---

## 7. Testing

- Unit test all pure logic (coin selection, fee calculation, script/address construction, PSBT parsing) with no live node required.
- Validate against the **official BIP test vectors** for every BIP the project implements (BIP32, BIP39, BIP173/350 Bech32/Bech32m, BIP174 PSBT, BIP340/341/342 Taproot/Schnorr, etc.) — check these vectors into `test/vectors/` and run them in CI; don't rely on hand-written test cases alone for consensus-sensitive encoding/crypto logic.
- **Integration tests run against a real `bitcoind -regtest`** (not solely mocks) for anything that constructs and broadcasts transactions, so the code is proven against real consensus rules and real mempool/RPC behavior, not just an idealized mock.
- Explicitly test edge cases that are common real-world bug sources: dust outputs, maximum-size transactions, zero-value OP_RETURN outputs, RBF replacement sequences, reorg handling, malformed/truncated input, and network-mismatched addresses.
- Fuzz test any parser touching untrusted external input (raw transactions, PSBTs, network messages, descriptors) — malformed input must fail safely, never panic/crash or be silently misparsed.
- Every bug fix — especially any fix touching amount/fee logic, key derivation, or address validation — ships with a regression test that reproduces the bug and fails without the fix.
- Never run a test against mainnet with real keys/funds as part of the normal dev loop. Manual mainnet verification (if ever needed) uses a small, disposable, isolated amount and is a deliberate, documented, one-off act — not part of CI.

---

## 8. Fee estimation and mempool interaction

- Use a live fee estimate (from the connected node/service) rather than a hardcoded fee rate, and make the fallback behavior explicit and conservative if the estimate is unavailable.
- Handle fee-estimation failure gracefully — don't silently fall back to a fee rate low enough that the transaction could remain unconfirmed indefinitely, and don't silently overpay dramatically either. Surface the actual fee rate to the caller/user rather than hiding it.
- If the product supports fee bumping, implement it via a well-understood mechanism (RBF per BIP125, or CPFP) rather than a bespoke replacement scheme, and test that the replacement transaction is valid and actually propagates.

---

## 9. Dependencies

- Prefer established, widely audited Bitcoin libraries (`rust-bitcoin`/BDK, `btcsuite`, `bitcoinjs-lib`, `python-bitcoinlib`, `libbitcoin`) over reimplementing consensus-relevant primitives.
- Pin exact dependency versions for anything touching keys, signing, or transaction/script encoding — a minor-version bump in a crypto library is a security-relevant event, not a routine update, and should be reviewed as such rather than auto-merged.
- Before adding any new dependency that touches key material or transaction construction, check its audit history, how widely it's used in production wallets/services, and its maintenance status; note the justification in the PR.
- Avoid adding a second library for the same job (e.g. two different Bech32 implementations) — consolidate on one to avoid subtle encoding mismatches between them.

---

## 10. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `audit/short-description`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `fix(tx): prevent fee-sweep on max-send`.
- PR description states: what changed, the security/funds-safety implications explicitly (again, "no change to fund-handling logic" is a valid statement, but state it), which network(s) it was tested against, and whether it touches key handling, signing, or fee logic.
- Any change touching private keys, signing, transaction construction, fee calculation, or address/network validation needs at least one reviewer explicitly signing off on the security implications, not just code style.
- Never force-push to `main`. Never amend or rebase a commit someone else may have pulled.
- Do not commit: `.env`, seeds/xprvs, wallet.dat or descriptor files with real key material, RPC cookies/credentials, `target/`/`node_modules/`/build artifacts.

---

## 11. Rules for agents

**Do**
- Read the surrounding module and its tests before editing; match existing conventions (unit handling, error types, coin-selection approach) rather than introducing a second one.
- Treat §6 (Security) as binding, not advisory — apply it even when not explicitly reminded to.
- Run the full verify gate (build, lint, unit tests, regtest integration tests) and report actual output before claiming success.
- Flag explicitly, in the response, any change that touches: private key handling, transaction construction, fee/amount calculation, address or network validation, or PSBT signing — even if asked to make an unrelated change nearby.
- Default to **regtest** for any example, test, or script you write or run; never generate an example that uses a real mainnet address or asks for a real seed/private key.
- Ask before: changing coin-selection logic, changing derivation paths, changing how fees are calculated, or adding a new dependency that touches key material.

**Don't**
- Don't write or modify code touching amounts, fees, keys, or signing without also writing/updating tests for it in the same change.
- Don't hand-roll cryptography, encoding (Base58Check/Bech32), or key derivation "just this once" — use the existing library function even if a custom version looks simpler.
- Don't generate or suggest a real-looking seed phrase, private key, or xprv, even as a placeholder — use obviously-fake/documented test vectors (e.g. the standard BIP39 test mnemonic) instead.
- Don't treat testnet/regtest code as exempt from these rules — bugs in fee/amount/key logic on testnet are the same bugs that will exist on mainnet.
- Don't mark work complete with a failing test, an un-investigated lint/fuzz finding, or missing tests on new amount/fee/key-handling logic.

---

## 12. Definition of done

1. Full build and lint gate passes with no warnings the project doesn't already accept.
2. Unit tests pass, including relevant BIP test vectors.
3. Integration tests against local regtest `bitcoind` pass for any change touching transaction construction or broadcast.
4. New logic touching amounts, fees, keys, or addresses has explicit tests for edge cases (dust, max-send/fee-sweep, network mismatch, malformed input).
5. No private key, seed, or sensitive material appears in logs, test fixtures, or the diff.
6. The PR description states the funds-safety implications of the change and which network(s) it was verified against.
7. For anything shipping to a mainnet-custodying release: an external audit has been completed and its findings resolved.

---

## 13. If this is actually Bitcoin Core (node/consensus code)

Everything above assumes an application built *on top of* Bitcoin. If this repository is **Bitcoin Core itself** (C++, `src/`, consensus/validation/p2p/wallet subsystems), the rules above still apply in spirit but the specifics differ substantially:

- **Consensus code is held to a categorically higher bar.** Any change to `src/consensus/`, `src/validation.cpp`, script interpretation, or anything affecting what blocks/transactions are considered valid must preserve exact backward-compatible behavior unless it's an explicitly designed, reviewed, and (for anything consensus-critical) soft-fork-gated change. An accidental consensus change can split the network.
- Follow Bitcoin Core's own [contributor documentation](https://github.com/bitcoin/bitcoin/blob/master/CONTRIBUTING.md) and [developer notes](https://github.com/bitcoin/bitcoin/blob/master/doc/developer-notes.md) as the primary style/process authority — this file's language-agnostic style guidance (§5) does not apply; Bitcoin Core has its own detailed C++ style guide.
- Build and test via the project's own tooling (`./autogen.sh && ./configure && make`, or CMake in newer versions; `make check`; `test/functional/test_runner.py` for the Python-based functional test suite) rather than the generic commands in §3.
- Changes go through Bitcoin Core's own review process (multiple reviewers, ACK/NACK conventions, extensive functional and fuzz testing) — an AI agent should treat any consensus-adjacent change as something that requires human expert review before merge, full stop, regardless of how confident the change looks.
- If you're working in this repository, ask for a dedicated Bitcoin Core `AGENTS.md` built around its actual subsystem boundaries (consensus, net, wallet, RPC, GUI) rather than relying on this application-focused file.
