# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.
> **Domain note:** This is a smart contract repository handling real (or eventually real) value. Mistakes here are frequently irreversible and public. Treat every rule below as load-bearing, not stylistic.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Solidity version | `<0.8.24>` (pinned exactly, no `^`) |
| Framework | `<Foundry / Hardhat>` |
| Node.js (if Hardhat/tooling) | `>=20.11` — see `.nvmrc` |
| Package manager | `pnpm` |
| Target chain(s) | `<Ethereum mainnet / L2 name(s)>` |
| Upgradeability | `<none — immutable / UUPS / Transparent Proxy>` |
| Libraries | `<OpenZeppelin Contracts vX.Y, Solmate, etc.>` |
| Audit status | `<unaudited / audited by X on <date> / in progress>` |

Replace the `<...>` placeholders before relying on this file. **If `Audit status` says "unaudited," no contract from this repo is deployed to mainnet with real funds at stake, ever, regardless of how confident anyone is in the code.**

---

## 2. Setup

### Foundry
```bash
foundryup
forge install
cp .env.example .env
forge build
```

### Hardhat
```bash
nvm use
pnpm install --frozen-lockfile
cp .env.example .env
pnpm compile
```

Never commit `.env`, a private key, a mnemonic, or an Etherscan/RPC API key. If a secret is accidentally committed, it must be treated as **compromised and rotated immediately** — removing it from a later commit does not undo exposure in git history.

---

## 3. Commands

| Task | Foundry | Hardhat |
| --- | --- | --- |
| Build | `forge build` | `pnpm compile` |
| Test | `forge test` | `pnpm test` |
| Test, verbose traces | `forge test -vvvv` | `pnpm test --verbose` |
| Single test | `forge test --match-test testTransferReverts` | `pnpm test --grep "transfer reverts"` |
| Fuzz/invariant only | `forge test --match-contract Invariant` | `pnpm test:invariant` |
| Coverage | `forge coverage` | `pnpm coverage` |
| Gas report | `forge test --gas-report` | `pnpm test:gas` |
| Static analysis | `slither .` | `slither .` |
| Format | `forge fmt` | `pnpm format` |
| Format check (CI) | `forge fmt --check` | `pnpm format:check` |
| Local node | `anvil` | `pnpm hardhat node` |
| Deploy (script) | `forge script script/Deploy.s.sol --rpc-url $RPC_URL --broadcast --verify` | `pnpm hardhat run scripts/deploy.ts --network <net>` |
| Full pre-push gate | `pnpm verify` (wraps: build → test → coverage → slither → fmt check) |

**`pnpm verify` (or the equivalent full gate) must pass before any code is considered done — no exceptions for "just a small change."**

---

## 4. Directory structure

```
.
├── src/ (or contracts/)
│   ├── interfaces/           # I-prefixed interfaces, no implementation
│   ├── libraries/            # internal libraries, pure logic where possible
│   ├── core/                 # primary protocol contracts
│   ├── periphery/            # non-critical-path helper contracts
│   └── mocks/                # test-only mocks — never deployed, excluded from prod build if possible
├── test/
│   ├── unit/                 # one file per contract under test
│   ├── invariant/            # stateful fuzzing / invariant suites
│   ├── fork/                 # mainnet/testnet fork tests against real deployed deps
│   └── helpers/              # shared test utilities, fixtures, harnesses
├── script/ (or scripts/)      # deployment and operational scripts
├── audits/                    # audit reports, checked in as received
├── deployments/                # deployed addresses per network, checked in
└── AGENTS.md
```

Mocks live only under `test/` or `src/mocks/` and are never imported by production contracts. If a mock's logic starts looking load-bearing, that's a sign the interface it mocks needs a real test double instead.

---

## 5. Solidity style

