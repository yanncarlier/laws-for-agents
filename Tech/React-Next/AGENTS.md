# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.
> **Stack assumed:** client-rendered React (e.g. Vite). See §12 if this is actually a Next.js project.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Runtime | Node.js `>=20.11` (LTS) — see `.nvmrc` |
| Package manager | `pnpm` (do **not** use `npm` or `yarn`) |
| Language | TypeScript (strict) |
| Build tool | Vite |
| React version | 18+ (function components + hooks only) |
| Styling | `<tailwind / css-modules / styled-components>` |
| State management | `<react-query + context / zustand / redux-toolkit>` |
| Routing | `<react-router / tanstack-router>` |
| Test runner | Vitest + React Testing Library |
| E2E | Playwright |

Replace the `<...>` placeholders before relying on this file.

---

## 2. Setup

```bash
nvm use
pnpm install --frozen-lockfile
cp .env.example .env.local
pnpm dev
```

If `pnpm install` fails because the lockfile is out of date, stop and report it rather than regenerating the lockfile as a side effect of unrelated work.

---

## 3. Commands

| Task | Command |
| --- | --- |
| Dev server | `pnpm dev` |
| Build | `pnpm build` |
| Preview production build | `pnpm preview` |
| Unit / component tests | `pnpm test` |
| Single test file | `pnpm test src/components/Button/Button.test.tsx` |
| Tests by name | `pnpm test -t "shows error state"` |
| Watch tests | `pnpm test:watch` |
| Coverage | `pnpm test:coverage` |
| E2E | `pnpm test:e2e` |
| Lint | `pnpm lint` |
| Lint + autofix | `pnpm lint:fix` |
| Format | `pnpm format` |
| Type check | `pnpm typecheck` |
| Storybook (if present) | `pnpm storybook` |
| Full pre-push gate | `pnpm verify` |

`pnpm verify` runs `typecheck → lint → test → build`. **Run it before declaring any task done.**

---

## 4. Directory structure

```
.
├── src/
│   ├── main.tsx                 # entrypoint: renders <App/>, mounts providers
│   ├── App.tsx                  # top-level router/layout composition
│   ├── components/              # shared, reusable, presentation-focused
│   │   └── Button/
│   │       ├── Button.tsx
│   │       ├── Button.test.tsx
│   │       ├── Button.stories.tsx
│   │       └── index.ts
│   ├── features/                # feature-sliced: UI + hooks + api for one domain
│   │   └── orders/
│   │       ├── components/
│   │       ├── hooks/
│   │       ├── api.ts
│   │       ├── types.ts
│   │       └── index.ts
│   ├── hooks/                   # cross-feature shared hooks
│   ├── lib/                     # framework-agnostic utilities, no React imports
│   ├── api/                     # HTTP client setup, request/response schemas
│   ├── routes/ (or pages/)      # route-level components, wired to the router
│   ├── styles/                  # global styles, design tokens
│   └── types/                   # shared ambient + exported types
├── test/
│   ├── setup.ts                 # RTL/jsdom setup, custom matchers
│   └── e2e/
├── public/
└── AGENTS.md
```

**Import direction:** `routes → features → components/hooks/lib`. A component in `components/` must never import from `features/`. `lib/` never imports React.

Co-locate a component's test and story with the component. A file with meaningful logic and no matching test file should raise a flag in review, not slip through.

---

## 5. Component conventions

- **Function components only.** No class components, no `React.FC` type (it adds implicit `children` and provides no real benefit — just type props directly).
- One component per file; file name matches the component name (`OrderSummary.tsx` exports `OrderSummary`).
- Props: define an explicit `interface Props { ... }` (or `OrderSummaryProps` when exported for reuse). Destructure props in the function signature, not inside the body.
- Keep components small and single-purpose. If a component's JSX needs more than ~3 levels of conditional nesting, extract a subcomponent.
- **Presentational vs. container split:** components in `components/` receive data and callbacks via props and contain no data fetching. Data fetching and business logic live in hooks (`useOrders`, `useOrderActions`) called from `features/` or route components.
- Default export only for route-level components required to by the router; everything else uses named exports.
- Co-locate a component's styles, test, and story files; do not scatter a single component's concerns across distant directories.

