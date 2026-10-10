# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.
> **Domain note:** This is a TRON smart contract repository handling real (or eventually real) value. TRON runs the TVM, which is close to but **not identical** to the EVM — several assumptions that hold on Ethereum do not hold here (see §6). Mistakes here are frequently irreversible and public. Treat every rule below as load-bearing, not stylistic.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Solidity version | `<0.8.x — check TRON's supported compiler versions before pinning>` |
| Framework | `<TronBox / TronIDE / Hardhat + tronbox-hardhat compat>` |
| Node.js | `>=20.11` — see `.nvmrc` |
| Package manager | `pnpm` |
| Target network(s) | `<Mainnet / Shasta testnet / Nile testnet / local TRE (TRON private chain)>` |
| Client library | `<TronWeb / TronGrid API / TronStation>` |
| Token standard(s) | `<TRC-10 (native, no contract) / TRC-20 / TRC-721 / TRC-1155>` |
| Upgradeability | `<none — immutable / proxy pattern (TVM-compatible)>` |
| Audit status | `<unaudited / audited by X on <date> / in progress>` |

Replace the `<...>` placeholders before relying on this file. **If `Audit status` says "unaudited," no contract from this repo is deployed to mainnet with real funds at stake, ever, regardless of how confident anyone is in the code.**

---

## 2. Setup

```bash
nvm use
pnpm install --frozen-lockfile

# TronBox (most common toolchain)
pnpm add -g tronbox
cp .env.example .env
tronbox compile
```

### Local development network
Prefer a local **TRE (TRON private chain, `java-tron` in dev mode)** or the **Shasta/Nile testnets** for development and CI — never develop or run automated tests against Mainnet.

```bash
# Example: running java-tron locally in a single-node dev config
# (see project-specific docs/scripts for the exact image/command in use)
docker run -it -p 9090:9090 <tron-quickstart-image>
```

Never commit `.env`, a private key, a mnemonic, or a TronGrid/TronStation API key. If a secret is accidentally committed, treat it as **compromised and rotated immediately** — removing it from a later commit does not undo exposure in git history.

---

## 3. Commands

| Task | Command |
| --- | --- |
| Compile | `tronbox compile` |
| Test | `tronbox test` |
| Test (specific file) | `tronbox test ./test/Token.test.js` |
| Migrate/deploy (per network) | `tronbox migrate --network shasta` |
| Console | `tronbox console --network shasta` |
| Lint (Solidity) | `<solhint 'contracts/**/*.sol'>` |
| Lint (JS/TS tooling) | `<eslint>` |
| Format | `<prettier + prettier-plugin-solidity>` |
| Format check (CI) | `<prettier --check>` |
| Static analysis | `<slither . — see §6 caveat on EVM-vs-TVM false positives>` |
| Full pre-push gate | `pnpm verify` (wraps: compile → lint → test → format check) |

**`pnpm verify` (or the equivalent full gate) must pass before any code is considered done — no exceptions for "just a small change."**

---

## 4. Directory structure

```
.
├── contracts/
│   ├── interfaces/           # I-prefixed interfaces, no implementation
│   ├── libraries/             # internal libraries, pure logic where possible
│   ├── core/                  # primary protocol contracts
│   ├── token/                 # TRC-20/721/1155 implementations
│   └── mocks/                 # test-only mocks — never deployed
├── test/
│   ├── unit/                  # one file per contract under test
│   └── integration/           # exercises a real TRE/testnet deployment
├── migrations/                 # TronBox migration/deploy scripts, numbered
├── scripts/                    # operational scripts (TronWeb-based)
├── audits/                     # audit reports, checked in as received
├── deployments/                 # deployed addresses per network, checked in (base58 + hex form)
└── AGENTS.md
```

Mocks live only under `test/` or `contracts/mocks/` and are never imported by production contracts.

---

## 5. Solidity style

Mostly identical to standard Solidity best practice — TRON's TVM executes Solidity bytecode with only the differences called out in §6, so the language-level rules below are the same discipline any Solidity project should follow:

