# AGENTS.md

Development guidelines for AI coding agents working in this repository.
Humans are welcome to read it too — it is the single source of truth for how we build here.

> **Scope:** These rules apply to the whole repo. A nested `AGENTS.md` in a subdirectory overrides this file for that subtree.

---

## 1. Project overview

| Field | Value |
| --- | --- |
| Name | `<project-name>` |
| Framework | React `19.x`, function components only |
| Build tool | Vite |
| Language | TypeScript (strict) |
| Package manager | `pnpm` (do **not** use `npm` or `yarn`) |
| Routing | `<react-router / tanstack-router / none>` |
| Server state | TanStack Query |
| Client state | React state + Context; `<zustand / none>` for cross-tree state |
| Styling | `<tailwind / css-modules>` |
| Forms | `react-hook-form` + `zod` |
| Testing | Vitest + React Testing Library; Playwright for e2e |

Replace the `<...>` placeholders before relying on this file.

---

## 2. Setup

```bash
nvm use
pnpm install --frozen-lockfile
cp .env.example .env.local   # Vite only exposes VITE_* vars to the client
pnpm dev
```

If `pnpm install` fails because the lockfile is out of date, stop and report it rather than regenerating the lockfile as a side effect of unrelated work.

---

## 3. Commands

Always use these scripts instead of invoking tools directly, so local runs match CI.

| Task | Command |
| --- | --- |
| Dev server | `pnpm dev` |
| Production build | `pnpm build` |
| Preview built app | `pnpm preview` |
| Unit/component tests | `pnpm test` |
| Single test file | `pnpm test src/features/cart/CartItem.test.tsx` |
| Tests by name | `pnpm test -t "disables checkout when empty"` |
| Watch tests | `pnpm test:watch` |
| Coverage | `pnpm test:coverage` |
| E2E | `pnpm test:e2e` |
| Lint | `pnpm lint` |
| Lint + autofix | `pnpm lint:fix` |
| Format | `pnpm format` |
| Type check | `pnpm typecheck` |
| Full pre-push gate | `pnpm verify` |

`pnpm verify` runs `typecheck → lint → test → build`. **Run it before declaring any task done.**

---

## 4. Directory structure

Organise by feature, not by file type.

```
src/
├── main.tsx                  # entry: providers, router mount
├── app/
│   ├── App.tsx
│   ├── providers.tsx         # QueryClient, theme, router, error boundary
│   └── routes.tsx
├── features/
│   └── cart/
│       ├── components/       # UI private to this feature
│       ├── hooks/            # useCart, useCartTotals
│       ├── api/              # query/mutation hooks + request functions
│       ├── types.ts
│       └── index.ts          # the feature's public surface
├── components/
│   └── ui/                   # generic, app-agnostic primitives (Button, Dialog)
├── hooks/                    # cross-feature hooks
├── lib/                      # pure utilities, http client, formatters
├── styles/
└── types/
```

Rules:
- A feature imports another feature only through its `index.ts`. Never reach into `features/x/components/...` from `features/y`.
- `components/ui` must not import from `features/`. It knows nothing about the domain.
- Shared code earns its way into `lib/` or `components/ui/` after a second consumer appears — not before.

---

## 5. Component rules

- Function components only. No class components except an error boundary.
- One exported component per file; the filename matches the component: `CartItem.tsx`.
- Keep components under ~150 lines. If a component grows past that, extract a child component or a hook.
- Props are typed with an explicit `type Props = {...}`. No `React.FC`. Don't spread arbitrary props onto a DOM node unless the component is a `ui/` primitive wrapping that element.
- Destructure props in the signature with defaults inline.
- Components describe *what* to render. Data fetching, subscriptions, and orchestration go in hooks.
- Never define a component inside another component's body — it remounts the whole subtree on every render.
- Prefer composition (`children`, slots) over boolean prop explosions. More than ~3 boolean props usually means two components.

```tsx
type Props = {
  item: CartLine;
  onRemove: (id: string) => void;
  compact?: boolean;
};

export function CartItem({ item, onRemove, compact = false }: Props) {
  // ...
}
```

