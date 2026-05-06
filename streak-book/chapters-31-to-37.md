# Part V — SvelteKit routing and the file conventions

> *"By Chapter 37 the reader can map any URL in their app to its file, and explain server vs client execution out loud."*

---

# Chapter 31 — `+page.svelte` vs `+page.ts` — the first split

> *Today's job:* the home page reads habits from `+page.ts` instead of hardcoding them. Visible win: the `data` prop carries the data, and refresh works the same.

---

## Lesson 31.1 — The split

> *"`+page.ts` decides what data the page needs; `+page.svelte` decides how to render it."*

`+page.ts` exports a `load` function that returns data. `+page.svelte` reads the result via the `data` prop.

```ts
// src/routes/+page.ts
import type { PageLoad } from './$types';
import type { Habit } from '$lib/types';
import { habitId } from '$lib/types';

export const load: PageLoad = async () => {
  const habits: Habit[] = [
    { id: habitId('demo-1'), name: 'Drink water', createdAt: Date.now() - 86400000 },
    { id: habitId('demo-2'), name: 'Read 20 minutes', createdAt: Date.now() - 3600000 },
  ];
  return { habits };
};
```

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<h1>Today</h1>
<p>{data.habits.length} habits.</p>
```

> **`load`** — a function exported from `+page.ts`/`+page.server.ts` that returns the data the page needs.
>
> **`./$types`** — auto-generated types. `PageLoad`, `PageProps`, `LayoutLoad`, `PageServerLoad`, etc.
>
> **universal `load`** — a `+page.ts` `load` runs on the server during SSR, then on the client during navigation. It must not access secrets or DB.

> **Version note** — `PageProps` was added to `./$types` in SvelteKit 2.16 (December 2024). The book pins to SvelteKit 2.5x (May 2026). Older versions used `$$Props` or hand-typed `data`; if you're reading code from before late 2024 you'll see those shapes instead.

### Read this code

**Snippet A** — given:

```ts
// src/routes/+page.ts
export const load: PageLoad = async () => {
  console.log('load running');
  return { now: Date.now() };
};
```

You hard-reload `/` in the browser. How many times does `'load running'` print, and where?

<details>
<summary>Worked answer</summary>

Twice. Once in your terminal (the server log, during SSR), once in the browser console (the client, during hydration). A universal `load` runs on the server first to render the HTML, then SvelteKit re-runs it on the client so the same `data` is reactive to client navigations. Subsequent client-side navigations only run it on the client.

If you want `load` to run server-only, move it to `+page.server.ts` — then it runs once per request, on the server, and the client gets the serialised result.
</details>

**Snippet B** — given a route folder `src/routes/habits/[id]/+page.ts`, what is the type of `params.id` inside `load`?

<details>
<summary>Worked answer</summary>

`string`. Folder names like `[id]` always become `string` keys on `params`, regardless of what shape you intend (numeric ID, UUID, slug). If you want a branded `HabitId`, you have to validate in `load` (Chapter 27's parsers) — TypeScript can't infer the brand from a folder name.
</details>

---

## Lesson 31.2 — `data` is read-only from the page's perspective

The page can't mutate `data` directly. Mutations go through actions or store methods. We seed the existing `HabitStore` from `data.habits` *immutably*:

```svelte
<script lang="ts">
  import type { PageProps } from './$types';
  import { HabitStore } from '$lib/habits.svelte';

  let { data }: PageProps = $props();
  const store = new HabitStore();
  // Seed with the loaded habits — assign, don't mutate (Ch 8's rule still holds).
  store.habits = [...data.habits];
</script>
```

(Cleaner is to pass the seed into the constructor; we'll do that in Chapter 32 once context enters.)

> **SSR note.** This `const store = new HabitStore()` at the top of `+page.svelte` is per-component-instance, not module-level — it's SSR-safe. The `localStorage`-seeded singleton in Chapter 38 is where SSR matters; we'll context-ise it in Chapter 32.

---

## Lesson 31.3 — The special `fetch`

`load` receives a special `fetch`:

```ts
import { parseHabits } from '$lib/parseHabit';

export const load: PageLoad = async ({ fetch }) => {
  const res = await fetch('/api/habits');
  const raw: unknown = await res.json();
  return { habits: parseHabits(raw) };
};
```

This fetch:
- inherits cookies for same-origin during SSR,
- inlines responses into the SSR HTML so the client doesn't refetch,
- passes through CSRF protections.

Don't use the global `fetch` in `load` — use the one from the parameter. And don't trust the JSON shape: route it through the boundary parser from Chapter 26 (here, `parseHabits`).

> **`Result` reconciliation.** Once Chapter 27 lands `Result`, `parseHabits` returns an array where each entry is `Result<Habit, ParseError>`. We drop the errors and keep the OK rows here:
>
> ```ts
> export const load: PageLoad = async ({ fetch }) => {
>   const res = await fetch('/api/habits');
>   const raw: unknown = await res.json();
>   const parsed = parseHabits(raw);
>   const habits = parsed.filter((r) => r.ok).map((r) => r.value);
>   return { habits };
> };
> ```
>
> In production you'd want to log the dropped error rows (Chapter 56's `handleError` is the natural sink). For now the filter keeps the page rendering when one row is malformed instead of nuking the whole page.

---

## Lesson 31.4 — Now you write it

**The English sentence first:**

> *"Build a `/stats` route. `+page.ts` returns a hardcoded `{ totalHabits: 5, addedToday: 2 }`. `+page.svelte` displays both numbers."*

<details>
<summary>Worked answer</summary>

`src/routes/stats/+page.ts`:

```ts
import type { PageLoad } from './$types';