### Hooks
- Prefix custom hooks with `use` and follow the Rules of Hooks — no conditional or looped hook calls.
- One hook = one responsibility. A hook that both fetches data and manages form state should be split.
- Extract a custom hook once a piece of `useEffect`/`useState` logic is reused, or once it's complex enough to deserve its own unit test — don't extract prematurely for a single trivial `useState`.
- Return a stable, explicit shape from custom hooks (an object with named keys beats a positional array, once you have more than two values).

### State
- Local UI state (`isOpen`, `activeTab`) → `useState` in the component that owns it.
- Cross-component/shared state → the project's chosen state library (React Query for server state, `<zustand/redux-toolkit>` for client state). **Do not fetch server data with `useEffect` + `fetch`** — use the data-fetching library consistently.
- Lift state only as high as the nearest common consumer, no higher. Prefer composition (passing components as `children`/props) over lifting state to avoid prop drilling.
- Derive, don't duplicate: if a value can be computed from existing state/props, compute it during render (or memoize) — don't store it as separate state that can drift out of sync.

### Effects
- `useEffect` is for synchronizing with an external system (subscriptions, DOM APIs, non-React widgets) — not for computing derived state and not for handling user events.
- Always return a cleanup function for subscriptions, timers, and listeners.
- Include the full, honest dependency array. Do not silence `react-hooks/exhaustive-deps`; if a dependency is intentionally excluded, that's a sign the effect is doing more than one thing.

### Performance
- Don't reach for `useMemo`/`useCallback`/`React.memo` by default. Add them when a profiler shows a real cost, or when passing a callback to a memoized child, or memoizing an expensive computation. Premature memoization adds noise for no benefit.
- Provide a stable `key` (a real id) for list items — never the array index, except for static lists that never reorder.
- Code-split route-level components with `React.lazy` + `Suspense`.

---

## 6. TypeScript & style

- `strict: true`. **No `any`.** Use `unknown` with narrowing when the type is genuinely unknown.
- No `@ts-ignore`. Use `@ts-expect-error` with a comment when a suppression is unavoidable.
- Type all props, hook returns, and API response shapes explicitly. Infer local variables where TypeScript already gets it right — don't annotate everything.
- Naming: components/types `PascalCase`; variables, functions, hooks `camelCase`; files `PascalCase.tsx` for components, `camelCase.ts` for everything else.
- Booleans read as predicates: `isLoading`, `hasError`, `canSubmit`.
- Formatting is owned by Prettier; linting by ESLint (`eslint-plugin-react`, `eslint-plugin-react-hooks`, `eslint-plugin-jsx-a11y`). Never hand-format — run `pnpm format`.
- Comment *why*, not *what*. Delete commented-out code rather than shipping it.

---

## 7. Styling

- Use `<the project's chosen system>` consistently — don't mix inline styles, CSS modules, and a CSS-in-JS library in the same feature.
- No inline `style={{ ... }}` for anything expressible in the styling system, except values genuinely computed at runtime (e.g. a measured position).
- Respect existing design tokens (spacing, color, typography scale) instead of hardcoding pixel values or hex codes.
- Mobile-first responsive rules; verify at minimum a small mobile width and a standard desktop width before calling UI work done.

---

## 8. Accessibility

Non-negotiable, not a nice-to-have:

- Every interactive element is keyboard-operable (works with Tab/Enter/Space, has visible focus).
- Use semantic HTML first (`<button>`, `<nav>`, `<label>`) before reaching for ARIA attributes.
- Every `<img>` has meaningful `alt` text (or `alt=""` if purely decorative).
- Form inputs have an associated `<label>`.
- Color is never the only signal for state (error, success, disabled) — pair it with an icon or text.
- Run `eslint-plugin-jsx-a11y` (already in the lint config) and address its warnings; don't disable a rule to silence it.

---

## 9. Testing

- **React Testing Library** — test components the way a user interacts with them: query by role/label/text, not by implementation detail (no snapshot-testing internal state, no reaching into component internals).
- Prefer `getByRole` / `getByLabelText` over `getByTestId`. Add `data-testid` only when there's no accessible query available.
- Use `userEvent`, not `fireEvent`, for interactions — it models real browser event sequences.
- Test behavior, not implementation: assert on what's rendered/called, not on internal state shape or hook call counts.
- Mock the network layer (MSW) rather than mocking `fetch`/API client modules directly, so tests exercise the real request/response flow.
- Every bug fix ships with a regression test that fails before the fix.
- No arbitrary `setTimeout`/sleep in tests — use RTL's `findBy*`/`waitFor`.
- E2E (Playwright) covers critical user flows (auth, checkout, primary CRUD) — not every component variant; that's what unit/component tests are for.

