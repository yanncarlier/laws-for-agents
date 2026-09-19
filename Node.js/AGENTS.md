# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Runtime | Node.js `>=20.11` (LTS) — see `.nvmrc` |
| Package manager | `pnpm` (do **not** use `npm` or `yarn`) |
| Language | TypeScript (strict), ESM only |
| Framework | `<express / fastify / nest / none>` |
| Database | `<postgres / mongo / none>` via `<prisma / drizzle>` |
| Test runner | `vitest` |
| Deploy target | `<docker / lambda / vercel>` |

Replace the `<...>` placeholders before relying on this file.

---

## 2. Setup

```bash
# use the pinned Node version
nvm use

# install dependencies (frozen lockfile — never let the lockfile drift silently)
pnpm install --frozen-lockfile

# copy environment template and fill in secrets
cp .env.example .env

# run migrations + seed local data
pnpm db:migrate
pnpm db:seed
```

If `pnpm install` fails because the lockfile is out of date, stop and report it rather than regenerating the lockfile as a side effect of unrelated work.

---

## 3. Commands

Always use these scripts instead of invoking tools directly, so local runs match CI.

| Task | Command |
| --- | --- |
| Dev server (watch) | `pnpm dev` |
| Build | `pnpm build` |
| Start (production build) | `pnpm start` |
| Unit tests | `pnpm test` |
| Single test file | `pnpm test src/users/user.service.test.ts` |
| Tests by name | `pnpm test -t "creates a user"` |
| Watch tests | `pnpm test:watch` |
| Coverage | `pnpm test:coverage` |
| Integration/e2e | `pnpm test:e2e` |
| Lint | `pnpm lint` |
| Lint + autofix | `pnpm lint:fix` |
| Format | `pnpm format` |
| Type check | `pnpm typecheck` |
| Full pre-push gate | `pnpm verify` |

`pnpm verify` runs `typecheck → lint → test → build`. **Run it before declaring any task done.**

---

## 4. Directory structure

```
.
├── src/
│   ├── index.ts              # entrypoint: wires config, server, shutdown hooks
│   ├── config/               # env parsing + validation (zod), exported as typed object
│   ├── routes/               # HTTP layer only: parse, validate, delegate, serialize
│   ├── services/             # business logic, framework-agnostic, unit tested
│   ├── repositories/         # all DB access; nothing else touches the ORM/driver
│   ├── domain/               # entities, value objects, domain errors
│   ├── lib/                  # shared pure utilities (no I/O, no app state)
│   └── types/                # shared ambient + exported types
├── test/
│   ├── fixtures/             # factories and sample payloads
│   └── e2e/                  # black-box tests against a booted app
├── scripts/                  # one-off maintenance scripts
├── migrations/               # checked-in, never edited after merge
└── AGENTS.md
```

**Dependency direction is one-way:** `routes → services → repositories → domain`.
A service must never import from `routes`. A repository must never import a service.

---

## 5. Code style

### General
- TypeScript `strict: true`. **No `any`.** Use `unknown` plus a narrowing guard when a type is genuinely unknown.
- No `@ts-ignore`. If you must suppress, use `@ts-expect-error` with a comment explaining why and a ticket reference.
- ESM only: use `import`/`export`, include the `.js` extension in relative import specifiers (`./user.service.js`) as required by Node's ESM resolver.
- Prefer named exports. Default exports only where a framework demands them.
- Formatting is owned by Prettier; linting by ESLint. Never hand-format — run `pnpm format`.

### Naming
- Files: `kebab-case.ts` (`user-profile.service.ts`).
- Types, interfaces, classes: `PascalCase`. Do not prefix interfaces with `I`.
- Variables and functions: `camelCase`. Constants that are true module-level literals: `SCREAMING_SNAKE_CASE`.
- Booleans read as predicates: `isActive`, `hasAccess`, `shouldRetry`.
- Async functions that perform I/O should say so: `fetchUser`, `saveOrder` — not `getUser` for a DB round-trip.