export const load: PageLoad = async () => {
  return { totalHabits: 5, addedToday: 2 };
};
```

`src/routes/stats/+page.svelte`:

```svelte
<script lang="ts">
  import type { PageProps } from './$types';
  let { data }: PageProps = $props();
</script>

<h1>Stats</h1>
<p>Total habits: {data.totalHabits}</p>
<p>Added today: {data.addedToday}</p>
```

Visit `/stats`. Renders.
</details>

---

## Lesson 31.5 — Recurring concepts from earlier chapters

- **`parseHabits`** (Ch 26) — every fetch boundary uses it.
- **`HabitStore`** (Ch 29) — re-seeded from `data` on each load.
- **Immutable assignment** (Ch 8) — `store.habits = [...data.habits]`, never `.push()`.

---

## Lesson 31.6 — What you can now read in the wild

After Chapter 31 you can:

- Read **`export const load: PageLoad = async ({ fetch, params, url, depends, parent }) => { ... }`** confidently.
- Read **`let { data }: PageProps = $props()`** in `+page.svelte`.
- Read the special **`fetch` parameter** as the SSR-aware fetcher.
- Spot **untrusted JSON** entering `load` and add a `parseFoo` boundary call.

---

## End-of-chapter checkpoint

- [ ] `+page.ts` exists at `src/routes/+page.ts`.
- [ ] The home page receives `data.habits`.
- [ ] You wrote a `/stats` route end to end.

---

# Chapter 32 — `+layout.svelte` — chrome around every page

> *Today's job:* a top nav with the Streak logo and a Sign-in placeholder, on every route. Visible win: same nav on `/` and `/stats`.

---

## Lesson 32.1 — Layouts

`src/routes/+layout.svelte` wraps every page:

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { children }: { children: Snippet } = $props();
</script>

<header>
  <a href="/" class="logo">Streak</a>
  <nav>
    <a href="/stats">Stats</a>
    <a href="/login">Sign in</a>
  </nav>
</header>

<main>
  {@render children()}
</main>

<footer>
  <small>© Streak — May 2026</small>
</footer>

<style>
  header { display: flex; justify-content: space-between; padding: 1rem; border-bottom: 1px solid #eee; }
  .logo { font-weight: 700; text-decoration: none; }
  nav a { margin-left: 1rem; }
  main { padding: 1rem; max-width: 720px; margin: 0 auto; }
  footer { text-align: center; padding: 2rem; color: #888; }
</style>
```

`{@render children()}` — Chapter 20's snippet rendering — places the page's content here.

---

## Lesson 32.2 — `+layout.ts` for layout-level data

```ts
// src/routes/+layout.ts
import type { LayoutLoad } from './$types';

export const load: LayoutLoad = async () => {
  return {
    appName: 'Streak',
    version: '0.1.0',
  };
};
```

In `+layout.svelte`, `data` is now available:

```svelte
<script lang="ts">
  import type { LayoutProps } from './$types';
  let { data, children }: LayoutProps = $props();
</script>

<small>{data.appName} v{data.version}</small>
```

Page `load`s can `await parent()` to get layout data:

```ts
export const load: PageLoad = async ({ parent }) => {
  const { appName } = await parent();
  return { appName };
};
```

---

## Lesson 32.3 — Per-request stores via context

The senior pattern for per-user/per-request stores: `setContext`/`getContext` in the layout. This *replaces* the per-page `const store = new HabitStore()` we wrote in Chapter 31.

> **`setContext(key, value)` / `getContext(key)`** — Svelte primitives for sharing values down a component tree without prop-drilling. The key can be a string or (preferred for big apps) a `Symbol`.

**Lead with the typed wrapper.** Define `setHabitStore` / `getHabitStore` once, in `$lib/contexts.ts`, and have the rest of the codebase call those helpers exclusively:

```ts
// ✅ src/lib/contexts.ts — the only file that touches setContext/getContext directly.
import { setContext, getContext } from 'svelte';
import type { HabitStore } from '$lib/habits.svelte';

const HABIT_STORE_KEY = Symbol('habit-store');

export function setHabitStore(store: HabitStore): void {
  setContext(HABIT_STORE_KEY, store);
}

export function getHabitStore(): HabitStore {
  const store = getContext<HabitStore | undefined>(HABIT_STORE_KEY);
  if (store === undefined) {
    throw new Error('HabitStore not in context — did you call setHabitStore?');
  }
  return store;
}
```