---

## 10. Dependencies

- Check whether React or the existing toolkit already solves the problem before adding a package (e.g. don't add a date library for something `Intl.DateTimeFormat` handles).
- Adding a new runtime dependency requires justification in the PR: what it does, why it can't be done in ~30 lines, bundle-size impact, maintenance status.
- Check bundle size impact for anything added to a client-shipped path (use the bundle analyzer before/after for non-trivial additions).
- Never add a dependency to work around a type error.
- Pin exact versions for anything security-sensitive.

---

## 11. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `chore/...`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat(orders): add partial refunds ui`.
- Keep commits focused; unrelated formatting churn goes in its own commit.
- PR description: what changed, why, how it was verified (include a screenshot/GIF for UI changes), any accessibility considerations.
- Never force-push to `main`. Never amend or rebase a commit someone else may have pulled.
- Do not commit: `node_modules/`, `dist/`, `.env*`, editor config, `*.log`, coverage output.

---

## 12. If this is actually Next.js

This file is written for a plain client-rendered React app. If the project uses Next.js, keep everything above (component structure, hooks, state, TypeScript, styling, accessibility, testing, dependency, and git rules all still apply) and layer on:

- **Router:** state whether the project uses the **App Router** (`app/`) or **Pages Router** (`pages/`) — conventions differ significantly; don't mix new code in the router the project isn't using.
- **Server vs. Client Components (App Router):** default to Server Components; add `"use client"` only where interactivity, hooks, or browser APIs require it, and push that boundary as far down the tree as possible (wrap the interactive leaf, not the whole page).
- **Data fetching:** fetch data in Server Components / route handlers, not `useEffect`. Document the project's caching intent explicitly (`fetch(url, { cache: ... })` / `revalidate`) rather than relying on defaults, since Next's caching behavior has changed across versions.
- **Routing structure:** file-based routes (`page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts`) replace `src/routes/` from §4 — document the actual `app/`/`pages/` layout here instead.
- **Navigation & assets:** use `next/link` for internal navigation and `next/image` for images — don't reach for raw `<a>`/`<img>` on paths those cover.
- **Environment variables:** only variables prefixed `NEXT_PUBLIC_` are exposed to the browser; document which secrets must stay server-only.
- **Route Handlers / Server Actions:** treat these like backend endpoints — validate input (e.g. with `zod`), and apply the same error-handling and logging discipline you'd want in any API layer.
- **Metadata/SEO:** use the Metadata API (`generateMetadata` / the `metadata` export) rather than manually injecting `<head>` tags.

If you're maintaining a Next.js project long-term, it's worth asking me for a dedicated Next.js AGENTS.md rather than bolting this section on indefinitely — the App Router conventions above are substantial enough to deserve their own file.

---

## 13. Rules for agents

**Do**
- Read the surrounding component/feature before editing; match its existing patterns over the ones in this file when they conflict, and flag the conflict.
- Make the smallest change that fully solves the problem.
- Run `pnpm verify` and report the actual output before claiming success.
- Check accessibility (§8) for any new or changed interactive UI.
- Ask before: adding a new state-management library, changing the router, altering a shared design token, or adding a dependency.

**Don't**
- Don't refactor, rename, or reformat code outside the scope of the task.
- Don't delete or weaken a failing test to make the suite green. Fix the code or report the failure.
- Don't introduce a second way of doing something the codebase already does one way (a second HTTP client, a second styling approach, a second state library).
- Don't leave `TODO` without an owner and ticket: `// TODO(alice, PROJ-123): ...`.
- Don't mark work complete with a failing typecheck, lint error, or skipped test.

---

## 14. Definition of done

1. `pnpm verify` passes locally.
2. New behaviour has component/unit test coverage; bug fixes include a regression test.
3. No new `any`, `@ts-ignore`, unnecessary `console.log`, or lint/a11y suppressions.
4. UI changes were checked against §8 (accessibility) and at a mobile + desktop viewport.
5. Public-facing changes are reflected in `README.md`/Storybook where applicable.
6. The PR description explains the change, how it was verified, and includes a screenshot/GIF for visual changes.