- Pin the exact compiler version in `pragma solidity 0.8.x;` — no `^` or `>=` in production contracts. **Verify the exact version against TronBox/TRON's currently supported `solc` versions before picking one** — TRON does not always support the very latest Solidity release on day one.
- One primary contract per file; file name matches the contract name.
- Follow standard Solidity ordering: `type declarations → state variables → events → errors → modifiers → constructor → receive/fallback → external → public → internal → private`.
- Use custom errors over `require` string messages where the toolchain/TVM version in use supports them at acceptable gas/energy cost — confirm custom error support and cost behavior on TRON specifically rather than assuming Ethereum's cost model applies (see §6).
- Explicit visibility on every function and state variable; explicit `view`/`pure` mutability.
- `immutable` for values set once in the constructor and never changed; `constant` for compile-time literals.
- NatSpec on every external/public function, event, and custom error — this is the audit-readable spec of intent.
- Naming: contracts/interfaces/libraries `PascalCase` (interfaces prefixed `I`); functions/variables `camelCase`; constants `SCREAMING_SNAKE_CASE`; private/internal state variables prefixed `_`.
- No magic numbers. Name every non-obvious constant, and comment where a constant's value is chosen specifically because of a TRON-specific limit (see §6).

---

## 6. TRON-specific differences from Ethereum — read before assuming EVM parity

TRON's TVM is **EVM-compatible, not EVM-identical**. Code and mental models carried over uncritically from Ethereum are a real source of bugs here. Confirm current behavior against TRON's official developer documentation before relying on any of the following, since specifics evolve with TVM upgrades — but always check explicitly rather than assuming Ethereum's rules apply:

### Resource model: Energy and Bandwidth, not gas
- TRON transactions consume **Energy** (computation) and **Bandwidth** (transaction size), not ETH-denominated gas. Contract calls can fail with an out-of-energy condition analogous to out-of-gas — code that assumes a simple linear "gas price × gas used" cost model from Ethereum needs to be re-verified against TRON's energy pricing.
- Accounts can **stake TRX for Energy/Bandwidth** (via Stake 2.0) instead of paying per-transaction, or **delegate** resources to another account/contract. If the product relies on a contract or relayer having enough Energy to execute, document explicitly how that Energy is funded/staked/delegated — don't assume a wallet always has enough native balance to cover costs the way ETH gas works.
- Contract deployers can configure a **percentage of Energy consumed by callers to be paid by the contract's own staked resources** (`consume_user_resource_percent` at deployment). Set this value deliberately and document why; getting it wrong either starves callers of a working contract or silently drains the deployer's staked resources.

### Addresses
- TRON addresses are **Base58Check-encoded, prefixed `T`** in their user-facing form (e.g. `TXYZ...`), distinct from Ethereum's `0x`-prefixed hex addresses. Internally TRON addresses are 21-byte values (a `0x41` prefix + the 20-byte EVM-style address), which differs from Ethereum's raw 20 bytes.
- **Never assume address interop between TRON and Ethereum without explicit conversion.** A hex address from an Ethereum contract/tool is not directly usable as a TRON Base58 address without proper conversion via the client library (TronWeb) — validate and convert explicitly, and add tests around any code path that parses or displays an address.
- Be explicit in code and variable names about which representation is in use (`addressBase58`, `addressHex`) — silently mixing the two representations is a real, funds-affecting bug class.

### Precompiles and opcode support
- Not every EVM precompile or opcode is guaranteed identically available/priced on TVM. If the contract relies on a specific precompile (e.g. for signature recovery, hashing, or pairing operations), verify it's supported and priced as expected on TRON before assuming Ethereum semantics — check the current TRON documentation, since TVM opcode support has expanded over successive upgrades and isn't a fixed, one-time fact.
- Static analysis tools built primarily for Ethereum (Slither, Mythril) can produce **false positives or miss TRON-specific issues** since their rule sets assume EVM/Ethereum context. Treat their findings as a starting point, not a complete picture, for TRON-specific behavior — supplement with manual review of anything flagged as environment-dependent.

### Native token and standards
- TRX is the native coin (like ETH on Ethereum); **TRC-10** is a separate, native token standard issued directly on-chain without a smart contract (unlike ERC-20, which is always a contract). If the project interacts with TRC-10 tokens, that interaction happens via TRON's native token transfer mechanism, not via a standard token-contract ABI call — don't assume every token on TRON is a TRC-20 contract.
- **TRC-20** is TRON's ERC-20-equivalent contract standard and is the one most reused from Ethereum tooling; still verify the actual TRC-20 contract in use matches the standard interface exactly (some TRON token deployments have used non-standard variants), and apply the same "never trust a token to behave per spec" discipline as in any EVM project (safe-transfer wrappers, checked return values).