Then use the helpers, never the raw primitives:

```svelte
<!-- +layout.svelte -->
<script lang="ts">
  import { HabitStore } from '$lib/habits.svelte';
  import { setHabitStore } from '$lib/contexts';

  const store = new HabitStore();
  setHabitStore(store);
</script>
```

```svelte
<!-- any descendant page (e.g. +page.svelte) -->
<script lang="ts">
  import { getHabitStore } from '$lib/contexts';
  const store = getHabitStore();
</script>
```

Each user navigating their own browser tab has their own component tree, hence their own `HabitStore`. The `Symbol` makes the key globally unique; the wrapper enforces presence at the type level. No `as` cast anywhere. No SSR-singleton landmine.

> **Symbol keys and SSR.** Symbol-as-key is module-scope; it doesn't share state across SSR requests, just identity. Each request still gets fresh instances if instantiated per-request inside a component or layout. The Symbol is the same identity in every render; the *value* stored under it lives inside the component tree, which SvelteKit recreates per request.

**What the wrapper hides — never write this directly:**

```ts
// ❌ Bible rule #3 violation (`as` lies to the type system).
const store = getContext('habit-store') as HabitStore;
```

If the consumer's key doesn't match what was set, you get `undefined` at runtime that the type system doesn't catch. The wrapper above is what protects you from that.

---

## Lesson 32.4 — Recurring concepts from earlier chapters

- **`{@render children()}`** (Ch 20) — the layout wraps the page via the `children` snippet.
- **`HabitStore`** (Ch 29) — now per-tree, not per-page.
- **The SSR-singleton landmine** (Ch 29) — defeated by per-request context.

---

## Lesson 32.5 — What you can now read in the wild

After Chapter 32 you can:

- Read **`+layout.svelte`** as page chrome.
- Read **`+layout.ts`** as layout-level data loading.
- Read **`setContext` / `getContext`** with a typed wrapper.
- Read **`await parent()`** in a child `load` to inherit layout data.

---

## End-of-chapter checkpoint

- [ ] `+layout.svelte` exists with header/footer.
- [ ] Both `/` and `/stats` show the same chrome.
- [ ] (Optional) you wired the `HabitStore` via typed context.

---

# Chapter 33 — Route parameters: `/habits/[id]`

> *Today's job:* clicking a habit row navigates to `/habits/[id]` — a detail page. Visible win: per-habit page; bookmarking and refreshing work.

---

## Lesson 33.1 — `[id]` in the file path

Create `src/routes/habits/[id]/+page.ts` and `+page.svelte`:

```ts
// src/routes/habits/[id]/+page.ts
import type { PageLoad } from './$types';
import type { Habit, HabitId } from '$lib/types';
import { habitId, parseHabitId } from '$lib/types';
import { error } from '@sveltejs/kit';

export const load: PageLoad = async ({ params }) => {
  // params.id is `string` — narrow it to HabitId at the boundary.
  const id: HabitId | null = parseHabitId(params.id);
  if (id === null) {
    throw error(404, { message: 'Habit not found' });
  }

  // For now, fake data. Real DB in Part VI.
  const fakeHabits: Habit[] = [
    { id: habitId('demo-1'), name: 'Drink water', createdAt: Date.now() },
    { id: habitId('demo-2'), name: 'Read 20 minutes', createdAt: Date.now() },
  ];
  const habit = fakeHabits.find((h) => h.id === id);
  if (habit === undefined) {
    throw error(404, { message: 'Habit not found' });
  }
  return { habit };
};
```

> **`error(status, message)`** is documented to be used as `throw error(...)`. The helper does throw internally (its return type is `never`), so a bare `error(...)` works at runtime — but writing `throw` makes the control-flow visible in the source. Senior habit, and consistent with `throw redirect(...)`. We use `throw` everywhere.

The `parseHabitId` helper is the same shape we built in Chapter 27:

```ts
// src/lib/types.ts (sketch)
export function parseHabitId(s: string): HabitId | null {
  if (!/^[a-zA-Z0-9_-]{4,40}$/.test(s)) {
    return null;
  }
  return s as HabitId; // brand assertion lives behind one validated function
}
```

In `Result` form (post-Ch-27) it'd be `Result<HabitId, ParseError>`; the call site swaps to `if (!id.ok) { throw error(404, ...); }` and uses `id.value`. Either way, `params.id: string` never flows into a `HabitId`-typed slot without a check.

```svelte
<!-- src/routes/habits/[id]/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  import { formatRelativeTime } from '$lib/formatRelativeTime';

  let { data }: PageProps = $props();
</script>

<h1>{data.habit.name}</h1>
<p><small>Added {formatRelativeTime(data.habit.createdAt)}</small></p>
<p><a href="/">← Back</a></p>
```

Visit `http://localhost:5173/habits/demo-1`. The detail page renders. Visit `/habits/nonexistent` — `error(404, ...)` triggers.

---

## Lesson 33.2 — Optional and rest params

