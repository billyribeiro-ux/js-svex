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

```svelte
<!-- +layout.svelte -->
<script lang="ts">
  import { setContext } from 'svelte';
  import { HabitStore } from '$lib/habits.svelte';

  const store = new HabitStore();
  setContext('habit-store', store);
</script>
```

```svelte
<!-- any descendant page (e.g. +page.svelte) -->
<script lang="ts">
  import { getContext } from 'svelte';
  import type { HabitStore } from '$lib/habits.svelte';

  const store = getContext('habit-store') as HabitStore;
</script>
```

Each user navigating their own browser tab has their own component tree, hence their own `HabitStore`. No SSR-singleton landmine.

> **`setContext(key, value)` / `getContext(key)`** — Svelte primitives for sharing values down a component tree without prop-drilling. The key can be a string or (preferred for big apps) a `Symbol`.

A small typing note: `getContext` is typed `<T>(key) => T`, but it doesn't actually verify the key was set with that type. The `as HabitStore` cast above is a *trust* — if the consumer's key doesn't match what was set, you get a runtime undefined. For type safety in a real app, wrap context calls in a typed helper:

```ts
// src/lib/contexts.ts
import { setContext, getContext } from 'svelte';
import type { HabitStore } from '$lib/habits.svelte';

const HABIT_STORE_KEY = Symbol('habit-store');

export function setHabitStore(store: HabitStore): void {
  setContext(HABIT_STORE_KEY, store);
}
export function getHabitStore(): HabitStore {
  const store = getContext<HabitStore | undefined>(HABIT_STORE_KEY);
  if (store === undefined) throw new Error('HabitStore not in context');
  return store;
}
```

Use `setHabitStore(store)` in the layout, `const store = getHabitStore()` in pages. The `Symbol` makes the key globally unique; the wrapper enforces presence.

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
import type { Habit } from '$lib/types';
import { habitId } from '$lib/types';
import { error } from '@sveltejs/kit';

export const load: PageLoad = async ({ params }) => {
  // For now, fake data. Real DB in Part VI.
  const fakeHabits: Habit[] = [
    { id: habitId('demo-1'), name: 'Drink water', createdAt: Date.now() },
    { id: habitId('demo-2'), name: 'Read 20 minutes', createdAt: Date.now() },
  ];
  const habit = fakeHabits.find((h) => h.id === params.id);
  if (habit === undefined) throw error(404, { message: 'Habit not found' });
  return { habit };
};
```

> **`error(status, message)`** is documented to be used as `throw error(...)`. The helper does throw internally (its return type is `never`), so a bare `error(...)` works at runtime — but writing `throw` makes the control-flow visible in the source. Senior habit, and consistent with `throw redirect(...)`. We use `throw` everywhere.

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

> **`error(status, message)`** — `@sveltejs/kit` helper that throws a typed error. Caught by `+error.svelte`.

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

---

## Lesson 34.4 — Recurring concepts from earlier chapters

- **`error()` helper** (Ch 33) — same call, now visualised by `+error.svelte`.
- **`?.` and `??`** (Ch 4) — `page.error?.message ?? 'Something went wrong.'`.
- **Layout cascade** (Ch 32) — `+error.svelte` renders inside the parent layout.

---

## Lesson 34.5 — What you can now read in the wild

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

<a href="/" class="logo" class:hidden={page.url.pathname === '/'}>Streak</a>
<small>{page.url.pathname}</small>
{#if navigating}<small>navigating…</small>{/if}

<style>
  .hidden { display: none; }
</style>
```

`page` and `navigating` are *reactive runes* — when the URL changes, anything reading `page.url.pathname` re-renders.

> **`$app/state`** — Svelte 5's runes-style replacement for the legacy `$app/stores`. Members: `page`, `navigating`, `updated`.

---

## Lesson 35.2 — `aria-current`

For nav links:

```svelte
<a href="/stats" aria-current={page.url.pathname === '/stats' ? 'page' : undefined}>Stats</a>
```

Screen readers announce *"current page"* on the active link.

---

## Lesson 35.3 — Recurring concepts from earlier chapters

- **`class:foo={cond}`** (Ch 5) — used here for `class:hidden`.
- **`{#if navigating}`** — derived rendering on the navigation state rune.

---

## Lesson 35.4 — What you can now read in the wild

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
  await goto('/');
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
    if (isDirty && !window.confirm('Discard unsaved changes?')) cancel();
  });
</script>
```

---

## Lesson 36.4 — Recurring concepts from earlier chapters

- **`async`/`await`** (Ch 22) — `goto` and `invalidate` return Promises.
- **`window.confirm`** (Ch 15) — placeholder dialog; we'll replace with `<ConfirmDialog>` in Ch 53.
- **`depends('app:foo')`** — declares a custom invalidation key.

---

## Lesson 36.5 — What you can now read in the wild

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
+page.svelte         page UI
+page.ts             universal load (no DB/secrets)
+page.server.ts      server-only load + actions (DB, secrets, cookies)
+layout.svelte       layout UI
+layout.ts           universal layout load
+layout.server.ts    server-only layout load (auth gating)
+server.ts           REST endpoint (GET/POST/...)
+error.svelte        error boundary
+hooks.server.ts     handle/handleError/handleFetch
+hooks.client.ts     handleError on the client
+hooks.ts            reroute, transport (universal)
```

Print this. Stick it above your monitor.

---

## Lesson 37.5 — Now you write it

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