### Tooling and RPC
- TronWeb (or an equivalent TRON-aware client) is required for constructing, signing, and broadcasting transactions and for address conversion — a plain `ethers.js`/`web3.js` client talking to an Ethereum-style JSON-RPC endpoint is **not** a drop-in substitute for TRON's HTTP API surface (TronGrid/full node API), even though contract-level Solidity code may look familiar.
- Event/log parsing, transaction receipt shapes, and confirmation semantics differ from Ethereum's — verify assumptions against TronWeb's actual response shapes and TRON's block-confirmation/finality model rather than porting Ethereum-specific parsing code unchanged.

---

## 7. Security — non-negotiable rules

The core smart-contract security discipline is the same as any EVM-adjacent chain; the items below are stated in full because they matter regardless of chain, with TRON-specific notes inline where relevant.

### Checks-Effects-Interactions and reentrancy
- Update all state (effects) **before** making an external call (interaction). Reentrancy guards (`ReentrancyGuard`-equivalent) are still required on any function making an external call and later touching shared state — TVM's execution model does not remove this risk.
- Watch for cross-function and read-only reentrancy, not just same-function reentrancy.

### Arithmetic
- Solidity ≥0.8's built-in overflow/underflow reverts apply on TVM as well. Don't wrap arithmetic in `unchecked {}` without proving it's safe, and say so in a comment when you do.
- Watch precision loss and rounding direction in division; round in the protocol's favor and state explicitly which direction was chosen.

### External calls and trust boundaries
- Treat every external contract call as potentially malicious, including calls to TRC-20 tokens the project doesn't control.
- Never assume a TRC-20 token behaves exactly per spec; use checked-return-value transfer patterns and consider non-standard token behavior explicitly.
- Avoid `tx.origin` for authorization — use `msg.sender`.
- Avoid `delegatecall` to untrusted or user-supplied addresses; any use needs an explicit justification comment and a storage-layout-compatibility test.