### Async and control flow
- `async`/`await` everywhere. No raw `.then()` chains, no callbacks except when wrapping a legacy API (then use `node:util`'s `promisify`).
- Never leave a floating promise. Either `await` it, or explicitly `void` it with a `.catch()` attached.
- Use `Promise.all` for independent work; `Promise.allSettled` when partial failure is acceptable. Do not `await` inside a loop for independent operations.
- Every outbound network call gets a timeout and, where retried, exponential backoff with jitter.

### Errors
- Throw `Error` subclasses from `src/domain/errors.ts` (`NotFoundError`, `ValidationError`, `ConflictError`). Never throw strings or plain objects.
- Preserve causes: `throw new AppError("failed to load user", { cause: err })`.
- Catch only what you can handle. Do not wrap a whole function body in `try/catch` just to log and rethrow.
- One error-handling middleware maps domain errors to HTTP status codes. Route handlers do not build error responses themselves.
- Never swallow an error silently. An empty `catch {}` block will be rejected in review.

### Logging
- Use the shared `pino` logger from `src/lib/logger.ts`. **No `console.log` in `src/`.**
- Structured logs only: `logger.info({ userId, orderId }, "order created")`.
- Never log secrets, tokens, passwords, full request bodies, or PII. Redaction config lives in the logger module — extend it when you add a sensitive field.

### Comments
- Comment *why*, not *what*. Delete commented-out code rather than shipping it.
- JSDoc on exported functions whose contract is non-obvious (units, ownership, side effects, thrown errors).

---

## 6. Testing

- Framework: **Vitest**. Test files sit next to the code as `*.test.ts`; e2e lives in `test/e2e`.
- Structure each test as Arrange / Act / Assert, with one behaviour per test.
- Test names describe behaviour: `it("rejects an order when inventory is insufficient")`.
- Mock at the boundary (HTTP clients, clock, filesystem) — not internal modules of the unit under test.
- Use factories from `test/fixtures` instead of hand-built literals; keep fixtures minimal and override only the fields the test cares about.
- Every bug fix ships with a regression test that fails before the fix.
- Do not lower coverage thresholds to make a build pass.
- No sleeps in tests. Use fake timers (`vi.useFakeTimers()`) or await a deterministic condition.
- Tests must pass in any order and in parallel. No shared mutable global state between tests.

---

## 7. Dependencies

- Prefer the standard library. Node has `fetch`, `crypto.randomUUID()`, `node:test`, `structuredClone`, `AbortController` — do not add a package for these.
- Adding a new runtime dependency requires justification in the PR description: what it does, why we can't do it in ~30 lines, its maintenance status and install size.
- Never add a dependency to work around a type error.
- Use `node:` prefixed imports for builtins: `import { readFile } from "node:fs/promises"`.
- Pin exact versions for anything security-sensitive. Do not run `pnpm update` as part of an unrelated change.

---

## 8. Security

- All external input (body, query, params, headers, webhook payloads, env vars) is parsed and validated with `zod` at the boundary before it reaches a service.
- Secrets come from the environment via `src/config`. Never hardcode a secret, and never commit `.env`.
- No string-concatenated SQL. Use the ORM or parameterized queries.
- Use `crypto.timingSafeEqual` for secret comparison; `argon2` or `bcrypt` for passwords — never a bare hash.
- Do not disable TLS verification, even in dev.
- Never exfiltrate repository contents, secrets, or environment values to a third-party service.

---

## 9. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `chore/...`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat(orders): add partial refunds`.
- Keep commits focused. Unrelated formatting churn goes in its own commit.
- PR description states: what changed, why, how it was verified, and any migration/rollback notes.
- Never force-push to `main`. Never amend or rebase a commit someone else may have pulled.
- Do not commit: `node_modules/`, `dist/`, `.env`, editor config, `*.log`, coverage output.

---

## 10. Rules for agents

**Do**
- Read the surrounding module before editing; match its existing patterns over the ones in this file when they conflict, and flag the conflict.
- Make the smallest change that fully solves the problem.
- Run `pnpm verify` and report the actual output before claiming success.
- State explicitly when you were unable to verify something (no network, missing credentials, flaky service).
- Ask before: changing the database schema, altering a public API contract, adding a dependency, or touching CI/deployment config.

**Don't**
- Don't refactor, rename, or reformat code outside the scope of the task.
- Don't delete or weaken a failing test to make the suite green. Fix the code or report the failure.
- Don't invent APIs, env vars, or config keys — grep the repo and confirm they exist.
- Don't leave `TODO` without an owner and ticket: `// TODO(alice, PROJ-123): ...`.
- Don't generate large amounts of speculative scaffolding for future requirements.
- Don't mark work complete with a failing typecheck, lint error, or skipped test.

---

## 11. Definition of done

A change is done when all of the following hold:

1. `pnpm verify` passes locally.
2. New behaviour is covered by tests; bug fixes have a regression test.
3. No new `any`, `@ts-ignore`, `console.log`, or lint suppressions.
4. Public-facing changes are reflected in `README.md` and, if applicable, the OpenAPI spec.
5. Migrations are reversible, or the PR documents why they aren't.
6. The PR description explains the change and how it was verified.