### Rendering
- Every item in a list needs a stable domain `key`. **Never use the array index** unless the list is static and never reordered.
- Derive values during render instead of mirroring props into state. State that can be computed from props is a bug waiting to happen.
- Use ternaries or early returns for conditional rendering. Guard against `0 && ...` rendering a stray `0` — write `list.length > 0 && ...`.

---

## 6. Hooks and state

- Rules of Hooks are non-negotiable: top level only, never inside conditions or loops. The `eslint-plugin-react-hooks` rules are errors, not warnings.
- **Never disable `react-hooks/exhaustive-deps`.** If the lint rule complains, the dependency list is wrong — restructure the effect instead of silencing it.
- Custom hooks start with `use`, live beside the feature that owns them, and return an object (not a positional tuple) once there are more than two values.

### `useEffect` is a last resort
Do not use an effect to:
- transform data for rendering → compute it during render
- respond to a user event → do it in the event handler
- sync state that derives from props → derive it, or `key` the component to reset it
- fetch data → use TanStack Query

Legitimate uses: subscribing to an external system, imperative DOM work, analytics on mount. Every effect that subscribes must return a cleanup function.

### State placement
1. Local `useState` first.
2. Lift to the nearest common parent when two siblings need it.
3. URL search params for anything shareable or bookmarkable (filters, tabs, pagination).
4. TanStack Query for server data — it is the cache; do not copy query results into `useState`.
5. Context or the store only for genuinely global, low-frequency state (auth, theme, locale).

Context holding frequently-changing values re-renders every consumer. Split value and setter into separate contexts, or use a store.

### Memoisation
- Don't reach for `useMemo`/`useCallback`/`memo` by default. Add them for measured problems: expensive computation, referentially-stable props into a memoised child, or a dependency of an effect.
- Premature memoisation adds cost and hides the real re-render cause. Profile first with React DevTools.

---

## 7. Data fetching

- All server state goes through TanStack Query. No `fetch` inside `useEffect`.
- Query keys are structured and centralised per feature: `['cart', cartId]`. Never inline an ad-hoc string key.
- Every screen handles all four states explicitly: loading, error, empty, and success. An empty state is not the same as loading.
- Mutations invalidate the affected query keys in `onSuccess`. Optimistic updates must implement `onError` rollback.
- The HTTP client lives in `src/lib/http.ts` and attaches auth, base URL, timeout, and error normalisation. Don't call `fetch` directly in a feature.
- Validate API responses with `zod` at the boundary. Do not trust the server to match the TypeScript type.

---

## 8. Forms

- `react-hook-form` with a `zod` resolver. The zod schema is the single source of truth for both validation and the inferred TypeScript type.
- Inputs stay uncontrolled via `register` unless a controlled component genuinely requires `Controller`.
- Disable submit while the mutation is in flight; surface server-side field errors back into the form via `setError`.
- Every input has an associated `<label htmlFor>`. Error text is linked with `aria-describedby` and `aria-invalid`.

---

## 9. Styling

- Use the tokens in `src/styles` (or the Tailwind theme). No hardcoded hex colours, magic pixel values, or one-off `z-index` numbers.
- No inline `style` except for genuinely dynamic values (a computed transform, a measured width).
- Build mobile-first. Don't hide content at small widths as a layout fix.
- Respect `prefers-reduced-motion` for any animation.
- Don't use `!important`. If specificity is fighting you, the selector is wrong.

---

## 10. Accessibility

Non-negotiable, checked in review:
- Semantic HTML first. A clickable thing is a `<button>`; navigation is an `<a>`. Never attach `onClick` to a `<div>`.
- Every interactive element is reachable and operable by keyboard, with a visible focus ring. Do not remove `outline` without a replacement.
- Images have meaningful `alt`, or `alt=""` when decorative.
- Dialogs trap focus, close on `Escape`, and restore focus to the trigger on close.
- Colour is never the sole carrier of meaning; contrast meets WCAG AA.
- Run `eslint-plugin-jsx-a11y` clean. Don't disable its rules.

---

## 11. Performance