### Access control
- Every privileged function (mint, pause, upgrade, withdraw, set-parameter) has an explicit modifier — never rely on an off-chain assumption about who calls it.
- Prefer a well-reviewed `AccessControl`/two-step-ownership pattern over a hand-rolled role system, ported carefully for TVM compatibility (confirm any OpenZeppelin-derived code compiles and behaves as expected under the project's pinned `solc`/TVM version, per §6).

### Upgradeability (if applicable)
- Never reorder or retype existing storage variables in an upgrade — only append. Diff the storage layout before every upgrade and include the diff in the PR.
- Constructors on upgradeable (logic) contracts must disable initializers; initialization logic goes in a guarded `initialize()` function.

### Randomness, timing, and front-running
- Never use block-derived values (timestamp, block hash) as a randomness source for anything of value — use a verifiable randomness source.
- TRON transactions are visible before confirmation like any public mempool/block-production system; anything price- or order-sensitive (swaps, auctions, liquidations) needs slippage/deadline protection, stated explicitly in the design.

### Denial of service
- Never loop over an unbounded, user-controlled or externally-growable collection in a function that must succeed. Use pull-based patterns instead of push-based patterns a single bad actor can grief.

### General
- No hardcoded addresses in production contract logic other than well-known, genuinely immutable constants, documented as such — and stored/compared in a consistent address representation (see §6).
- Run static analysis as part of the standard gate, with the TRON caveat from §6 in mind — don't treat it as optional.
- **Any contract handling real value must go through an external audit, ideally one with specific TRON/TVM experience, before mainnet deployment.** An EVM-focused audit alone may miss TRON-specific issues (energy/resource exhaustion vectors, address-handling bugs, TRC-10 interaction bugs). No amount of internal review or test coverage substitutes for this.

---

## 8. Testing

- Unit test every external and public function: happy path, every documented revert condition, and boundary values.
- Fuzz test functions taking numeric or array inputs, bounding inputs meaningfully rather than discarding most runs.
- Invariant/stateful tests for core protocol properties that must hold regardless of call sequence.
- **Test against a real TRE (local TRON private chain) or testnet (Shasta/Nile) deployment, not solely mocks**, for anything involving Energy/Bandwidth consumption, address conversion, or TRC-20/TRC-10 interaction — these are exactly the areas where TVM behavior diverges from an idealized EVM mock.
- Explicitly test Energy-exhaustion scenarios for functions expected to run under tight resource constraints, and test address-representation handling (Base58 vs. hex) wherever an address crosses a boundary (input parsing, event emission, external call).
- Every bug fix ships with a regression test that reproduces the bug and fails without the fix.

---

## 9. Deployment

- Deployment/migration scripts live in `migrations/` (TronBox convention) and are version-controlled like any other code — no manual, undocumented deploys.
- Record every deployed address, per network, in `deployments/` in **both** Base58 and hex form, with deployer address, block/transaction reference, and constructor args.
- Verify contracts on TronScan (or the project's chosen explorer) as part of the deploy step, so deployed bytecode is publicly auditable against source.
- Use a hardware wallet or a secrets-managed signer for mainnet deploys — never a plaintext private key in an env var beyond local/testnet use.
- Testnet (Shasta/Nile) deployment and a manual review pass are required before any mainnet deployment, regardless of test coverage.
- Set `consume_user_resource_percent` and any other deploy-time resource parameters deliberately, and record the chosen values and rationale in the deployment record.

---

## 10. Dependencies

- Prefer well-audited, widely used contract libraries where TRON/TVM compatibility has been verified (e.g. OpenZeppelin Contracts, confirming compatibility with the project's pinned `solc` version and TVM behavior) over reimplementing standard primitives.
- Pin exact dependency versions — never a floating range for anything shipping to production.
- Before adding a new dependency, check its audit history and TRON-specific compatibility (some Ethereum-oriented tooling has known TVM caveats); note the justification in the PR.
- Vendor or pin dependency source at a fixed commit/exact version so the build is reproducible.

---

## 11. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `audit/short-description`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat(token): add TRC-20 pausable transfer`.
- PR description states: what changed, security implications (explicitly), how it was tested (unit/fuzz/invariant/TRE-or-testnet), and Energy/Bandwidth impact for hot-path functions.
- Any change touching access control, arithmetic on value, external calls, address handling, or upgrade logic needs at least one reviewer explicitly signing off on the security implications.
- Never force-push to `main`. Never amend or rebase a commit someone else may have pulled.
- Do not commit: `.env`, private keys, build artifacts (`build/`, `node_modules/`).

---

## 12. Rules for agents

**Do**
- Read the surrounding contract and its tests before editing; match existing patterns rather than introducing a second convention.
- Treat §6 (TRON-specific differences) and §7 (Security) as binding, not advisory — apply them even when not explicitly reminded to.
- Run the full verify gate (compile, lint, test, static analysis) and report actual output before claiming success.
- Explicitly verify, rather than assume, any claim about TRON's current opcode/precompile support, Energy pricing, or resource model against up-to-date TRON documentation before relying on it in code or in an explanation — these specifics change across TVM upgrades.
- Flag explicitly any change that touches: fund transfers, access control, upgrade logic, external calls, address-representation handling, or Energy/Bandwidth-sensitive logic — even if asked to make an unrelated change nearby.
- Ask before: changing storage layout on an upgradeable contract, modifying access-control roles, changing how funds are transferred or accounted for, changing deploy-time resource parameters, or adding a new external dependency.

**Don't**
- Don't assume Ethereum tooling, gas-cost intuition, or address formats carry over unchanged — verify against §6 first.
- Don't write or modify a contract touching value transfer, access control, or upgrade logic without also writing/updating tests for it in the same change.
- Don't silence a static-analysis finding, compiler warning, or failing test without understanding and stating why it's a false positive (remembering that EVM-focused static analysis tools can misfire on TVM-specific patterns).
- Don't treat testnet/TRE code as exempt from security practices — bad habits from test code end up in mainnet code.
- Don't suggest or generate a private key, mnemonic, or real credential, even a placeholder-looking one, for anything beyond an obviously fake example.
- Don't mark work complete with a failing test, an un-investigated static-analysis finding, or a missing NatSpec on a new external function.

---

## 13. Definition of done

1. `tronbox compile` succeeds with no warnings the project doesn't already accept.
2. Full test suite passes, including fuzz/invariant suites where they exist, run against a real TRE or testnet for anything Energy/address/TRC-token-sensitive.
3. New external/public functions have unit tests covering the happy path and every revert condition, plus fuzz tests for numeric inputs.
4. Static analysis has been run and findings addressed or explicitly justified, with TRON-specific false positives noted as such.
5. Format/lint checks pass.
6. NatSpec is present and accurate on every new/changed external and public function, event, and custom error.
7. The PR description states the security implications of the change, Energy/Bandwidth impact, and how it was tested.
8. For anything deploying to mainnet: an external audit (ideally TRON-experienced) has been completed and its findings resolved.