- Pin the exact compiler version in `pragma solidity 0.8.24;` — no `^` or `>=` in production contracts. Floating pragmas are acceptable only in interfaces meant for external consumption.
- One primary contract per file; file name matches the contract name.
- Follow the [Solidity Style Guide](https://docs.soliditylang.org/en/latest/style-guide.html) ordering: `type declarations → state variables → events → errors → modifiers → constructor → receive/fallback → external → public → internal → private`.
- **Use custom errors, not `require` with string messages** (`error InsufficientBalance(uint256 available, uint256 required);`) — cheaper and more informative.
- Explicit visibility on every function and state variable. No implicit `public` defaults left unstated.
- Explicit function mutability: mark `view`/`pure` wherever accurate; never mark something `view` that isn't.
- Use `immutable` for values set once in the constructor and never changed; `constant` for compile-time literals. Don't leave something mutable that never actually changes.
- NatSpec (`/// @notice`, `/// @param`, `/// @return`) on every external and public function, and on every custom error and event. This is not optional documentation — it is the audit-readable spec of intent.
- Naming: contracts/interfaces/libraries `PascalCase` (interfaces prefixed `I`, e.g. `IERC20`); functions/variables `camelCase`; constants `SCREAMING_SNAKE_CASE`; private/internal state variables prefixed with `_` (`_totalSupply`).
- No magic numbers. Name every non-obvious constant.
- Prefer composition and internal libraries over deep inheritance chains. If a contract inherits more than ~3 levels deep, question whether that's necessary.
- Emit an event for every state-changing external function — off-chain systems and indexers depend on this.

---

## 6. Security — non-negotiable rules

Read this section fully before touching any contract that moves value or controls access.

### Checks-Effects-Interactions
- Update all state (effects) **before** making an external call (interaction). Never call out to an untrusted address and then update state afterward.
- Any function that sends ETH or tokens, or calls an arbitrary/external address, needs an explicit reentrancy analysis in its NatSpec or a comment — even if `nonReentrant` is applied. Don't rely on the modifier alone as an excuse to skip the checks-effects-interactions pattern.

### Reentrancy
- Use OpenZeppelin's `ReentrancyGuard` (or an equivalent audited implementation) on any function that makes an external call and later touches shared state — don't hand-roll a reentrancy lock.
- Watch for **cross-function** and **read-only** reentrancy, not just same-function reentrancy: a view function returning a manipulable value mid-callback is a real vector.

### Arithmetic
- Solidity ≥0.8 reverts on overflow/underflow by default — do not wrap arithmetic in `unchecked {}` unless you've proven it cannot overflow, and say so in a comment when you do. An unjustified `unchecked` block is treated as a bug in review.
- Watch for **precision loss and rounding direction** in division — round in the protocol's favor (e.g. round down when paying out, round up when charging), and say explicitly which direction was chosen and why.

### External calls and trust boundaries
- Treat every external contract call as **potentially malicious**, including calls to tokens the project itself doesn't control (a "standard" ERC-20 can still have hooks, fee-on-transfer behavior, or reentrant callbacks).
- Never assume an ERC-20 behaves per spec. Check return values (`SafeERC20`), account for tokens that don't return a bool, and consider fee-on-transfer and rebasing tokens explicitly if the contract accepts arbitrary tokens.
- Avoid `tx.origin` for authorization — always use `msg.sender`.
- Avoid `delegatecall` to untrusted or user-supplied addresses. Any `delegatecall` usage needs an explicit justification comment and a test proving storage layout compatibility.
- Low-level `call`/`send`/`transfer`: prefer `call` with an explicit gas stipend consideration and always check the return value; never use `transfer`/`send` as the sole ETH-sending mechanism (the fixed 2300 gas stipend breaks with many wallets/contracts).

### Access control
- Every privileged function (mint, pause, upgrade, withdraw, set-parameter) has an explicit modifier (`onlyOwner`, `onlyRole(...)`) — never rely on a comment saying "only called by admin off-chain."
- Prefer OpenZeppelin's `AccessControl`/`Ownable2Step` over hand-rolled role systems. Use the two-step ownership transfer pattern — a single-step `transferOwnership` to a typo'd address is an unrecoverable loss of control.
- Document, in NatSpec, exactly who can call each privileged function and what the blast radius of misuse is.

### Upgradeability (if applicable)
- If contracts are upgradeable: never change the order or type of existing storage variables in an upgrade — only append new ones. Run a storage-layout diff (`forge inspect <Contract> storage-layout` or the Hardhat/OZ equivalent) before every upgrade and include it in the PR.
- Constructors on upgradeable (logic) contracts must call `_disableInitializers()`; initialization logic goes in an `initialize()` function protected by `initializer`.
- Every upgrade needs a plan for what happens if it fails partway (timelock, multisig review, rollback capability).

### Randomness, timing, and MEV
- Never use `block.timestamp`, `blockhash`, or block-derived values as a source of randomness for anything of value — use a verifiable randomness source (e.g. Chainlink VRF).
- Assume every transaction is visible in the mempool before inclusion. Anything vulnerable to front-running (price-dependent swaps, auctions, liquidations) needs slippage/deadline protection or a commit-reveal scheme, and this needs to be called out explicitly in the design, not left implicit.

### Denial of service
- Never loop over an unbounded, user-controlled or externally-growable array/mapping in a function that must succeed (e.g. distributing rewards to "all holders"). Use pull-based patterns (users withdraw their own share) instead of push-based patterns that can be griefed by one failing recipient.
- Consider gas griefing: a malicious external call target can consume all forwarded gas or always revert — don't let one bad actor block a function meant to serve many users.

### General
- No `selfdestruct` unless there is a specific, reviewed reason — its semantics changed post-Cancun and it's rarely the right tool.
- No hardcoded addresses in production contract logic other than well-known, genuinely immutable protocol constants (documented with a comment on why they're safe to hardcode).
- Run **Slither** (and, for anything security-critical, consider Mythril/Echidna/Foundry invariant fuzzing) as part of the standard gate — don't treat static analysis as optional or a "nice to have later."
- **Any contract handling real value must go through an external audit before mainnet deployment.** No amount of internal review or test coverage substitutes for this.

---

## 7. Testing

- **Foundry** (preferred for unit/fuzz/invariant) or **Hardhat + Chai/Waffle** — follow whichever the project already uses; don't introduce a second framework.
- Unit test every external and public function: the happy path, every documented revert condition, and boundary values (zero, max uint, empty array).
- **Fuzz test** functions that take numeric or array inputs — Foundry's `forge test` fuzzes automatically for parameters left unbound; write explicit `testFuzz_` cases and use `vm.assume` to bound inputs meaningfully rather than discarding most fuzz runs.
- **Invariant/stateful tests** for core protocol properties that must hold no matter what sequence of actions occurs (e.g. "total supply always equals sum of balances," "protocol never insolvent"). These matter more than line coverage for catching real bugs.
- **Fork tests** against real deployed dependencies (a real token, a real oracle, a real DEX pool) before assuming an integration works — mocks alone are not sufficient for anything touching an external protocol.
- Test gas costs for functions users will call frequently; a regression in gas cost is a real regression, not a style nit.
- Line/branch coverage is a floor, not a target — a fully covered function with no assertion on the revert reason or emitted event data is not actually tested.
- Every bug fix ships with a regression test that reproduces the bug and fails without the fix.

---

## 8. Deployment

- Deployment scripts live in `script/`/`scripts/` and are version-controlled like any other code — no manual, undocumented deploys from a local console.
- Every deployed address, per network, is recorded in `deployments/` immediately after deployment, with the deployer address, block number, and constructor args.
- Verify contracts on the relevant block explorer as part of the deploy step (`--verify` in Foundry, the Hardhat verify plugin), so the deployed bytecode is publicly auditable against source.
- Use a hardware wallet, multisig (e.g. Safe), or a secrets-managed signer for mainnet deploys — never a plaintext private key in an env var for anything beyond a local testnet.
- Testnet (or a mainnet fork) deployment and a manual review pass are required before any mainnet deployment, regardless of test coverage.
- Time-sensitive or privileged deploy steps (setting the final owner, renouncing a deployer role, unpausing) get their own explicit, reviewed script step — not folded silently into contract construction.

---

## 9. Dependencies

- Prefer well-audited, widely used libraries (OpenZeppelin Contracts, Solmate/Solady) over reimplementing standard primitives (ERC-20/721/1155, access control, reentrancy guards, safe math wrappers).
- Pin exact dependency versions (`forge install <lib>@<tag>` at a specific tag/commit, or an exact `package.json` version) — never a floating range for anything that ships to production.
- Before adding any new external dependency, check its audit history and how widely it's used in production; note the justification in the PR.
- Vendor or pin dependency source (git submodule at a fixed commit, or npm exact version) so the build is reproducible and can't silently change underneath the project.

---

## 10. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `audit/short-description`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat(vault): add withdrawal fee`.
- PR description states: what changed, the security implications (explicitly — "no change to trust assumptions" is a valid answer, but state it), how it was tested (unit/fuzz/invariant/fork), and gas impact for hot-path functions.
- Any change touching access control, arithmetic on value, external calls, or upgrade logic needs at least one reviewer explicitly signing off on the security implications, not just code style.
- Never force-push to `main`. Never amend or rebase a commit someone else may have pulled.
- Do not commit: `.env`, private keys, `cache/`, `out/`/`artifacts/`, `node_modules/`.

---

## 11. Rules for agents

**Do**
- Read the surrounding contract and its tests before editing; match existing patterns (custom errors vs. require strings, existing access-control approach) rather than introducing a second convention.
- Treat §6 (Security) as binding, not advisory — apply it even when not explicitly reminded to.
- Run the full verify gate (build, test, coverage, slither, fmt check) and report actual output before claiming success.
- Flag explicitly, in the response, any change that touches: fund transfers, access control, upgrade logic, external calls, or arithmetic on user-supplied values — even if asked to make an unrelated change nearby.
- Ask before: changing storage layout on an upgradeable contract, modifying access-control roles, changing how funds are transferred or accounted for, or adding a new external dependency.

**Don't**
- Don't write or modify a contract touching value transfer, access control, or upgrade logic without also writing/updating tests for it in the same change.
- Don't silence a Slither finding, a compiler warning, or a failing invariant test without understanding and stating why it's a false positive.
- Don't invent a "this is fine because it's just a testnet" exception for security practices — bad habits from testnet code end up in mainnet code.
- Don't suggest or generate a private key, mnemonic, or real credential, even a placeholder-looking one, for anything beyond an obviously fake example (`0x000...dead`-style).
- Don't mark work complete with a failing test, an un-investigated Slither finding, or a missing NatSpec on a new external function.

---

## 12. Definition of done

1. `forge build` / `pnpm compile` succeeds with no warnings the project doesn't already accept.
2. Full test suite (`forge test` / `pnpm test`) passes, including fuzz/invariant suites where they exist.
3. New external/public functions have unit tests covering the happy path and every revert condition, plus fuzz tests for numeric inputs.
4. Slither (and any other configured static analysis) has been run and findings addressed or explicitly justified.
5. `forge fmt --check` / `pnpm format:check` passes.
6. NatSpec is present and accurate on every new/changed external and public function, event, and custom error.
7. The PR description states the security implications of the change and how it was tested.
8. For anything deploying to mainnet: an external audit has been completed and its findings resolved.