- Route-level code splitting with `React.lazy` + `Suspense`. Split heavy, rarely-used components (editors, charts, maps) too.
- Virtualise lists longer than ~100 rows.
- Keep the main bundle within the budget enforced in CI. If a change pushes it over, justify it in the PR.
- Images: correct dimensions, modern formats, `loading="lazy"` below the fold.
- Fix re-render problems by moving state down or splitting context before adding `memo`.

---

## 12. Testing

- Vitest + React Testing Library for unit and component tests; Playwright for e2e.
- **Test behaviour users can observe, not implementation.** No assertions on state, props, or internal function calls.
- Query priority: `getByRole` → `getByLabelText` → `getByText`. Use `getByTestId` only when nothing else can identify the element.
- Interact with `@testing-library/user-event`, not `fireEvent`.
- Use `findBy*` / `waitFor` for async UI. No arbitrary `setTimeout` waits.
- Mock the network with MSW at the HTTP boundary. Do not mock your own hooks or components to make a test pass.
- Snapshot tests only for small, stable output. A large snapshot asserts nothing useful.
- Every bug fix ships with a regression test that fails before the fix.

---

## 13. TypeScript

- `strict: true`. **No `any`.** Use `unknown` plus narrowing.
- No `@ts-ignore`. Use `@ts-expect-error` with a reason and a ticket if you truly must.
- Type event handlers properly: `React.ChangeEvent<HTMLInputElement>`, `React.FormEvent<HTMLFormElement>`.
- Derive types from zod schemas (`z.infer`) instead of hand-writing a parallel type.
- Prefer discriminated unions over optional-field soup for mutually exclusive states.

---

## 14. Dependencies

- Prefer the platform and what's already installed. Check the existing deps before adding anything.
- A new dependency needs justification in the PR: what it does, why we can't do it in ~30 lines, maintenance status, and bundle cost.
- Never add a date library for one `Intl.DateTimeFormat` call, or a utility library for one `Array.prototype` method.
- Don't run `pnpm update` as part of an unrelated change.

---

## 15. Security

- Never use `dangerouslySetInnerHTML`. If markup rendering is truly required, sanitise with DOMPurify and document why in a comment.
- Only `VITE_*` env vars reach the client — treat every one of them as public. No secrets, API keys, or private endpoints in client code.
- Validate and encode any user-controlled value used in a URL, `href`, or redirect. Reject `javascript:` schemes.
- Don't log tokens, PII, or full API payloads to the console.

---

## 16. Git and pull requests

- Branches: `feat/short-description`, `fix/short-description`, `chore/...`.
- Commits follow [Conventional Commits](https://www.conventionalcommits.org/): `feat(cart): support partial refunds`.
- PR description states what changed, why, how it was verified, and includes a screenshot or clip for any visual change.
- Never force-push to `main`. Never amend a commit others may have pulled.
- Do not commit: `node_modules/`, `dist/`, `.env.local`, coverage output, editor config.

---

## 17. Rules for agents

**Do**
- Read the surrounding feature before editing; match its existing patterns over the ones in this file when they conflict, and flag the conflict.
- Reuse an existing `ui/` primitive rather than writing a new button, input, or modal.
- Make the smallest change that fully solves the problem.
- Run `pnpm verify` and report the actual output before claiming success.
- Ask before: changing a shared `ui/` component's API, adding a dependency, restructuring routing, or introducing a new state management library.

**Don't**
- Don't refactor, rename, or reformat code outside the scope of the task.
- Don't delete or skip a failing test to make the suite green. Fix the code or report the failure.
- Don't invent component props, API endpoints, or env vars — grep the repo and confirm they exist.
- Don't disable `react-hooks` or `jsx-a11y` lint rules.
- Don't leave `TODO` without an owner and ticket: `// TODO(alice, PROJ-123): ...`.
- Don't mark work complete with a failing typecheck, lint error, or skipped test.

---

## 18. Definition of done

1. `pnpm verify` passes locally.
2. New behaviour is covered by tests; bug fixes have a regression test.
3. All four UI states (loading, error, empty, success) are handled for any new data-driven view.
4. Keyboard navigation and focus states work; a11y lint is clean.
5. No new `any`, `@ts-ignore`, `console.log`, or lint suppressions.
6. Bundle size is within budget, or the increase is justified in the PR.
7. The PR includes a screenshot or clip of any visual change.