- `[[id]]` — optional segment.
- `[...rest]` — catch-all (matches `/foo/bar/baz` as `rest = 'foo/bar/baz'`).

---

## Lesson 33.3 — Param matchers

If you want only certain values to match:

```ts
// src/params/habitId.ts
export function match(param: string): boolean {
  return /^[a-zA-Z0-9_-]{4,40}$/.test(param);
}
```

Then `src/routes/habits/[id=habitId]/+page.svelte` — the `=habitId` ties the param to the matcher.

> **No runes in matchers.** Param matchers are pure functions in plain `.ts` files (not `.svelte.ts`); they don't use runes. They run on the server during request routing, before any component context exists. Keep them deterministic: same input, same boolean.

---

## Lesson 33.4 — Linking

In the home page (where `store` comes from context, Ch 32):

```svelte
<script lang="ts">
  import { getHabitStore } from '$lib/contexts';
  const store = getHabitStore();
</script>

{#each store.habits as habit (habit.id)}
  <li>
    <a href={`/habits/${habit.id}`}>{habit.name}</a>
  </li>
{/each}
```

Use `<a href>`, not click handlers + `goto`. SvelteKit intercepts; the user can middle-click, right-click, share. Senior habit.

---

## Lesson 33.5 — Recurring concepts from earlier chapters

- **`HabitId`** (Ch 27) — `params.id` is a `string`; you'd cast to `HabitId` (or rerun the boundary parser) before using it as an identity.
- **`error()`** — discriminated-error helper, throws on call.
- **Anchor tags** — bookmarkable, sharable, middle-clickable URLs are a senior habit.

---

## Lesson 33.6 — What you can now read in the wild

After Chapter 33 you can:

- Read **`[id]`**, **`[[optional]]`**, **`[...rest]`** in folder names.
- Read **`params.id`** in a `load` and know it's `string`.
- Read **param matchers** (`[id=habitId]`) and write your own.
- Read **`error(404, ...)`** as the expected-error shorthand.

---

## End-of-chapter checkpoint

- [ ] `/habits/[id]` works with a real habit.
- [ ] `/habits/nonexistent` returns 404.
- [ ] You can read `params.id`'s typing aloud.

---

# Chapter 34 — `+error.svelte` — friendly errors

> *Today's job:* when a route throws, render a polite card; the layout chrome stays. Visible win: `/habits/nope` shows a "Habit not found" card inside the normal nav.

---

## Lesson 34.1 — `+error.svelte`

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<div class="error-card">
  <h2>{page.status}</h2>
  <p>{page.error?.message ?? 'Something went wrong.'}</p>
  <a href="/">Go home</a>
</div>

