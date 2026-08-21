---
name: review-web
description: Repo-type checklist for web and frontend PRs (React, Vue, Svelte, Next.js, Angular). Layer on top of review-core. Covers rendering correctness, state, accessibility, bundle cost, and client-side security.
---

# Web / frontend review checklist

Layered on `review-core`. Severity definitions and the output template come from there — this file only adds *what to look for*.

## Internationalisation — severity depends on the repo

**Establish first whether the repo already has i18n.** Look for: `i18n/`, `locales/`, `messages/`, `lang/`, `.po`/`.arb`/`.ftl` files, or `react-i18next` / `next-intl` / `vue-i18n` / `lingui` / `formatjs` / `svelte-i18n` in the manifest, or a `t()` / `$t()` / `<Trans>` helper already in use. State which case applies in the review.

**"User-facing text" means:** visible copy, button and menu labels, headings, placeholders, validation and error messages, toasts, empty states, tooltips, `aria-label`s, page titles, and email/notification copy rendered from this repo. It does **not** mean log lines, console output, test fixtures, analytics event names, or internal-only debug tooling — do not flag those.

### Case A — the repo already has i18n

The bar is hard here, because a hardcoded string in an i18n'd codebase is a permanent hole that nobody comes back for.

- Any new user-facing string hardcoded in a component → **BLOCKER (MAINTAINABILITY)**. No exemption for "it's only a label" or "it's temporary".
- Key added to the default locale file but missing from the other locale files present in the repo → **BLOCKER** — this ships an untranslated or empty string to real users.
- Sentence assembled by **concatenation** (`t('you_have') + count + t('items')`) instead of one key with interpolation params → SHOULD-FIX. Word order is not universal; concatenation cannot be translated correctly.
- Pluralisation done with a ternary on `count` instead of the library's plural API → SHOULD-FIX. Many languages have more than two plural forms.
- Dates, numbers, currency, or relative times formatted by hand instead of via `Intl` or the i18n formatter → SHOULD-FIX.
- New key that doesn't follow the repo's existing key-naming convention (namespacing, casing) → NIT.
- Layout that assumes English string length (fixed-width buttons, `nowrap` on translated copy, no room for ~30% expansion in German) → NIT, or SHOULD-FIX if it visibly truncates.

### Case B — the repo has no i18n yet

Treat this as forward-planning advice, **not** an obstruction. Do not flood the review.

- Report **one consolidated NIT** for the whole PR, not one per string: *"N user-facing strings are hardcoded (`file:line`, `file:line`, …). No i18n in this repo yet, so this is not blocking — but extracting them into a `messages.ts` constants module now makes a future i18n migration a mechanical change instead of a rewrite."*
- If the PR adds a **large** new user-facing surface (a whole page, flow, or feature), raise it once as a PROCESS SHOULD-FIX recommending the strings go into a messages module from the start.
- Suggest adopting i18n **at most once per repo**, not on every review. If a previous review already raised it, don't repeat it.

Note that §3.1 of `review-core` (no hardcoded literals) already covers non-display strings in both cases — the i18n rules above are specifically about *user-visible copy*.

## Correctness

- **Effect dependencies.** `useEffect`/`watch` with a wrong or missing dependency array → stale closure or infinite loop. BLOCKER if it causes an infinite render or a request loop; SHOULD-FIX otherwise.
- **Async race on unmount.** A fetch that resolves after unmount and sets state, with no abort/cleanup. SHOULD-FIX; BLOCKER if it writes user data.
- **Key on lists.** Array index used as `key` on a reorderable or filterable list → wrong item state after reorder. SHOULD-FIX.
- **Derived state duplicated into state.** Values that could be computed during render stored in state and now able to drift. SHOULD-FIX.
- **Optional chaining hiding a bug.** `a?.b?.c` added to silence a crash instead of fixing why `a` is undefined. SHOULD-FIX, and say what the real cause looks like.
- **Uncontrolled → controlled input flips** (value starts `undefined`). NIT unless it breaks form submit.

## Data fetching and state

- Loading, empty, and error states all handled for any new fetch. Missing error state on a user-facing surface → SHOULD-FIX.
- Cache invalidation on mutation (React Query/SWR/Apollo): a mutation that does not invalidate or update the affected query leaves stale data on screen → SHOULD-FIX.
- Waterfall requests that could be parallel, or a request inside a list render → PERF SHOULD-FIX (BLOCKER if it's per-row on an unbounded list).
- Global state added for something only one subtree needs → MAINTAINABILITY NIT.

## Security (client side)

- `dangerouslySetInnerHTML` / `v-html` / `innerHTML` with any value not provably static or sanitised → **BLOCKER (SECURITY)**.
- Secrets, API keys, or tokens in client-bundled code or in `NEXT_PUBLIC_*` / `VITE_*` env vars → **BLOCKER (SECURITY)**.
- Tokens in `localStorage` where the codebase otherwise uses httpOnly cookies → SECURITY SHOULD-FIX (BLOCKER if it's a new auth flow).
- `target="_blank"` without `rel="noopener"` → NIT.
- User-controlled value flowing into `href`/`src` without a scheme allowlist (`javascript:` URLs) → SECURITY BLOCKER.
- Authorisation decided only in the client (a hidden button as the access control) → **BLOCKER**, note that the server must enforce it.

## Accessibility

Treat as SHOULD-FIX by default; BLOCKER only if the repo has a stated a11y compliance requirement.

- Interactive element that is a `div`/`span` with an onClick and no role, tabindex, or key handler.
- Form control without an associated label; icon-only button without `aria-label`.
- New modal/drawer without focus trap, focus restore, or Escape handling.
- Colour used as the sole signal for state (error, required).
- Heading levels skipped, or `alt` missing on meaningful images.

## Performance and bundle

- A new heavy dependency (date/lodash/chart/icon libraries) imported wholesale rather than per-function, or added when a small local utility would do → PERF SHOULD-FIX; state the approximate cost.
- Large component or route not code-split when siblings are → SHOULD-FIX.
- Non-memoised expensive computation or a new object/function prop that breaks memoisation of a heavy child → SHOULD-FIX only when the child is genuinely expensive. **Do not report speculative memoisation as a finding.**
- Images without dimensions (layout shift) or unoptimised full-size assets → SHOULD-FIX.
- Long list rendered without virtualisation where the data is unbounded → PERF BLOCKER.

## Tests

- New branching UI logic with no test → TESTS SHOULD-FIX.
- Tests asserting on implementation detail (internal state, class names) rather than user-visible behaviour → NIT.
- Snapshot test added as the only coverage for logic → SHOULD-FIX.

## Repo consistency

Check the diff against how the rest of this repo already does things — styling approach, folder conventions, data-fetching library, form library. A PR introducing a second way of doing an already-solved thing is a MAINTAINABILITY SHOULD-FIX; name the existing pattern it should have used.