<style>
  .error-card { padding: 2rem; text-align: center; border: 1px solid #fcc; border-radius: 0.5rem; }
</style>
```

Now `error(404, { message: 'Habit not found' })` from any `load` renders this. The layout's nav and footer still appear — only the page slot becomes the error card.

---

## Lesson 34.2 — Expected vs unexpected

- **Expected** — you `throw error(...)` deliberately. Caught by `+error.svelte`. Status code controllable.
- **Unexpected** — anything else thrown. Caught by `handleError` (Chapter 56) and rendered as a generic 500.

---

## Lesson 34.3 — `App.Error` shape

In `src/app.d.ts`:

```ts
declare global {
  namespace App {
    interface Error {
      message: string;
      code?: string;
    }
  }
}
export {};
```

This typing flows into `page.error`.

> **Forward reference.** Chapter 56 expands `App.Error` to include `category?: 'auth' | 'billing' | 'database' | 'unknown';` so `handleError` can route different failure classes to different telemetry buckets. We start minimal here — `message` plus an optional `code` is enough until the unexpected-error story lands.

---

## Lesson 34.4 — Now you write it

**The English sentence first:**

> *"Write a `+error.svelte` for `(app)/habits/[id]/` that distinguishes 404 (\"habit not found, try one of these:\") from other errors. Include a list of two example habit IDs the user could try."*

<details>
<summary>Worked answer</summary>

`src/routes/(app)/habits/[id]/+error.svelte`:

```svelte
<script lang="ts">
  import { page } from '$app/state';

  const examples: ReadonlyArray<{ id: string; name: string }> = [
    { id: 'demo-1', name: 'Drink water' },
    { id: 'demo-2', name: 'Read 20 minutes' },
  ];
</script>

<div class="error-card">
  {#if page.status === 404}
    <h2>Habit not found</h2>
    <p>The habit you tried to open doesn't exist or has been deleted.</p>
    <p>Try one of these:</p>
    <ul>
      {#each examples as example (example.id)}
        <li><a href={`/habits/${example.id}`}>{example.name}</a></li>
      {/each}
    </ul>
  {:else}
    <h2>{page.status}</h2>
    <p>{page.error?.message ?? 'Something went wrong.'}</p>
    <a href="/dashboard">Back to dashboard</a>
  {/if}
</div>

<style>
  .error-card { padding: 2rem; border: 1px solid #fcc; border-radius: 0.5rem; }
</style>
```

The route-specific `+error.svelte` *replaces* the parent one for errors thrown by this route or any descendant — SvelteKit walks up the tree until it finds an `+error.svelte`. Visit `/habits/nope`: this card shows. Throw a 500 from the load instead: the `else` branch shows.
</details>

---

## Lesson 34.5 — Read this code

**Snippet A** — given the root `+error.svelte` from Lesson 34.1, what shows when a `load` does `throw error(403, 'forbidden')` versus when a `load` runs an uncaught `throw new Error('boom')`?

<details>
<summary>Worked answer</summary>

For `throw error(403, 'forbidden')`:
- `page.status` is `403`.
- `page.error?.message` is `'forbidden'`.
- The card shows `403` and the literal string `forbidden`.

For uncaught `throw new Error('boom')`:
- This is *unexpected*. SvelteKit catches it, runs `handleError` (default or your override), and renders `+error.svelte` with `page.status === 500` and `page.error?.message === 'Internal Error'` (the safe default — your real message is logged server-side, not leaked to the user). You'd need to override `handleError` (Chapter 56) to surface a custom message to the UI.

The takeaway: `throw error(...)` is the only way to control the user-visible message from a `load`. Anything else gets sanitized to a generic 500 message.
</details>

---

## Lesson 34.6 — Recurring concepts from earlier chapters

- **`error()` helper** (Ch 33) — same call, now visualised by `+error.svelte`.
- **`?.` and `??`** (Ch 4) — `page.error?.message ?? 'Something went wrong.'`.
- **Layout cascade** (Ch 32) — `+error.svelte` renders inside the parent layout.

---

## Lesson 34.7 — What you can now read in the wild

After Chapter 34 you can:

- Read **`+error.svelte`** as the route-level error boundary.
- Read **`page.error`** and **`page.status`** from `$app/state`.
- Customise **`App.Error`** in `app.d.ts`.
- Tell where `+error.svelte` does *not* fire (root layout errors, `+server.ts` errors — those go to `src/error.html` or `handleError`).

---

## End-of-chapter checkpoint

- [ ] `+error.svelte` shows on a deliberate `error(404)`.
- [ ] The layout's chrome remains.

---

# Chapter 35 — `$app/state` — `page` and `navigating`

> *Today's job:* a "← Back" link in the layout that hides on the home page. Visible win: nav adapts to where you are.

---

## Lesson 35.1 — `page` rune

```svelte
<script lang="ts">
  import { page, navigating } from '$app/state';
</script>

<a href="/dashboard" class="logo" class:hidden={page.url.pathname === '/dashboard'}>Streak</a>
<small>{page.url.pathname}</small>
{#if navigating}<small>navigating…</small>{/if}

<style>
  .hidden { display: none; }
</style>
```

`page` and `navigating` are *reactive values backed by runes*, not user-callable runes themselves. They expose runes-tracked state via getters — reading `page.url.pathname` registers a dependency in the surrounding reactive context, so anything that reads it re-renders when the URL changes. You don't call them like `$state(...)`; you import them and read fields off them.

> **`$app/state`** — Svelte 5's runes-style replacement for the legacy `$app/stores`. Members: `page`, `navigating`, `updated`.

---

## Lesson 35.2 — `aria-current`

For nav links:

```svelte
<a href="/stats" aria-current={page.url.pathname === '/stats' ? 'page' : undefined}>Stats</a>
```

Screen readers announce *"current page"* on the active link.

---

## Lesson 35.3 — Now you write it

**The English sentence first:**

> *"Build a `<NavLink href, label>` component that wraps an `<a>`. It reads `page.url.pathname` and applies `aria-current=\"page\"` when its `href` matches the current path."*

<details>
<summary>Worked answer</summary>

`src/lib/NavLink.svelte`:

```svelte
<script lang="ts">
  import { page } from '$app/state';

  type Props = {
    href: string;
    label: string;
  };

  const { href, label }: Props = $props();
  const isCurrent = $derived(page.url.pathname === href);
</script>

<a {href} aria-current={isCurrent ? 'page' : undefined}>{label}</a>
```

Use it in the layout:

```svelte
<nav>
  <NavLink href="/dashboard" label="Dashboard" />
  <NavLink href="/stats" label="Stats" />
</nav>
```

When the user navigates between routes, `page.url.pathname` changes, `isCurrent` re-derives, and the `aria-current` attribute toggles automatically. No event listeners, no manual subscriptions — the `$derived` rune does the wiring.
</details>

---

## Lesson 35.4 — Read this code

**Snippet** — given:

```svelte
<script lang="ts">
  import { page } from '$app/state';
  const search = $derived(page.url.search);
</script>

<p>Query: {search}</p>
```

The user navigates from `/foo?q=hello` to `/foo?q=hello#section`. Does `search` re-compute?

<details>
<summary>Worked answer</summary>

Yes. `page.url` is a fresh `URL` instance per navigation — SvelteKit doesn't mutate the existing URL, it constructs a new one and re-assigns. So even though only the hash changed, the `URL` object is identity-different, and `page.url.search` is being read off a new object. The `$derived` re-runs and `search` is recalculated to the same string `'?q=hello'`.

The render output is identical (Svelte's reactivity reconciliation skips DOM updates when the new value matches the old), but the derivation *does* run. If you depended on the derivation NOT running for hash-only changes (e.g., it logs analytics on every change), you'd want to compare strings yourself, not rely on URL identity.
</details>

---

## Lesson 35.5 — Glossary

| Term | Meaning |
|---|---|
| `page` | Reactive value from `$app/state`. Exposes `url`, `params`, `route`, `status`, `error`, `data`, `form`, `state` of the current request. Reads register dependencies. |
| `navigating` | Reactive value from `$app/state`. Truthy (a `Navigation` object) while a client-side navigation is in flight, `null` otherwise. Use for spinners/progress UI. |
| `updated` | Reactive value from `$app/state`. `updated.current` flips `true` when SvelteKit detects a new app version is available (after `pollInterval`). Pair with a "reload to update" banner. |
| `aria-current` | ARIA attribute on a link/nav-item announcing the current location. Values: `'page'`, `'step'`, `'location'`, `'date'`, `'time'`, `'true'`, `'false'`, or omit. Screen readers announce *"current page"* when set to `'page'`. |

---

## Lesson 35.6 — Recurring concepts from earlier chapters

- **`class:foo={cond}`** (Ch 5) — used here for `class:hidden`.
- **`{#if navigating}`** — derived rendering on the navigation state rune.

---

## Lesson 35.7 — What you can now read in the wild

After Chapter 35 you can:

- Read **`import { page, navigating, updated } from '$app/state'`**.
- Read **`page.url.pathname`**, **`page.params`**, **`page.error`**, **`page.status`**, **`page.data`**, **`page.form`** as reactive properties.
- Read **`aria-current="page"`** on active nav links.
- Spot legacy **`$app/stores`** code as a refactor target.

---

## End-of-chapter checkpoint

- [ ] You replaced `$app/stores` use (if any) with `$app/state`.
- [ ] `aria-current="page"` on the active link.

---

# Chapter 36 — `$app/navigation` — `goto`, `invalidate`, `beforeNavigate`

> *Today's job:* after deleting a habit on the detail page, navigate home. Optionally guard "unsaved changes". Visible win: delete from `/habits/[id]`, you land on `/`.

---

## Lesson 36.1 — `goto`

```ts
import { goto } from '$app/navigation';

async function handleDelete(): Promise<void> {
  await goto('/dashboard');
}
```

---

## Lesson 36.2 — `invalidate`

```ts
import { invalidate } from '$app/navigation';

await invalidate('streak:habits'); // re-runs loads that called depends('streak:habits')
```

In a `load`:

```ts
export const load: PageLoad = async ({ depends }) => {
  depends('streak:habits');
  return { habits: /* ... */ };
};
```

Now any later `invalidate('streak:habits')` re-runs that load. The page's data prop refreshes without a full nav.

---

## Lesson 36.3 — `beforeNavigate`

```svelte
<script lang="ts">
  import { beforeNavigate } from '$app/navigation';
  let isDirty = $state(false);

  beforeNavigate(({ cancel }) => {
    // TODO: replace with confirmDialog at Ch 53
    if (isDirty && !window.confirm('Discard unsaved changes?')) {
      cancel();
    }
  });
</script>
```

> **Browser-native dialogs are placeholder code.** `window.confirm` / `window.alert` / `window.prompt` are unstyled and ignore the dark theme — they look like 1998 browser chrome dropped into a modern app. Senior teams replace them with a custom dialog primitive; Chapter 53 builds one. We use `window.confirm` for now to keep the focus on `beforeNavigate`. The `TODO` comment in the snippet is the real-world habit: never let a placeholder ship without a tracked replacement.

---

## Lesson 36.4 — Read this code

**Snippet** — given a `/dashboard` `+page.ts` with a `load` that prints `'dashboard load running'`. The user is on `/about` and code runs `await goto('/foo')`. Does the dashboard `load` re-run?

<details>
<summary>Worked answer</summary>

No. `goto('/foo')` navigates to `/foo`, not `/dashboard`. Loads run for the *destination* route's hierarchy (`/foo`'s `+page.ts`, plus any `+layout.ts` ancestors that depend on something that changed). The dashboard `+page.ts` is for `/dashboard` and isn't on the new route's hierarchy, so it doesn't run.

Two cases where the dashboard load *would* re-run:
1. `goto('/dashboard')` — destination is `/dashboard`, so its load runs.
2. After an `invalidate(...)` whose key matches a `depends(...)` declared in the dashboard load, *and* the user navigates back to `/dashboard`. The invalidation marks the load as stale; the next time `/dashboard` is visited, it re-runs.

Loads don't run "everywhere on every navigation" — they run for the route you're navigating *to*, plus shared ancestors whose dependencies changed.
</details>

---

## Lesson 36.5 — Now you write it

**The English sentence first:**

> *"Inside the root `+layout.svelte`, register an `afterNavigate` hook that scrolls the `<main>` element to the top after every client-side navigation."*

<details>
<summary>Worked answer</summary>

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  import { afterNavigate } from '$app/navigation';

  let { children }: { children: Snippet } = $props();
  let mainEl: HTMLElement | undefined = $state();

  afterNavigate(() => {
    mainEl?.scrollTo({ top: 0, behavior: 'instant' });
  });
</script>

<header><!-- nav --></header>
<main bind:this={mainEl}>
  {@render children()}
</main>
```

`afterNavigate` runs after every successful client-side navigation (including the initial one). The `mainEl?.scrollTo` is null-safe in case the binding hasn't attached yet. We use `'instant'` because a smooth scroll competes with the user's reading focus on a fresh page.

Why scope the scroll to `<main>` and not `window`? If the page layout puts the scrollbar on `<main>` (common with sticky headers), `window.scrollTo` is a no-op. Scoping to the actual scroll container is the senior habit.
</details>

---

## Lesson 36.6 — Glossary

| Term | Meaning |
|---|---|
| `goto(url, options?)` | Imperatively navigate. Returns a `Promise<void>` that resolves when the destination's loads finish. `await` it if you have follow-up logic. |
| `invalidate(key)` | Mark loads that called `depends(key)` as stale. Re-runs them in place; the page's `data` prop refreshes without a full nav. `key` is a string or a URL-matching predicate. |
| `invalidateAll()` | Mark *every* active load as stale. The blunt instrument; reach for `invalidate(key)` first. |
| `beforeNavigate({ from, to, cancel, type })` | Fires before a navigation. Call `cancel()` to block it. The classic unsaved-changes guard. Doesn't fire for full-page reloads (the browser owns those). |
| `afterNavigate({ from, to, type })` | Fires after a successful navigation, including the initial render. Use for analytics, scroll-restoration, focus management. |
| `depends(key)` | Declared *inside* a `load`. Registers that this load is invalidated by `invalidate(key)`. The handshake that wires `invalidate` to specific loads. |

---

## Lesson 36.7 — Recurring concepts from earlier chapters

- **`async`/`await`** (Ch 22) — `goto` and `invalidate` return Promises.
- **`window.confirm`** (Ch 15) — placeholder dialog; we'll replace with `<ConfirmDialog>` in Ch 53.
- **`depends('app:foo')`** — declares a custom invalidation key.

---

## Lesson 36.8 — What you can now read in the wild

After Chapter 36 you can:

- Read **`goto(url, options?)`** and decide whether to `await` it (await if you have follow-up logic).
- Read **`invalidate('app:foo')`** + `depends('app:foo')` in a load and trace re-runs.
- Read **`beforeNavigate(({ from, to, cancel }) => …)`** as the unsaved-changes guard.
- Read **`afterNavigate`**, **`preloadData`**, **`preloadCode`** in performance-tuning code.

---

## End-of-chapter checkpoint

- [ ] `goto`, `invalidate`, `beforeNavigate` each used somewhere.

---

# Chapter 37 — Route groups and the Part V checkpoint

> *Today's job:* split `/`, `/about`, `/pricing` (marketing) from `/app/...` (the app). Visible win: different chrome, same project.

---

## Lesson 37.1 — Route groups

```
src/routes/
  +layout.svelte                # base — common to ALL pages
  (marketing)/
    +layout.svelte              # marketing-specific chrome
    +page.svelte                # /
    about/+page.svelte
    pricing/+page.svelte
  (app)/
    +layout.svelte              # app-specific chrome
    +layout.server.ts           # auth gate (Chapter 45)
    habits/
      +page.svelte              # /habits
      [id]/+page.svelte         # /habits/[id]
```

`(marketing)` and `(app)` are **route groups** — folders that don't show up in the URL. They let you have different layouts for different sections.

---

## Lesson 37.2 — Layout resets

If you want a page *not* to inherit a parent layout:

```
+page@.svelte         # don't inherit immediate parent
+page@(marketing).svelte  # inherit specifically from (marketing)
```

Used rarely but knowing the syntax matters.

---

## Lesson 37.3 — Sorting rules

When two routes could match, SvelteKit picks the more *specific* one:

1. exact paths win over params.
2. matched params win over unmatched.
3. required params win over optional / rest.

You'll occasionally need to think about this; mostly it just works.

---

## Lesson 37.4 — The seven `+`-files cheat sheet

```
+page.svelte               page UI
+page.ts                   universal load (no DB/secrets)
+page.server.ts            server-only load + actions (DB, secrets, cookies)
+layout.svelte             layout UI
+layout.ts                 universal layout load
+layout.server.ts          server-only layout load (auth gating)
+server.ts                 REST endpoint (GET/POST/...)
+error.svelte              error boundary
+hooks.server.ts           handle/handleError/handleFetch
+hooks.client.ts           handleError on the client
+hooks.ts                  reroute, transport (universal)
+page@.svelte              page UI, layout-RESET — skip the immediate parent layout
+page@(group).svelte       page UI, layout-RESET — re-anchor to the named group/segment
+layout@.svelte            layout UI, layout-RESET — same idea, but for nested layouts
+layout@(group).svelte     layout UI, layout-RESET — re-anchor to the named group/segment
```

Print this. Stick it above your monitor.

The `@`-form (last four entries) is the layout-reset syntax introduced in Lesson 37.2 — `+page@.svelte` skips the immediate parent layout entirely; `+page@(marketing).svelte` re-anchors at the `(marketing)` group's layout. Same shape applies to `+layout@.svelte` for re-rooting a nested layout. Cross-reference Ch 37.2 for the worked example; this cheat sheet exists so you can spot it in the wild without flipping back.

---

## Lesson 37.5 — Now you write it

> **URL sweep callout — read before you start.** Earlier chapters used `/` as the home/dashboard URL. Once you do this exercise, the marketing landing page owns `/` and the app dashboard moves to `/dashboard`. **If you're moving to Ch 37's split, swap `'/'` for `/dashboard` everywhere a previous chapter wrote it as the app's home.** Concretely:
>
> - Ch 33's `<a href="/">← Back</a>` on the habit detail page → `<a href="/dashboard">← Back</a>`.
> - Ch 35.1's `class:hidden={page.url.pathname === '/dashboard'}` is already correct in this chapter; if you're cross-referencing an older draft that says `=== '/'`, update it.
> - Ch 36.1's `await goto('/dashboard')` after delete is already correct; older drafts that said `goto('/')` should be updated.
> - The `<a href="/" class="logo">` in `+layout.svelte` (Ch 32) stays as `/` — the logo points to the marketing landing, not the dashboard.
>
> Anything else inside earlier files stays untouched (those snippets predate the split). The sweep only matters from here forward.

**The English sentence first:**

> *"Move the home page into `(app)/+page.svelte`. Create `(marketing)/+page.svelte` with simple landing-page content. Create `(marketing)/+layout.svelte` with the marketing nav."*

<details>
<summary>Worked answer</summary>

1. Create `src/routes/(app)/` and move the *contents* of `src/routes/+page.svelte` and `+page.ts` into it as `(app)/+page.svelte` and `(app)/+page.ts`. The home page URL stays `/` because `(app)` is hidden in the URL.

2. Create `src/routes/(marketing)/+layout.svelte`:

```svelte
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { children }: { children: Snippet } = $props();
</script>

<header class="marketing">
  <a href="/" class="logo">Streak</a>
  <nav>
    <a href="/about">About</a>
    <a href="/pricing">Pricing</a>
    <a href="/login">Sign in</a>
  </nav>
</header>

<main>{@render children()}</main>

<style>
  .marketing { display: flex; gap: 1rem; padding: 1rem; background: white; border-bottom: 1px solid #eee; }
  .marketing .logo { font-weight: 700; text-decoration: none; }
  main { padding: 2rem; max-width: 720px; margin: 0 auto; }
</style>
```

3. Create `src/routes/(marketing)/+page.svelte` (becomes `/`):

```svelte
<h1>Streak — habits that stick.</h1>
<p>Track what matters. Pro tier coming soon.</p>
<a href="/signup">Get started</a>
```

4. Create `(marketing)/about/+page.svelte` and `(marketing)/pricing/+page.svelte` with placeholder content.

But wait — `(marketing)/+page.svelte` becomes `/`, and `(app)/+page.svelte` *also* becomes `/`. Conflict. **Pick one** — the marketing landing for unauthenticated users, the app dashboard for authenticated users. We'll route between them via auth in Part VII; for now, move the home dashboard to `(app)/dashboard/+page.svelte` (URL `/dashboard`) and let `(marketing)/+page.svelte` own `/`.

Final tree:

```
src/routes/
  +layout.svelte           # base nav
  +error.svelte
  (marketing)/
    +layout.svelte
    +page.svelte           # /
    about/+page.svelte
    pricing/+page.svelte
  (app)/
    +layout.svelte
    dashboard/+page.svelte # /dashboard
    habits/[id]/+page.ts + +page.svelte
```

</details>

---

## Lesson 37.6 — Recurring concepts from earlier chapters

- **`+layout.svelte` and the children snippet** (Ch 32) — applied per-group.
- **Folder-as-URL convention** (Ch 1) — `(group)` folders break the convention without breaking the URL.

---

## Lesson 37.7 — What you can now read in the wild

After Chapter 37 you can:

- Read a SvelteKit project's `src/routes/` tree and map each URL to its file.
- Spot **`(group)` folders** and explain what they do.
- Read **`+page@.svelte`** layout-reset syntax.
- Explain server vs client execution for each `+`-file out loud.

---

## End-of-chapter checkpoint

- [ ] Marketing and app are visually distinct sections.
- [ ] You can map any URL in your app to its file.
- [ ] You can read each `+`-file's purpose aloud.
- [ ] Part V is done.

End of Part V. Next: real persistence.
