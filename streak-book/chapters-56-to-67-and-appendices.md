# Part IX — Shipping it, operating it, the principal-engineer mindset

> *"By Chapter 67, Streak is live, observable, tested at every level, and the reader has shipped a feature alone from a one-paragraph brief and written its post-mortem."*

---

# Chapter 56 — Error boundaries, `handleError`, the failure budget

> *Today's job:* an unexpected error from anywhere in the app — server, client, root layout, deep child component — produces a user-facing apology page with an error ID, a structured log entry the on-call engineer can grep, and PII redacted before anything leaves the process. *Visible win:* trigger a deliberate `throw new Error('boom')` from inside a `load`; see the polite page; copy the error ID; grep it out of the log file.

You've already met `error()` for *expected* errors (Chapter 33–34). This chapter is the second half of the story: *unexpected* errors. The ones a programmer mistake, a bad cast, or a flaky third-party API caused. They get caught by `handleError` and rendered through `+error.svelte`, but with a **generic** message — never the raw stack trace, never the database error string with PII inside it.

---

## Lesson 56.1 — `handleError` on the server

```ts
// src/hooks.server.ts
import type { HandleServerError } from '@sveltejs/kit';
import { logger } from '$lib/logger';

export const handleError: HandleServerError = ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID();
  logger.error('unhandled', {
    errorId,
    status,
    message,
    method: event.request.method,
    path: event.url.pathname,
    userId: event.locals.user?.id ?? null,
    error: redact(error),
  });
  return { message: 'An unexpected error occurred.', code: errorId };
};

function redact(err: unknown): { name: string; message: string; stack?: string } | { value: string } {
  if (err instanceof Error) {
    return { name: err.name, message: err.message, stack: err.stack };
  }
  return { value: String(err) };
}
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `export const handleError: HandleServerError = ({ error, event, status, message }) => { ... }` | *"Export the server-side error handler. It receives the error, the event, the status code, and a generic message."* |
| `const errorId = crypto.randomUUID();` | *"Mint a fresh error ID for this incident."* |
| `logger.error('unhandled', { ... });` | *"Log the full context — the ID, the route, the user, the redacted error."* |
| `return { message: '...', code: errorId };` | *"Hand the user a generic message plus the ID they can quote."* |
| `function redact(err: unknown): ... { ... }` | *"Reduce the error to a JSON-serialisable shape with name + message + stack; PII fields are deliberately omitted."* |

The `logger` is the structured-pino logger we'll formally build in Chapter 63 — for now treat it as `console.log` for JSON.

> **`handleError`** *(noun)* — SvelteKit's hook for *unexpected* errors. Returns an `App.Error` shape that flows into `page.error`. Must never throw.
>
> **error ID** — a UUID minted per incident, returned to the user, written to the log. It's the bridge between *"something broke"* and *"here's exactly which something."*
>
> **redaction** — stripping fields a logger shouldn't see. Bible rule #15.

---

## Lesson 56.2 — `handleError` on the client

```ts
// src/hooks.client.ts
import type { HandleClientError } from '@sveltejs/kit';

export const handleError: HandleClientError = ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID();
  // In the browser, ship to a real telemetry service in production (Sentry, Logflare, etc.).
  // For now we console.error with structured shape.
  console.error('client.unhandled', {
    errorId,
    status,
    message,
    path: event.url.pathname,
    error: error instanceof Error ? { name: error.name, message: error.message } : { value: String(error) },
  });
  return { message: 'An unexpected error occurred.', code: errorId };
};
```

Same contract as the server side: don't throw, don't leak PII, return a generic `App.Error`. The client cannot redact anything that's already in the user's browser memory — but it *can* refuse to ship sensitive payloads to your telemetry vendor.

---

## Lesson 56.3 — Error IDs surfaced to the user

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<div class="error-card">
  <h2>Something went wrong</h2>
  <p>{page.error?.message ?? 'Please try again.'}</p>
  {#if page.error?.code}
    <p class="error-id">Error ID: <code>{page.error.code}</code></p>
    <p class="hint">Quote this ID if you contact support.</p>
  {/if}
  <a href="/">Go home</a>
</div>

<style>
  .error-card { padding: 2rem; text-align: center; max-width: 480px; margin: 4rem auto; }
  .error-id { color: #666; font-size: 0.875rem; }
  .hint { color: #888; font-size: 0.8125rem; margin-top: 0.5rem; }
</style>
```

When the user emails support: *"Error ID `f3a1...` — what happened?"* — you grep the logs for `f3a1` and find the full incident in seconds. That's the contract.

---

## Lesson 56.4 — `<svelte:boundary>` for granular catching

The route-level `+error.svelte` catches errors from `load`. Errors inside *render* — a child component throwing during rendering — bubble up through the component tree. To catch them locally and keep the rest of the page alive, use `<svelte:boundary>`:

```svelte
<svelte:boundary>
  <RiskyChartWidget data={chart} />
  {#snippet failed(error: unknown, reset: () => void)}
    <div class="boundary-fail">
      <p>This chart couldn't load.</p>
      <button type="button" onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>
```

Read aloud: *"render the chart; if it throws during render, swap in the failed snippet, which lets the user retry."*

The rest of the page (header, nav, other widgets) keeps working. This is the *graceful-degradation* pattern.

> **`<svelte:boundary>`** — Svelte 5 element that catches errors thrown during render of its children. Has a `failed(error, reset)` snippet for the fallback UI.

---

## Lesson 56.5 — Where `+error.svelte` does *not* fire

A senior knows the gaps:

- **Errors in `+server.ts`** — return a JSON error response or fall through to `src/error.html`. `+error.svelte` is for page renders, not API endpoints.
- **Errors in the root `+layout.svelte` itself** (or its `load`) — there's no parent to mount `+error.svelte` into. SvelteKit falls back to `src/error.html`.
- **Errors in the `handle` hook** — same fallback to `src/error.html`.

For these, customise `src/error.html`:

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="utf-8" />
    <title>%sveltekit.error.message%</title>
    <style>
      body { font-family: system-ui, sans-serif; max-width: 480px; margin: 4rem auto; padding: 0 1rem; text-align: center; }
    </style>
  </head>
  <body>
    <h1>%sveltekit.status%</h1>
    <p>%sveltekit.error.message%</p>
  </body>
</html>
```

`%sveltekit.status%` and `%sveltekit.error.message%` are placeholders SvelteKit fills.

---

## Lesson 56.6 — `App.Error` shape

We declared `App.Error` already in Chapter 34 with `{ message: string; code?: string }`. The `code` is now the error ID `handleError` returns. If you want richer fields (e.g. a category for downstream filtering), add them:

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Error {
      message: string;
      code?: string;
      category?: 'auth' | 'billing' | 'database' | 'unknown';
    }
  }
}
export {};
```

Don't add fields the user shouldn't see (no `userId`, no `stack`, no internal IDs).

---

## Lesson 56.7 — Build, break, fix

Add a deliberately-broken `load`:

```ts
// src/routes/(app)/dashboard/+page.server.ts (temporarily)
export const load: PageServerLoad = async () => {
  throw new Error('deliberate boom');
};
```

Save. Visit `/dashboard`. You see the polite *"Something went wrong"* page with an error ID. Open `dev.log` (or wherever `pino` writes); grep the ID; find the structured `unhandled` line. Confirm `error.message` is *"deliberate boom"*, but the user only saw the generic message. **Bible rule #15 in action: PII never leaves the boundary; the operator gets the truth.**

Restore the load function.

---

## Lesson 56.8 — Failure budgets, named

A **failure budget** is the maximum acceptable error rate over a time window. If your budget is *0.1% of requests can fail per month*, and you have 1M requests, you can tolerate 1000 failures before declaring an incident. This is the SRE concept. Streak isn't at scale yet, but the vocabulary lands here so the observability chapter (Ch 63) can build on it.

> **failure budget** *(noun)* — the threshold of failures you tolerate without action. Burn it, and you stop shipping features and fix reliability.

---

## Lesson 56.9 — Read this code

Two snippets.

### Snippet A — a leaky `handleError`

```ts
export const handleError: HandleServerError = ({ error }) => {
  console.error(error);
  return { message: error instanceof Error ? error.message : 'unknown' };
};
```

What's wrong?

<details>
<summary>Answer</summary>

Three things, all violations of Bible rule #15:

1. `console.error(error)` ships the entire error including stack traces (which can contain DB connection strings, file paths, sometimes user data) to the log without redaction.
2. The `message` is returned verbatim to the user — leaking internal error text to anyone who triggers a 500.
3. No error ID — the user has nothing to quote when they call support; the operator has no way to find this exact incident.

A senior would reject this in code review.
</details>

### Snippet B — `<svelte:boundary>` placement

```svelte
<svelte:boundary>
  <Header />
  <main>
    <RiskyWidget />
    <SafeWidget />
  </main>
  {#snippet failed()}<p>Something broke.</p>{/snippet}
</svelte:boundary>
```

If `RiskyWidget` throws, what does the user see?

<details>
<summary>Answer</summary>

*"Something broke."* — and the entire `<Header />` and `<main>` are gone. The boundary catches everything inside it, not just `RiskyWidget`.

The fix: place the boundary *around just the risky thing*:

```svelte
<Header />
<main>
  <svelte:boundary>
    <RiskyWidget />
    {#snippet failed()}<p>This widget broke.</p>{/snippet}
  </svelte:boundary>
  <SafeWidget />
</main>
```

Now `RiskyWidget` failing leaves the header and `SafeWidget` working. *Granular* boundaries are the senior pattern.
</details>

---

## Lesson 56.10 — Now you write it

**The English sentence first:**

> *"Wrap the habit list on the dashboard in a `<svelte:boundary>` so that one bad habit (a future bug rendering a corrupted habit) doesn't kill the rest of the page. The fallback should say 'Couldn't load habits — try refreshing.' with a retry button."*

<details>
<summary>Worked answer</summary>

```svelte
<!-- src/routes/(app)/dashboard/+page.svelte -->
<svelte:boundary>
  <ul class="habit-list">
    {#each visibleHabits as habit (habit.id)}
      <HabitRow {habit} onDelete={removeHabit} />
    {/each}
  </ul>

  {#snippet failed(error: unknown, reset: () => void)}
    <div class="boundary-fail">
      <p>Couldn't load habits — try refreshing.</p>
      <button type="button" onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>
```

Test: temporarily make `HabitRow` throw on a specific habit ID; confirm the rest of the page (nav, search, add-form) still works.
</details>

---

## Lesson 56.11 — Recurring concepts from earlier chapters

- **`+error.svelte`** (Ch 34) — route-level boundary for `load` errors.
- **`error()` helper** (Ch 33) — *expected* errors; complement to `handleError`'s *unexpected* errors.
- **`crypto.randomUUID()`** (Ch 9) — fresh ID per incident.
- **`page.error`, `page.status`** (Ch 35) — read inside `+error.svelte`.
- **Bible rule #15** — PII never logged; redaction at the boundary.

---

## Lesson 56.12 — What you can now read in the wild

After Chapter 56 you can:

- Read **`HandleServerError` / `HandleClientError`** function shapes.
- Read **`<svelte:boundary>` + `{#snippet failed(error, reset)}`** as the granular catcher.
- Spot **leaky error returns** that ship raw `error.message` to the user.
- Distinguish **expected** (`error()`) from **unexpected** (`handleError`) errors and the routing for each.
- Tell where `+error.svelte` does *not* render and reach for `src/error.html`.

---

## Glossary added in Chapter 56

| Term | Definition |
|---|---|
| `handleError` | Hook for unexpected errors; never throws. |
| error ID | UUID minted per incident, returned to user + logged. |
| redaction | Stripping PII before logging. |
| `<svelte:boundary>` | Svelte 5 element that catches render-time errors locally. |
| `failed` snippet | The fallback UI inside a boundary. |
| failure budget | Maximum tolerable error rate over a window. |

---

## End-of-chapter checkpoint

- [ ] You triggered a deliberate `throw` and saw the polite page with an error ID.
- [ ] You found the structured log line by grepping the ID.
- [ ] You wrapped the habit list in a `<svelte:boundary>`.
- [ ] `+error.svelte` renders inside the layout chrome.
- [ ] You can articulate why `error.message` should never be returned verbatim to the user.

---

# Chapter 57 — Environment variables — the four-quadrant matrix

> *Today's job:* every secret lives in the right quadrant; the build refuses to start with a malformed env var; no secret is ever bundled into client code. *Visible win:* you delete a value from `.env`, run `pnpm build`, and the build halts with a clear *"DATABASE_URL is required"* before any code runs.

A senior engineer can recite the four-quadrant matrix from memory. By the end of this chapter, so can you.

---

## Lesson 57.1 — The four quadrants

| | Static (build-time) | Dynamic (runtime) |
|---|---|---|
| **Private** | `$env/static/private` | `$env/dynamic/private` |
| **Public** | `$env/static/public` (`PUBLIC_*`) | `$env/dynamic/public` (`PUBLIC_*`) |

Two axes:

- **Static vs dynamic** — *static* values are baked into the build at `pnpm build` time. *Dynamic* values are read at runtime each time the server starts (so you can change `.env` between deploys without rebuilding).
- **Private vs public** — *private* values must never reach the browser. *Public* values are bundled into client code and visible to anyone who opens dev-tools.

What goes where:

| Quadrant | Examples |
|---|---|
| Static private | `DATABASE_URL`, `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `RESEND_API_KEY` |
| Static public | `PUBLIC_SITE_URL`, `PUBLIC_STRIPE_PUBLISHABLE_KEY`, `PUBLIC_GA_ID` |
| Dynamic private | Per-deploy secrets that rotate (e.g. SSO client secrets that change without rebuilds) |
| Dynamic public | Feature flags read fresh on each request (rare; usually use a flag service) |

> **`PUBLIC_` prefix** — SvelteKit's signal that a value is public. Anything *without* the prefix in `.env` is treated as private. The prefix is configurable via `kit.env.publicPrefix` in `svelte.config.ts`; we use the default.

---

## Lesson 57.2 — Imports per quadrant

```ts
// Static private — server-only
import { DATABASE_URL, STRIPE_SECRET_KEY } from '$env/static/private';

// Static public — anywhere (server or client)
import { PUBLIC_SITE_URL } from '$env/static/public';

// Dynamic private — server-only, read at runtime
import { env } from '$env/dynamic/private';
const dbUrl = env.DATABASE_URL;

// Dynamic public — anywhere
import { env as publicEnv } from '$env/dynamic/public';
const siteUrl = publicEnv.PUBLIC_SITE_URL;
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `import { DATABASE_URL } from '$env/static/private';` | *"Import the DATABASE_URL from the static-private quadrant — baked at build, server-only."* |
| `import { PUBLIC_SITE_URL } from '$env/static/public';` | *"Import the public site URL — baked at build, fine for the browser."* |
| `import { env } from '$env/dynamic/private';` | *"Import the dynamic-private bag — read at runtime, server-only."* |

Try this and watch the build fail:

```ts
// src/routes/(marketing)/+page.svelte (deliberately wrong)
<script lang="ts">
  import { DATABASE_URL } from '$env/static/private'; // ❌
</script>
<p>{DATABASE_URL}</p>
```

`pnpm build` halts with *"Cannot import `$env/static/private` into client-side code."* Bible rule #19, applied by the framework. **The reader sees the error deliberately.**

---

## Lesson 57.3 — Boot validator

The matrix tells you *which import to use*; the validator tells you *whether the values are sane*. Add `src/lib/env.ts`:

```ts
// src/lib/env.ts
import * as v from 'valibot';
import {
  DATABASE_URL,
  STRIPE_SECRET_KEY,
  STRIPE_WEBHOOK_SECRET,
  RESEND_API_KEY,
  R2_ACCOUNT_ID,
  R2_ACCESS_KEY_ID,
  R2_SECRET_ACCESS_KEY,
  R2_BUCKET,
} from '$env/static/private';

const Schema = v.object({
  DATABASE_URL: v.pipe(v.string(), v.url()),
  STRIPE_SECRET_KEY: v.pipe(v.string(), v.regex(/^sk_(test|live)_/, 'Stripe secret must start with sk_test_ or sk_live_')),
  STRIPE_WEBHOOK_SECRET: v.pipe(v.string(), v.regex(/^whsec_/)),
  RESEND_API_KEY: v.pipe(v.string(), v.regex(/^re_/)),
  R2_ACCOUNT_ID: v.pipe(v.string(), v.minLength(1)),
  R2_ACCESS_KEY_ID: v.pipe(v.string(), v.minLength(1)),
  R2_SECRET_ACCESS_KEY: v.pipe(v.string(), v.minLength(1)),
  R2_BUCKET: v.pipe(v.string(), v.minLength(1)),
});

export const env = v.parse(Schema, {
  DATABASE_URL,
  STRIPE_SECRET_KEY,
  STRIPE_WEBHOOK_SECRET,
  RESEND_API_KEY,
  R2_ACCOUNT_ID,
  R2_ACCESS_KEY_ID,
  R2_SECRET_ACCESS_KEY,
  R2_BUCKET,
});
```

Then import this module from `src/hooks.server.ts`:

```ts
// src/hooks.server.ts
import { env } from '$lib/env'; // forces the validator to run on first request
```

If any value is missing or malformed, `v.parse` throws a useful error and the server fails to boot. **Fail-fast is the senior pattern**: crash now with a clear message; don't crash three hours later when the first user hits the route that needs that variable.

---

## Lesson 57.4 — `.env.example`

Commit `.env.example` (no real secrets) so collaborators know what to set:

```bash
# .env.example
DATABASE_URL=postgres://postgres:dev@localhost:5432/streak_dev
STRIPE_SECRET_KEY=sk_test_xxx
STRIPE_WEBHOOK_SECRET=whsec_xxx
RESEND_API_KEY=re_xxx
R2_ACCOUNT_ID=xxx
R2_ACCESS_KEY_ID=xxx
R2_SECRET_ACCESS_KEY=xxx
R2_BUCKET=streak-uploads-dev

PUBLIC_SITE_URL=http://localhost:5173
PUBLIC_STRIPE_PUBLISHABLE_KEY=pk_test_xxx
```

`.gitignore` excludes `.env`; `.env.example` is checked in.

---

## Lesson 57.5 — Secret rotation, briefly

When a secret leaks (a key in a screenshot, a stale repo branch pushed to the wrong remote), you need a runbook — a short document that says *exactly what steps to run*. Create `docs/runbooks/secret-rotation.md`:

```markdown
# Secret rotation

When a secret has leaked or expired, follow these steps in order.

## Database
1. In Neon (or Supabase), generate a new connection string.
2. Update `DATABASE_URL` in Vercel project settings (production + preview).
3. Trigger a new deploy.
4. Once green, revoke the old credentials.

## Stripe
1. Stripe dashboard → Developers → API keys → Roll restricted key.
2. Update `STRIPE_SECRET_KEY` in Vercel.
3. Redeploy. Verify checkout flow.
4. Roll the webhook signing secret in Stripe → Webhooks → endpoint.
5. Update `STRIPE_WEBHOOK_SECRET` in Vercel.
6. Replay the last hour of events to confirm signature verification works.

## R2 / S3
1. Create a new key pair in Cloudflare R2.
2. Update `R2_ACCESS_KEY_ID` + `R2_SECRET_ACCESS_KEY` in Vercel.
3. Redeploy. Verify uploads work.
4. Delete the old key pair.
```

Treat the runbook as code: when you change a secret, update the runbook in the same PR.

---

## Lesson 57.6 — Read this code

```ts
import { DATABASE_URL } from '$env/static/private';
const url = DATABASE_URL ?? 'postgres://localhost/dev';
```

What's the issue?

<details>
<summary>Answer</summary>

`$env/static/private` imports are statically guaranteed to exist (or the build fails); the `?? 'postgres://localhost/dev'` fallback is dead code. Worse: if the developer thinks the fallback runs, they might forget to set the var, and the build will fail at `pnpm build` time without a friendly message.

The right pattern: validate at boot (Lesson 57.3) and let the import be authoritative.
</details>

---

## Lesson 57.7 — Now you write it

**The English sentence first:**

> *"Add a new env var `SENTRY_DSN` (private, runtime — it might rotate without rebuild). Wire it through `$env/dynamic/private` and validate it at boot via the env module."*

<details>
<summary>Worked answer</summary>

In `src/lib/env.ts`, *separate* dynamic from static — dynamic values aren't tree-shakable so they live alongside but use a different import:

```ts
import { env as dynPrivate } from '$env/dynamic/private';

const DynamicSchema = v.object({
  SENTRY_DSN: v.optional(v.pipe(v.string(), v.url())),
});

export const dynEnv = v.parse(DynamicSchema, {
  SENTRY_DSN: dynPrivate.SENTRY_DSN,
});
```

`SENTRY_DSN` is `optional` because dev environments don't ship to Sentry. In production, the deploy fails to boot if it's malformed. Add to `.env.example` (commented):

```
# Optional — set in production only
# SENTRY_DSN=https://xxx@sentry.io/xxx
```
</details>

---

## Lesson 57.8 — Recurring concepts from earlier chapters

- **Boundary parsing with Valibot** (Ch 41, 51) — env vars are *another* untrusted input.
- **Fail-fast** — crash on boot, not at first request.
- **Bible rule #19** — server-only secrets via `$env/static/private`; build error on misuse.

---

## Lesson 57.9 — What you can now read in the wild

After Chapter 57 you can:

- Read **`$env/static/private` / `$env/static/public` / `$env/dynamic/private` / `$env/dynamic/public`** and pick the right one.
- Recognise **`PUBLIC_*` prefix** as the marker for browser-exposed values.
- Read a **boot validator** built on Valibot.
- Spot a **dead `??` fallback** on a guaranteed-static import.
- Recognise a **secret-rotation runbook** as a senior artifact.

---

## Glossary added in Chapter 57

| Term | Definition |
|---|---|
| four-quadrant matrix | Static-vs-dynamic × private-vs-public for env vars. |
| `PUBLIC_*` prefix | SvelteKit's signal that a value is browser-safe. |
| boot validator | A module that throws on malformed env at startup. |
| fail-fast | Crash early with a clear message rather than late with a confusing one. |
| runbook | A document of exact steps for an operational task. |

---

## End-of-chapter checkpoint

- [ ] All env vars are typed and validated at boot.
- [ ] `.env.example` documents every required var.
- [ ] You triggered a deliberate import from `$env/static/private` inside a `.svelte` and saw the build fail.
- [ ] `docs/runbooks/secret-rotation.md` exists.

---

# Chapter 58 — Vitest unit tests, property-based tests, the test pyramid

> *Today's job:* `pnpm test:unit` runs in under two seconds, exercises every pure helper in `$lib/`, and includes a property-based test for `Money` that fires a thousand random inputs at `splitCents` and proves the sum-of-parts always equals the total. *Visible win:* the test suite is green; you deliberately break `splitCents` (off-by-one) and watch property-based testing catch it before any human notices.

---

## Lesson 58.1 — The test pyramid

The senior framework for thinking about tests:

```
                /\
               /  \   e2e (Playwright)
              /----\  → fewest, slowest, most realistic
             /      \
            /--------\  integration (real DB, real services)
           /          \  → some, medium speed, medium realism
          /------------\
         /              \  unit (pure functions, in isolation)
        /----------------\  → many, fast, narrow
```

**Rules of thumb:**
- Cover *every pure function* with unit tests. They're cheap.
- Cover *every database mutation* with an integration test (Ch 59). They're not cheap, but they're necessary.
- Cover *the happy path through each user-visible flow* with an e2e test (Ch 60). They're expensive; pick the journeys that matter.

**Don't invert the pyramid.** A suite that's mostly e2e is slow, flaky, and hard to debug. A suite that's mostly unit catches the most bugs per second of test runtime.

> **test pyramid** *(noun)* — the canonical ratio: lots of unit, fewer integration, fewest e2e.

---

## Lesson 58.2 — Vitest setup

`sv create` already added Vitest. Confirm in `package.json`:

```json
{
  "scripts": {
    "test:unit": "vitest run --dir tests/unit --dir src",
    "test:unit:watch": "vitest --dir tests/unit --dir src",
    "test:coverage": "vitest run --coverage"
  }
}
```

`vitest.config.ts` (or merged into `vite.config.ts`):

```ts
import { defineConfig } from 'vitest/config';
import { sveltekit } from '@sveltejs/kit/vite';

export default defineConfig({
  plugins: [sveltekit()],
  test: {
    environment: 'node', // pure functions test in node; DOM tests use jsdom (Ch 59)
    include: ['tests/unit/**/*.{test,spec}.ts', 'src/**/*.{test,spec}.ts'],
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      include: ['src/lib/**/*.ts'],
      exclude: ['src/lib/**/*.test.ts'],
    },
  },
});
```

> **`vitest`** — Vite-native test runner. Same config as your dev server; same TypeScript pipeline; instant.
>
> **`.test.ts` vs `.svelte.test.ts`** — plain `.test.ts` for pure-function tests; `.svelte.test.ts` for tests that use runes (`$state`, `$derived`). The latter enables the Svelte compiler in test files.

---

## Lesson 58.3 — Anatomy of a unit test

```ts
// tests/unit/parseHabit.test.ts
import { describe, it, expect } from 'vitest';
import { parseHabit } from '$lib/parseHabit';

describe('parseHabit', () => {
  it('rejects null', () => {
    const r = parseHabit(null);
    expect(r.ok).toBe(false);
  });

  it('rejects empty name', () => {
    const r = parseHabit({ id: 'x', name: '', createdAt: 1 });
    expect(r.ok).toBe(false);
  });

  it('rejects missing createdAt', () => {
    const r = parseHabit({ id: 'x', name: 'Read' });
    expect(r.ok).toBe(false);
  });

  it('accepts a valid habit', () => {
    const r = parseHabit({ id: 'x', name: 'Read', createdAt: 1700000000000 });
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect(r.value.name).toBe('Read');
      expect(r.value.createdAt).toBe(1700000000000);
    }
  });

  it('strips unknown fields', () => {
    const r = parseHabit({ id: 'x', name: 'Read', createdAt: 1, evil: 'payload' });
    expect(r.ok).toBe(true);
    if (r.ok) {
      expect('evil' in r.value).toBe(false);
    }
  });
});
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `describe('parseHabit', () => { ... });` | *"Group these tests under the name parseHabit."* |
| `it('rejects null', ...)` | *"This test asserts: parsing null returns a failure."* |
| `expect(r.ok).toBe(false);` | *"Expect r.ok to be exactly false."* |
| `if (r.ok) { ... }` | *"Narrow the type so we can read .value safely."* |

> **Arrange / Act / Assert** — the senior shape of a unit test. Arrange inputs, call the function, assert on the output. No mocking, no timing, no shared state.

---

## Lesson 58.4 — Property-based testing with `fast-check`

Example-based tests check specific cases. *Property-based* tests check **claims** — *"for all valid inputs, this property holds."* `fast-check` generates a thousand random inputs that try to break the claim.

```bash
pnpm add -D fast-check
```

```ts
// tests/unit/money.test.ts
import { describe, it, expect } from 'vitest';
import * as fc from 'fast-check';
import { splitCents, cents, formatCents, applyBps } from '$lib/money';

describe('splitCents', () => {
  it('the sum of parts always equals the input total', () => {
    fc.assert(fc.property(
      fc.integer({ min: 0, max: 1_000_000_000 }),
      fc.integer({ min: 1, max: 100 }),
      (total, n) => {
        const parts = splitCents(cents(total), n);
        const sum = parts.reduce<number>((a, b) => a + (b as number), 0);
        return sum === total;
      },
    ));
  });

  it('every part is non-negative', () => {
    fc.assert(fc.property(
      fc.integer({ min: 0, max: 1_000_000_000 }),
      fc.integer({ min: 1, max: 100 }),
      (total, n) => {
        const parts = splitCents(cents(total), n);
        return parts.every((p) => (p as number) >= 0);
      },
    ));
  });

  it('the largest and smallest part differ by at most 1', () => {
    fc.assert(fc.property(
      fc.integer({ min: 1, max: 1_000_000_000 }),
      fc.integer({ min: 1, max: 100 }),
      (total, n) => {
        const parts = splitCents(cents(total), n).map((p) => p as number);
        return Math.max(...parts) - Math.min(...parts) <= 1;
      },
    ));
  });
});

describe('applyBps', () => {
  it('0 bps returns 0', () => {
    fc.assert(fc.property(fc.integer({ min: 0, max: 1_000_000 }), (amount) => {
      return (applyBps(cents(amount), 0) as number) === 0;
    }));
  });

  it('10000 bps returns the amount', () => {
    fc.assert(fc.property(fc.integer({ min: 0, max: 1_000_000 }), (amount) => {
      return (applyBps(cents(amount), 10000) as number) === amount;
    }));
  });
});
```

Run `pnpm test:unit`. `fast-check` shrinks any failure to the smallest input that breaks the property — a senior debugging accelerator.

> **property-based test** *(noun)* — a test that asserts a claim holds for *all* inputs in a domain, then generates random inputs to try to falsify it.
>
> **shrinking** — when a property fails, `fast-check` simplifies the failing input toward the minimal counterexample. *"It failed at total=947, n=23"* gets shrunk to *"it failed at total=1, n=2"*.

---

## Lesson 58.5 — Fake timers

For time-dependent code (relative timestamps, debounce, throttle, expiry):

```ts
// tests/unit/formatRelativeTime.test.ts
import { describe, it, expect, beforeEach, vi } from 'vitest';
import { formatRelativeTime } from '$lib/formatRelativeTime';

describe('formatRelativeTime', () => {
  beforeEach(() => {
    vi.useFakeTimers();
    vi.setSystemTime(new Date('2026-05-05T12:00:00Z'));
  });

  it('shows "just now" for the present', () => {
    const now = Date.now();
    expect(formatRelativeTime(now)).toBe('just now');
  });

  it('shows "5 minutes ago" for 5 minutes earlier', () => {
    const fiveMinAgo = Date.now() - 5 * 60_000;
    expect(formatRelativeTime(fiveMinAgo)).toBe('5 minutes ago');
  });

  it('uses singular for exactly 1 minute', () => {
    expect(formatRelativeTime(Date.now() - 60_000)).toBe('1 minute ago');
  });
});
```

> **`vi.useFakeTimers` / `vi.setSystemTime`** — Vitest's time-control primitives. Make `Date.now()` deterministic so the test never flakes.

---

## Lesson 58.6 — Coverage, briefly

```bash
pnpm test:coverage
```

Open `coverage/index.html`. **Aim for high but not 100%** on `src/lib/`. 100% is a code smell — it usually means tests are testing implementation details, or that you've added tests for unreachable defensive branches that should be deleted.

Senior heuristic: if a function is gnarly enough that its coverage drops below 80%, either it has missing tests *or* it has too many branches and should be split.

---

## Lesson 58.7 — Read this code

```ts
it('addHabit succeeds', async () => {
  const result = await addHabit('Read', mockDb);
  expect(result.ok).toBe(true);
});
```

What's wrong?

<details>
<summary>Answer</summary>

`mockDb` — Bible rule #5: *don't mock the database*. Mocked DBs lie about transactional semantics, foreign-key behaviour, atomic UPDATEs, race conditions. The integration test against a real Postgres (Ch 59) is the right home for this.

A unit test should test a *pure* function. If `addHabit` touches the DB, push the DB-touching part down to a thin layer and unit-test the *pure logic* (validation, name normalisation, the `Result` shape).
</details>

---

## Lesson 58.8 — Now you write it

**The English sentence first:**

> *"Write property-based tests for `parseCents` from Chapter 30: round-trip a string through `parseCents` then `formatCents` and assert the integer-cents value is preserved."*

<details>
<summary>Worked answer</summary>

```ts
// tests/unit/parseCents.test.ts
import { describe, it, expect } from 'vitest';
import * as fc from 'fast-check';
import { parseCents, formatCents, cents, type Cents } from '$lib/money';

describe('parseCents', () => {
  it('round-trips integer-cent strings', () => {
    fc.assert(fc.property(
      fc.integer({ min: 0, max: 9_999_999 }),
      (n) => {
        const c = cents(n);
        const formatted = formatCents(c).replace(/[^\d.]/g, '');
        const reparsed = parseCents(formatted);
        return reparsed.ok && (reparsed.value as number) === n;
      },
    ));
  });

  it('rejects non-decimal strings', () => {
    fc.assert(fc.property(
      fc.string().filter((s) => !/^\d+(\.\d{1,2})?$/.test(s.trim())),
      (s) => {
        const r = parseCents(s);
        return !r.ok;
      },
    ));
  });
});
```

When this is green, you've proved `parseCents` and `formatCents` are inverses for every integer-cents value up to ~$100k. Property-based tests that assert *inverse pairs* catch entire bug classes that example-based tests would miss.
</details>

---

## Lesson 58.9 — Recurring concepts from earlier chapters

- **`Result<T, E>`** (Ch 27) — every fallible function tested via `result.ok`.
- **The `Money` module** (Ch 30) — built deliberately to be unit-testable without a DOM or DB.
- **Pure functions over services** (Ch 42) — `addHabitForUser` is a thin wrapper; the *parsing* is what unit tests cover.

---

## Lesson 58.10 — What you can now read in the wild

After Chapter 58 you can:

- Read **`describe / it / expect`** as the Vitest unit-test shape.
- Read **`fc.assert(fc.property(gen1, gen2, (a, b) => boolean))`** as a property-based test.
- Read **`vi.useFakeTimers()`** as the deterministic-time pattern.
- Read **`pnpm test:coverage`** output and tell which functions are under-covered.
- Spot **a mocked database in unit code** as a code-review reject.

---

## Glossary added in Chapter 58

| Term | Definition |
|---|---|
| test pyramid | Many unit, fewer integration, fewest e2e. |
| Arrange/Act/Assert | The shape of a unit test. |
| `vitest` | Vite-native test runner. |
| `fast-check` | Property-based testing library. |
| property-based test | Asserts a claim for all inputs in a domain. |
| shrinking | Reducing a failing input to the minimal counterexample. |
| fake timers | Deterministic `Date.now()` for time-dependent tests. |

---

## End-of-chapter checkpoint

- [ ] `pnpm test:unit` runs in <2s.
- [ ] `parseHabit`, `parseCents`, `formatRelativeTime`, `splitCents`, `applyBps` all have unit tests.
- [ ] At least one property-based test on `Money` is green.
- [ ] You deliberately broke `splitCents` (off-by-one) and watched a property-based test catch it.

---

# Chapter 59 — Component, contract, and integration tests

> *Today's job:* `pnpm test:component` renders `<HabitRow>` and asserts it calls `onDelete` with the right ID; `pnpm test:integration` runs `addHabitForUser` against a real Postgres with truncate-before-each isolation; `pnpm test:contract` verifies `/api/v1/habits` matches `docs/openapi.yaml`. *Visible win:* three suites, three speeds, three layers of protection.

The unit tests from Chapter 58 cover *pure* functions. The chapters above also need:
- **Component tests** — render a Svelte 5 component in jsdom; assert the rendered DOM and the callback wiring.
- **Integration tests** — exercise database mutations against a real Postgres.
- **Contract tests** — verify the public REST API matches its hand-written OpenAPI spec.

Together with the unit tests, these are the four bottom-rungs of the test pyramid.

---

## Lesson 59.1 — `@testing-library/svelte` setup

```bash
pnpm add -D @testing-library/svelte @testing-library/user-event @testing-library/jest-dom jsdom
```

Add a separate Vitest project for component tests (so the DOM environment is only active where needed):

```ts
// vitest.config.ts (extended)
import { defineConfig } from 'vitest/config';
import { sveltekit } from '@sveltejs/kit/vite';

export default defineConfig({
  plugins: [sveltekit()],
  test: {
    projects: [
      {
        extends: true,
        test: {
          name: 'unit',
          environment: 'node',
          include: ['tests/unit/**/*.test.ts', 'src/**/*.test.ts'],
        },
      },
      {
        extends: true,
        test: {
          name: 'component',
          environment: 'jsdom',
          include: ['tests/component/**/*.svelte.test.ts'],
          setupFiles: ['./tests/setup-component.ts'],
        },
      },
      {
        extends: true,
        test: {
          name: 'integration',
          environment: 'node',
          include: ['tests/integration/**/*.test.ts'],
          setupFiles: ['./tests/setup-integration.ts'],
        },
      },
    ],
  },
});
```

`tests/setup-component.ts`:

```ts
import '@testing-library/jest-dom/vitest';
```

This adds matchers like `toBeInTheDocument()` to `expect`.

> **`@testing-library/svelte`** — render Svelte components in jsdom and query the resulting DOM by accessibility role / text / label.
>
> **jsdom** — a JavaScript implementation of the DOM. Lets us run `render(MyComponent)` in Node without spinning up a real browser.

---

## Lesson 59.2 — A component test, properly

```ts
// tests/component/HabitRow.svelte.test.ts
import { describe, it, expect, vi } from 'vitest';
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import HabitRow from '$lib/components/HabitRow.svelte';
import { habitId } from '$lib/types';

describe('HabitRow', () => {
  it('renders the habit name', () => {
    render(HabitRow, {
      habit: { id: habitId('h1'), name: 'Read 20 minutes', createdAt: Date.now() },
      onDelete: () => {},
    });
    expect(screen.getByText('Read 20 minutes')).toBeInTheDocument();
  });

  it('calls onDelete with the habit id when X is clicked', async () => {
    const onDelete = vi.fn();
    render(HabitRow, {
      habit: { id: habitId('h1'), name: 'Read', createdAt: Date.now() },
      onDelete,
    });

    const button = screen.getByRole('button', { name: /remove read/i });
    await userEvent.click(button);

    expect(onDelete).toHaveBeenCalledOnce();
    expect(onDelete).toHaveBeenCalledWith(habitId('h1'));
  });

  it('hides timestamp in compact mode', () => {
    render(HabitRow, {
      habit: { id: habitId('h1'), name: 'Read', createdAt: Date.now() },
      onDelete: () => {},
      compact: true,
    });
    expect(screen.queryByText(/ago/)).not.toBeInTheDocument();
  });
});
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `render(HabitRow, { habit, onDelete })` | *"Mount HabitRow with these props in jsdom."* |
| `screen.getByText('Read 20 minutes')` | *"Find the element whose text reads 'Read 20 minutes'."* |
| `screen.getByRole('button', { name: /remove read/i })` | *"Find the button accessible-named 'Remove Read' (case-insensitive)."* |
| `await userEvent.click(button)` | *"Simulate a real click — including focus, mouseup, etc."* |
| `expect(onDelete).toHaveBeenCalledWith(habitId('h1'))` | *"The onDelete callback was invoked exactly once with the right argument."* |

Senior habit: **prefer `getByRole`** over `getByTestId`. Querying by role mirrors how a screen-reader user navigates; if your test needs a `data-testid`, your component might not be accessible.

---

## Lesson 59.3 — Tests that use runes (`.svelte.test.ts`)

When the component-under-test uses runes, the test file must end in `.svelte.test.ts` (the Svelte plugin recognises this and enables the compiler):

```ts
// tests/component/Counter.svelte.test.ts
import { test, expect } from 'vitest';
import { flushSync } from 'svelte';

test('Counter rune', () => {
  let value = $state(0);
  // … exercise reactive logic …
  value += 1;
  flushSync(); // force pending reactivity to settle synchronously
  expect(value).toBe(1);
});
```

> **`flushSync()`** — Svelte primitive that runs all pending effects synchronously. Required in tests when you need to *observe* the consequence of a state change immediately.
>
> **`$effect.root(() => { ... })`** — wraps test code in an effect scope outside a component. Use when you're testing helpers that own `$state` and `$effect` directly.

---

## Lesson 59.4 — Integration tests against a real Postgres

Bible rule #5: *don't mock the database.* The integration suite spins up a real `streak_test` database, truncates before each test, and runs the actual Drizzle queries against it.

`tests/setup-integration.ts`:

```ts
import { beforeAll, beforeEach, afterAll } from 'vitest';
import { execSync } from 'node:child_process';
import { db, closeDb } from '$lib/db/client';
import { sql } from 'drizzle-orm';

beforeAll(() => {
  // Apply migrations to the test DB
  execSync('pnpm drizzle-kit migrate', {
    env: { ...process.env, DATABASE_URL: process.env.TEST_DATABASE_URL },
    stdio: 'inherit',
  });
});

beforeEach(async () => {
  await db.execute(sql`
    TRUNCATE webhook_events, audit_log, sessions, subscriptions, habits, users
    RESTART IDENTITY CASCADE
  `);
});

afterAll(async () => {
  await closeDb();
});
```

Add to `src/lib/db/client.ts`:

```ts
import postgres from 'postgres';
import { drizzle } from 'drizzle-orm/postgres-js';
import { DATABASE_URL } from '$env/static/private';

const url = process.env.TEST_DATABASE_URL ?? DATABASE_URL;
const client = postgres(url, { max: 5, idle_timeout: 20, connect_timeout: 10 });
export const db = drizzle(client);
export const closeDb = (): Promise<void> => client.end();
```

Run with:

```bash
TEST_DATABASE_URL=postgres://postgres:dev@localhost:5432/streak_test \
  pnpm vitest run --project integration
```

Or wire a `pnpm test:integration` script that exports the env var.

> **truncate-before-each** — wipe all rows between tests for fast, full isolation. Faster than `BEGIN`/`ROLLBACK` because of how Postgres handles statement-level rollbacks; works because each test's data is independent.

---

## Lesson 59.5 — A real integration test

```ts
// tests/integration/addHabitForUser.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { db } from '$lib/db/client';
import { users, habits } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { addHabitForUser } from '$lib/habits-server';

const TEST_USER = '00000000-0000-0000-0000-000000000099';

beforeEach(async () => {
  await db.insert(users).values({
    id: TEST_USER,
    email: 'test@example.com',
    passwordHash: 'fake',
  });
});

describe('addHabitForUser', () => {
  it('inserts a habit and increments the user counter', async () => {
    const result = await addHabitForUser(TEST_USER, 'Read');
    expect(result.ok).toBe(true);

    const rows = await db.select().from(habits).where(eq(habits.userId, TEST_USER));
    expect(rows).toHaveLength(1);
    expect(rows[0]?.name).toBe('Read');

    const [user] = await db.select().from(users).where(eq(users.id, TEST_USER));
    expect(user?.habitsCount).toBe(1);
  });

  it('rejects when the cap is reached', async () => {
    await db.update(users).set({ habitsCount: 50 }).where(eq(users.id, TEST_USER));
    const result = await addHabitForUser(TEST_USER, 'Read');
    expect(result.ok).toBe(false);
    if (!result.ok) expect(result.error).toBe('limit-reached');
  });

  it('is atomic under concurrency at the limit', async () => {
    await db.update(users).set({ habitsCount: 49 }).where(eq(users.id, TEST_USER));

    const results = await Promise.all(
      Array.from({ length: 10 }, () => addHabitForUser(TEST_USER, 'concurrent')),
    );

    const successes = results.filter((r) => r.ok);
    expect(successes).toHaveLength(1);
  });
});
```

The third test is the **runtime evidence** for Bible rule #11. Ten concurrent calls; exactly one succeeds; the atomic conditional UPDATE prevents the other nine.

---

## Lesson 59.6 — Contract tests against OpenAPI

When you publish a REST API (Ch 64), you write `docs/openapi.yaml` by hand. The contract test asserts the live API matches:

```bash
pnpm add -D @stoplight/spectral-core @stoplight/spectral-rulesets ajv ajv-formats
```

```ts
// tests/contract/api-v1-habits.test.ts
import { describe, it, expect } from 'vitest';
import yaml from 'yaml';
import fs from 'node:fs/promises';
import Ajv from 'ajv';
import addFormats from 'ajv-formats';

const BASE = process.env.TEST_BASE_URL ?? 'http://localhost:4173';

describe('GET /api/v1/habits matches the OpenAPI spec', () => {
  it('list response conforms to schema', async () => {
    const specRaw = await fs.readFile('docs/openapi.yaml', 'utf-8');
    const spec = yaml.parse(specRaw) as { components: { schemas: { HabitList: object } } };

    const r = await fetch(`${BASE}/api/v1/habits`, {
      headers: { authorization: `Bearer ${process.env.TEST_PAT}` },
    });
    expect(r.status).toBe(200);
    const body: unknown = await r.json();

    const ajv = new Ajv({ strict: false });
    addFormats(ajv);
    const validate = ajv.compile(spec.components.schemas.HabitList);
    const valid = validate(body);

    expect(valid).toBe(true);
    if (!valid) console.error(validate.errors);
  });

  it('returns 401 without a token', async () => {
    const r = await fetch(`${BASE}/api/v1/habits`);
    expect(r.status).toBe(401);
  });
});
```

The test loads the spec, fetches the live endpoint, and validates the response against the schema. If the API drifts from the spec — by accident or on purpose — the test fails *and points at exactly the field that doesn't match*.

> **contract test** *(noun)* — a test that verifies an API conforms to its declared shape. Catches drift between docs and reality.

---

## Lesson 59.7 — Read this code

```ts
const result = await myAction({ name: 'Read' }, mockEvent());
expect(result.success).toBe(true);
```

Why does a senior reviewer push back?

<details>
<summary>Answer</summary>

Several reasons:

1. **`mockEvent()` lies about reality.** SvelteKit's `RequestEvent` carries cookies, locals, headers, URL, and a special `fetch`. A mock can't replicate them all. The test passes; the real handler crashes on `event.cookies.get('session')` because the mock didn't include cookies.
2. **`result.success` checks a field that doesn't exist on `fail()`** — actions return `ActionResult` shapes that aren't directly assertable like that.
3. **The right place for this test is integration.** Spin up a real preview server, post a real form, assert the rendered response.

The senior pattern: test pure logic with unit tests (the validation, the parsing); test handlers via integration (a real HTTP request hits a real server hits a real DB).
</details>

---

## Lesson 59.8 — Now you write it

**The English sentence first:**

> *"Write a component test for `<EmptyState>` (Chapter 13). It should render the heading 'No habits yet' and the prompt 'Add your first one above.'"*

<details>
<summary>Worked answer</summary>

```ts
// tests/component/EmptyState.svelte.test.ts
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/svelte';
import EmptyState from '$lib/components/EmptyState.svelte';

describe('EmptyState', () => {
  it('renders the heading and prompt', () => {
    render(EmptyState);
    expect(screen.getByRole('heading', { name: /no habits yet/i })).toBeInTheDocument();
    expect(screen.getByText(/add your first one above/i)).toBeInTheDocument();
  });
});
```

Tiny but real. The test will fail the day someone changes the wording — which is a good thing if the wording is part of the user contract.
</details>

---

## Lesson 59.9 — Recurring concepts from earlier chapters

- **`addHabitForUser`** (Ch 42) — extracted as a pure-async function specifically *so* it could be integration-tested.
- **Atomic conditional UPDATE** (Ch 42) — the integration test is the runtime evidence for Bible rule #11.
- **`Result<T, E>`** (Ch 27) — every integration test asserts on `.ok` and narrows.
- **OpenAPI** — preview now; written formally in Ch 64.

---

## Lesson 59.10 — What you can now read in the wild

After Chapter 59 you can:

- Read **`render(Component, props)` + `screen.getByRole(...)` + `userEvent.click(...)`** as the component-test shape.
- Read **`flushSync()`** and **`$effect.root()`** in tests that exercise runes.
- Read **`beforeEach(() => db.execute(\`TRUNCATE ...\`))`** as the test-isolation pattern.
- Read an **OpenAPI-spec-validation** test and explain how it catches drift.
- Spot **mocked `RequestEvent`** as a code-review reject.

---

## Glossary added in Chapter 59

| Term | Definition |
|---|---|
| `@testing-library/svelte` | Render + query Svelte components in jsdom. |
| jsdom | Node-side DOM implementation for tests. |
| `flushSync` | Force pending Svelte reactivity to settle synchronously. |
| `$effect.root` | Effect scope for tests outside a component. |
| truncate-before-each | Test isolation by wiping all tables between tests. |
| contract test | Verifies an API matches its declared spec. |

---

## End-of-chapter checkpoint

- [ ] Component test for `HabitRow` and `EmptyState` is green.
- [ ] Integration test for `addHabitForUser` exists and runs against a real Postgres.
- [ ] Concurrent-insert test proves Bible rule #11 at runtime.
- [ ] (Optional) contract test against OpenAPI spec exists.

---

# Chapter 60 — Playwright e2e, accessibility, visual regression

> *Today's job:* `pnpm test:e2e` opens a real Chromium browser, signs up a fresh user, logs three habits, upgrades to Pro on a Stripe test card, signs out, signs in again, sees the habits and the Pro badge — every step asserted. `pnpm test:a11y` runs `@axe-core/playwright` against every key page and reports zero violations. `pnpm test:visual` flags layout regressions via screenshot diffs. *Visible win:* removing `aria-label` from a button breaks the a11y suite immediately, before any user notices.

E2e tests are the **runtime-evidence layer** Bible rule #21 demands. Compilation green and unit/integration green prove the parts work; e2e proves the *parts compose*.

---

## Lesson 60.1 — Playwright config

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: 'tests/e2e',
  fullyParallel: true,
  retries: process.env.CI ? 2 : 0,
  workers: process.env.CI ? 1 : undefined,
  reporter: process.env.CI ? 'github' : 'list',

  use: {
    baseURL: process.env.TEST_BASE_URL ?? 'http://localhost:4173',
    trace: 'on-first-retry',
    screenshot: 'only-on-failure',
    video: 'retain-on-failure',
  },

  webServer: process.env.TEST_BASE_URL
    ? undefined // CI: external URL, no local server
    : {
        command: 'pnpm build && pnpm preview',
        port: 4173,
        reuseExistingServer: !process.env.CI,
        timeout: 120_000,
      },

  projects: [
    { name: 'chromium', use: devices['Desktop Chrome'] },
    { name: 'firefox', use: devices['Desktop Firefox'] },
    { name: 'webkit', use: devices['Desktop Safari'] },
  ],
});
```

Read aloud:

| Field | Read aloud as |
|---|---|
| `fullyParallel: true` | *"Run tests in parallel by default."* |
| `retries: process.env.CI ? 2 : 0` | *"Retry twice on CI, never locally."* — masks transient flakiness in CI without hiding it locally. |
| `trace: 'on-first-retry'` | *"On first retry, save a full trace I can replay in the Playwright Inspector."* |
| `webServer` | *"Boot `pnpm preview` automatically; in CI, hit the real preview URL instead."* |
| `projects: [chromium, firefox, webkit]` | *"Run every test against Chrome, Firefox, and Safari engines."* |

> **trace** *(noun)* — a recording Playwright keeps of every action, network request, and DOM mutation during a test. Open with `pnpm exec playwright show-trace`. The single best e2e debugging tool.

---

## Lesson 60.2 — Page Object Model

For tests that touch the same UI repeatedly, extract page-objects. Keeps tests readable and isolates selectors.

```ts
// tests/e2e/pages/SignupPage.ts
import type { Page } from '@playwright/test';

export class SignupPage {
  constructor(private page: Page) {}

  async goto(): Promise<void> {
    await this.page.goto('/signup');
  }

  async fill(email: string, password: string): Promise<void> {
    await this.page.getByLabel(/email/i).fill(email);
    await this.page.getByLabel(/password/i).fill(password);
  }

  async submit(): Promise<void> {
    await this.page.getByRole('button', { name: /sign up/i }).click();
  }

  async signup(email: string, password: string): Promise<void> {
    await this.goto();
    await this.fill(email, password);
    await this.submit();
  }
}
```

Equivalent for `LoginPage`, `DashboardPage`, etc. Tests then read like English:

```ts
const signup = new SignupPage(page);
await signup.signup(email, password);
```

> **Page Object Model (POM)** *(noun)* — a senior pattern for organising e2e tests: one class per page, methods that match user verbs. Hides selector details behind a domain API.

---

## Lesson 60.3 — A full-flow e2e test

```ts
// tests/e2e/full-flow.spec.ts
import { test, expect } from '@playwright/test';
import { SignupPage } from './pages/SignupPage';
import { LoginPage } from './pages/LoginPage';
import { DashboardPage } from './pages/DashboardPage';

test('signup → log habits → logout → login → see habits', async ({ page }) => {
  const email = `user+${Date.now()}@example.com`;
  const password = 'correct horse battery staple';

  // 1. Sign up
  const signup = new SignupPage(page);
  await signup.signup(email, password);

  // 2. Verify email (in dev, the link goes to console; we read from the test DB instead)
  // For e2e we set a feature flag that skips email verification in test mode.
  // — see Lesson 60.5

  // 3. Land on dashboard
  const dashboard = new DashboardPage(page);
  await expect(page).toHaveURL(/\/dashboard$/);

  // 4. Log three habits
  await dashboard.addHabit('Read 20 minutes');
  await dashboard.addHabit('Walk 8000 steps');
  await dashboard.addHabit('Drink water');
  await expect(dashboard.habits()).toHaveCount(3);

  // 5. Logout
  await dashboard.logout();
  await expect(page).toHaveURL(/\/$|\/login/);

  // 6. Login
  const login = new LoginPage(page);
  await login.login(email, password);

  // 7. Habits survive
  await expect(dashboard.habits()).toHaveCount(3);
  await expect(page.getByText('Read 20 minutes')).toBeVisible();
});
```

---

## Lesson 60.4 — Test data and fixtures

E2e tests share state if you're not careful. The senior pattern is *one fresh user per test*, and a `beforeEach` that resets the test DB:

```ts
// tests/e2e/fixtures.ts
import { test as base } from '@playwright/test';
import { db } from '$lib/db/client';
import { sql } from 'drizzle-orm';

export const test = base.extend({
  page: async ({ page }, use) => {
    // Reset before each test — only safe against the test DB
    await db.execute(sql`TRUNCATE habits, sessions, users RESTART IDENTITY CASCADE`);
    await use(page);
  },
});

export { expect } from '@playwright/test';
```

Tests import `test` and `expect` from this fixture instead of `@playwright/test` directly. Every test starts from a clean slate.

---

## Lesson 60.5 — Testing flows that hit external services

Stripe and Resend can't run in the test DB. Two senior patterns:

1. **Stripe test mode** — use real Stripe APIs with `sk_test_*` keys; the `4242 4242 4242 4242` test card always succeeds.
2. **Test-mode bypass flags** — env vars like `TEST_SKIP_EMAIL_VERIFICATION=1` that the *server* respects only in test environments. Add a check inside the signup action:

```ts
import { dev } from '$app/environment';
const skipVerify = dev && process.env.TEST_SKIP_EMAIL_VERIFICATION === '1';
if (skipVerify) {
  await db.update(users).set({ emailVerifiedAt: new Date() }).where(eq(users.id, created.id));
}
```

Wired only in `dev` builds; never on production.

---

## Lesson 60.6 — Accessibility tests with axe

```bash
pnpm add -D @axe-core/playwright
```

```ts
// tests/e2e/a11y.spec.ts
import { test, expect } from '@playwright/test';
import AxeBuilder from '@axe-core/playwright';

const ROUTES = ['/', '/about', '/pricing', '/login', '/signup'];

for (const route of ROUTES) {
  test(`${route} has no a11y violations`, async ({ page }) => {
    await page.goto(route);
    const results = await new AxeBuilder({ page })
      .withTags(['wcag2a', 'wcag2aa'])
      .analyze();
    expect(results.violations).toEqual([]);
  });
}
```

Read aloud: *"For each marketing route, navigate, run axe with WCAG 2.0 A and AA tags, expect zero violations."*

When axe reports a violation, it includes the *exact element* and a link to the rule. Senior debugging: open the trace, find the highlighted element, fix it.

> **WCAG 2.0 A / AA / AAA** — the three conformance levels of the Web Content Accessibility Guidelines. AA is the senior bar (legally required in many jurisdictions for public-facing apps).

axe is *automated* a11y testing — it catches about 30% of WCAG violations. The other 70% require keyboard testing and screen-reader testing by hand. Senior habit: **also tab through the app** at least once per release.

---

## Lesson 60.7 — Visual regression

```ts
// tests/e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test('home matches snapshot', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot('home.png', {
    maxDiffPixelRatio: 0.01, // tolerate up to 1% pixel diff (anti-aliasing, etc.)
    fullPage: true,
  });
});

test('pricing matches snapshot', async ({ page }) => {
  await page.goto('/pricing');
  await expect(page).toHaveScreenshot('pricing.png', { maxDiffPixelRatio: 0.01, fullPage: true });
});
```

First run with `pnpm exec playwright test --update-snapshots` creates the baseline images. Subsequent runs compare. When a CSS change shifts the layout, the test fails *and* writes a diff image to `test-results/` you can inspect.

> **visual regression** — automated detection of pixel-level layout/color changes between runs.

---

## Lesson 60.8 — Read this code

```ts
test('add habit', async ({ page }) => {
  await page.goto('/dashboard');
  await page.click('.habit-form button');
  await page.waitForTimeout(1000);
  expect(await page.locator('.habit').count()).toBe(1);
});
```

Three issues. Find them.

<details>
<summary>Answer</summary>

1. **`page.click('.habit-form button')` selects by class.** Brittle: rename the CSS class and the test breaks. Use **`getByRole`** (e.g. `page.getByRole('button', { name: /add/i })`).
2. **`page.waitForTimeout(1000)` is a hardcoded sleep.** Flaky and slow. Replace with **`expect(...).toBeVisible()`** or **`expect(...).toHaveCount(1)`** — Playwright auto-retries until the assertion passes or the test times out.
3. **Doesn't fill the input.** The test "adds a habit" by clicking add with no name. The form's validation will reject. Add `await page.getByLabel(/habit name/i).fill('Read');` before the click.

The senior version:

```ts
test('add habit', async ({ page }) => {
  await page.goto('/dashboard');
  await page.getByLabel(/habit name/i).fill('Read');
  await page.getByRole('button', { name: /add/i }).click();
  await expect(page.getByText('Read')).toBeVisible();
  await expect(page.getByRole('listitem')).toHaveCount(1);
});
```
</details>

---

## Lesson 60.9 — Now you write it

**The English sentence first:**

> *"Write a Playwright test for the optimistic-delete behaviour from Chapter 43: navigate to the dashboard, add a habit, throttle the network to Slow 3G, click delete, assert the row vanishes immediately (before the network request completes)."*

<details>
<summary>Worked answer</summary>

```ts
// tests/e2e/optimistic-delete.spec.ts
import { test, expect } from './fixtures';

test('delete is optimistic on slow network', async ({ page, context }) => {
  await page.goto('/dashboard');
  await page.getByLabel(/habit name/i).fill('Doomed');
  await page.getByRole('button', { name: /add/i }).click();
  await expect(page.getByText('Doomed')).toBeVisible();

  // Throttle to "Slow 3G"
  const cdp = await context.newCDPSession(page);
  await cdp.send('Network.enable');
  await cdp.send('Network.emulateNetworkConditions', {
    offline: false,
    latency: 400,
    downloadThroughput: (50 * 1024) / 8,
    uploadThroughput: (50 * 1024) / 8,
  });

  // Click delete; assert the row vanishes inside 200 ms (well before the throttled network resolves)
  const deleted = page.getByText('Doomed');
  await page.getByRole('button', { name: /remove doomed/i }).click();
  await expect(deleted).not.toBeVisible({ timeout: 200 });
});
```

The 200 ms timeout is the proof: at 400 ms latency the request hasn't returned yet, but the row has already vanished. That's the optimistic update doing its job.
</details>

---

## Lesson 60.10 — Recurring concepts from earlier chapters

- **Optimistic UI** (Ch 43) — e2e is the runtime evidence.
- **ARIA roles** (Ch 53) — `getByRole` is the natural query because we wired ARIA correctly.
- **`use:enhance`** (Ch 41) — works with JS off (test it!) and feels instant with JS on (test that too!).

---

## Lesson 60.11 — What you can now read in the wild

After Chapter 60 you can:

- Read **`playwright.config.ts`** with `webServer`, `projects`, `trace`, `retries`.
- Read **`page.getByRole(...)` / `getByLabel(...)` / `getByText(...)`** as the accessibility-aligned queries.
- Read a **Page Object Model** test class and tell it from a brittle selector-based test.
- Read **`AxeBuilder({ page }).withTags(['wcag2a', 'wcag2aa']).analyze()`** as the a11y check.
- Read **`expect(page).toHaveScreenshot(...)`** as the visual-regression check.
- Spot **`page.waitForTimeout(...)` and `.css-class` selectors** as code-review rejects.

---

## Glossary added in Chapter 60

| Term | Definition |
|---|---|
| trace | Playwright's recording of every action; replayable in the Inspector. |
| Page Object Model | One class per page; methods that match user verbs. |
| WCAG AA | The senior accessibility bar; legally required in many jurisdictions. |
| visual regression | Screenshot-diff test that catches layout/color changes. |
| `getByRole` | The accessibility-aligned query; preferred over CSS selectors. |

---

## End-of-chapter checkpoint

- [ ] Full-flow e2e test passes against three browser engines (Chromium, Firefox, WebKit).
- [ ] `tests/e2e/a11y.spec.ts` reports zero WCAG-AA violations on every marketing page.
- [ ] Visual snapshots exist for `/`, `/about`, `/pricing`.
- [ ] Optimistic-delete test passes under throttled network.
- [ ] You tabbed through the entire app keyboard-only at least once.

---

# Chapter 61 — CI/CD with GitHub Actions

> *Today's job:* every PR runs lint + type-check + unit + integration + e2e + build before it can merge. `main` is protected against force pushes. Failing checks block the merge button. *Visible win:* push a PR with a deliberate type error; watch CI go red; the merge button is greyed out; fix it; CI goes green; merge.

CI/CD — Continuous Integration / Continuous Deployment — is the *automation layer* that catches what code review misses. The senior pattern: **every change runs every test before any human approves.**

---

## Lesson 61.1 — The CI workflow

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

env:
  NODE_VERSION: 22
  PNPM_VERSION: 9

jobs:
  lint-and-type:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm check
      - run: pnpm exec eslint src/

  unit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm test:unit

  integration:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16
        env:
          POSTGRES_PASSWORD: dev
          POSTGRES_DB: streak_test
        ports:
          - 5432:5432
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
    env:
      TEST_DATABASE_URL: postgres://postgres:dev@localhost:5432/streak_test
      DATABASE_URL: postgres://postgres:dev@localhost:5432/streak_test
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:migrate
      - run: pnpm test:integration

  e2e:
    runs-on: ubuntu-latest
    needs: [unit, integration]
    services:
      postgres:
        image: postgres:16
        env: { POSTGRES_PASSWORD: dev, POSTGRES_DB: streak_test }
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    env:
      DATABASE_URL: postgres://postgres:dev@localhost:5432/streak_test
      STRIPE_SECRET_KEY: sk_test_dummy
      STRIPE_WEBHOOK_SECRET: whsec_dummy
      RESEND_API_KEY: re_dummy
      R2_ACCOUNT_ID: dummy
      R2_ACCESS_KEY_ID: dummy
      R2_SECRET_ACCESS_KEY: dummy
      R2_BUCKET: dummy
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm db:migrate
      - name: Install Playwright browsers
        run: pnpm exec playwright install --with-deps chromium firefox
      - run: pnpm test:e2e
      - name: Upload Playwright report
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 7

  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm build
        env:
          DATABASE_URL: postgres://dummy
          STRIPE_SECRET_KEY: sk_test_dummy
          STRIPE_WEBHOOK_SECRET: whsec_dummy
          RESEND_API_KEY: re_dummy
          R2_ACCOUNT_ID: dummy
          R2_ACCESS_KEY_ID: dummy
          R2_SECRET_ACCESS_KEY: dummy
          R2_BUCKET: dummy
```

Five parallel jobs (`lint-and-type`, `unit`, `integration`, `e2e`, `build`); `e2e` depends on `unit` and `integration`. The Playwright report uploads as an artifact so you can download and replay the trace when something fails on CI but passes locally.

---

## Lesson 61.2 — Branch protection

In GitHub: **Settings → Branches → Branch protection rules → main**:

- ✅ **Require a pull request before merging** (1 review for solo dev, 2 for teams).
- ✅ **Require status checks to pass** — pick `lint-and-type`, `unit`, `integration`, `e2e`, `build`.
- ✅ **Require branches to be up to date before merging.**
- ✅ **Require linear history** (rebase or squash, no merge commits).
- ✅ **Do not allow bypassing the above settings** (yes, even for admins).
- ❌ **Allow force pushes**.
- ❌ **Allow deletions**.

The Bible's `--no-verify` rule is the inverse of branch protection. Branch protection prevents the *server side* from accepting a no-verify push that bypassed pre-commit hooks.

---

## Lesson 61.3 — Conventional commits and `release-please`

```bash
pnpm add -D @commitlint/cli @commitlint/config-conventional
```

`commitlint.config.cjs`:

```js
module.exports = { extends: ['@commitlint/config-conventional'] };
```

A pre-commit hook (via Husky) runs `commitlint` on every commit message; rejects messages that aren't `<type>(<scope>): <message>`. Examples:

- `feat(billing): add streak freezes`
- `fix(auth): rotate session on password change`
- `refactor(money): extract Cents type`

Pair with [`release-please`](https://github.com/googleapis/release-please) (a GitHub Action) — it parses your commit history and opens a PR every time `main` advances, with a generated CHANGELOG and a version bump.

> **Conventional Commits** — a commit-message format that machines can parse for changelog generation. `<type>(<scope>): <description>`.
>
> **`release-please`** — a tool that turns conventional commits into auto-generated CHANGELOGs and version bumps.

---

## Lesson 61.4 — Vercel preview deployments

Connect your GitHub repo to Vercel (Lesson 62.2). Vercel automatically deploys every PR to a preview URL like `streak-pr-123.vercel.app`. Senior pattern:

1. Set every required env var in Vercel under **Preview** environment.
2. Configure a separate **Preview** Postgres DB (e.g. a Neon branch per PR).
3. Run e2e tests against the preview URL via `TEST_BASE_URL`.

The preview-deploy workflow:

```yaml
# .github/workflows/preview-e2e.yml
name: Preview e2e
on:
  deployment_status:

jobs:
  e2e-preview:
    if: github.event.deployment_status.state == 'success' && github.event.deployment.environment == 'Preview'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm exec playwright install --with-deps chromium
      - run: pnpm test:e2e
        env:
          TEST_BASE_URL: ${{ github.event.deployment_status.target_url }}
```

---

## Lesson 61.5 — Caching `pnpm`, Playwright, build artifacts

`pnpm/action-setup` + `actions/setup-node` with `cache: pnpm` already caches `~/.pnpm-store` between runs. For Playwright browsers, add:

```yaml
- name: Cache Playwright browsers
  uses: actions/cache@v4
  with:
    path: ~/.cache/ms-playwright
    key: playwright-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}
- run: pnpm exec playwright install --with-deps chromium firefox
```

Saves ~2 minutes per CI run.

---

## Lesson 61.6 — The "no `--no-verify`" rule, applied

Every senior team has it as policy: **`git commit --no-verify` and `git push --no-verify` are banned for production code.** Bible rule echo: skipping hooks is bypassing the very tests CI exists to enforce. The hook rejects bad commits *before* CI; bypassing it just makes CI find them later. Don't.

If a hook fails, **fix the underlying issue, then commit again.** Never `--no-verify` to "ship the urgent thing"; the urgent thing is rarely as urgent as the broken thing you'll ship.

---

## Lesson 61.7 — Read this code

```yaml
- run: pnpm test:e2e
  continue-on-error: true
```

Why's this dangerous?

<details>
<summary>Answer</summary>

`continue-on-error: true` means *"if this fails, the job still passes."* Now your e2e suite can be entirely broken and the merge button stays green. Bible rule violation: *"compilation and tests are necessary, not sufficient — demand runtime evidence."* Removing the gate inverts the contract.

The right pattern: if a test is genuinely flaky and you can't fix it today, mark *that test* as `.skip` with a TODO, not the entire job as continue-on-error.
</details>

---

## Lesson 61.8 — Now you write it

**The English sentence first:**

> *"Add a job to the workflow that runs `pnpm security:headers` against the preview deploy. It should fail the PR if any required header is missing."*

<details>
<summary>Worked answer (sketch)</summary>

```yaml
security-headers:
  runs-on: ubuntu-latest
  needs: [build]
  if: github.event_name == 'pull_request'
  steps:
    - uses: actions/checkout@v4
    - uses: pnpm/action-setup@v3
      with: { version: 9 }
    - uses: actions/setup-node@v4
      with: { node-version: 22, cache: pnpm }
    - run: pnpm install --frozen-lockfile
    # Wait for the Vercel preview deploy via a polling action, then:
    - run: pnpm security:headers
      env:
        TEST_BASE_URL: ${{ steps.vercel-url.outputs.url }}
```

Add `security-headers` to the required status checks for `main`. Now a PR can't merge if security headers regress.
</details>

---

## Lesson 61.9 — Recurring concepts from earlier chapters

- **`pnpm install --frozen-lockfile`** — Bible rule #1 enforced in CI.
- **Boot validator** (Ch 57) — every CI job exports the env vars the validator demands.
- **`security:headers` test** (Ch 48) — wired into CI here.
- **The "no skipping hooks" rule** — Bible foundation, applied at the merge gate.

---

## Lesson 61.10 — What you can now read in the wild

After Chapter 61 you can:

- Read **`.github/workflows/*.yml`** with parallel jobs, services, secrets, artifacts.
- Read **branch-protection settings** and tell strict from permissive.
- Read **conventional-commit messages** and `release-please` outputs.
- Spot **`continue-on-error: true`** as a CI bypass.
- Spot **missing test caching** as a CI-runtime regression.

---

## Glossary added in Chapter 61

| Term | Definition |
|---|---|
| GitHub Actions | GitHub's CI/CD runner. |
| services (in CI) | Sidecar containers (e.g. Postgres) the job uses. |
| branch protection | Server-side rules preventing force-push and bypassing checks. |
| Conventional Commits | `<type>(<scope>): <message>` format. |
| `release-please` | Tool that auto-generates CHANGELOGs from conventional commits. |
| preview deploy | Per-PR auto-deployed URL for testing the change in production-shape infra. |

---

## End-of-chapter checkpoint

- [ ] PRs trigger CI; failing checks block merge.
- [ ] `main` is protected (required checks, no force-push, linear history).
- [ ] Playwright report uploads as artifact when e2e fails.
- [ ] Conventional commits enforced via commitlint.
- [ ] Preview deploy URL hits the same e2e suite.

---

# Chapter 62 — Deploying to Vercel, custom domains

## Lesson 62.1 — adapter-vercel

```bash
pnpm add -D @sveltejs/adapter-vercel
```

```ts
// svelte.config.ts
import adapter from '@sveltejs/adapter-vercel';
export default {
  kit: { adapter: adapter({ runtime: 'nodejs22.x' }) },
};
```

---

## Lesson 62.2 — Vercel project

1. Push to GitHub.
2. Import in Vercel.
3. Set every env var (production + preview).
4. Custom domain → DNS records → wait for cert.
5. Enable HSTS preload (after a few weeks of HTTPS-only).

---

## Lesson 62.3 — Pre-launch checklist

`docs/checklists/launch.md`:

```
- [ ] All env vars set in production
- [ ] Stripe webhook URL updated to prod URL
- [ ] DB migrations applied
- [ ] Sentry DSN active (or chosen logger)
- [ ] Rate-limits enabled
- [ ] robots.txt allows what it should
- [ ] sitemap.xml validates
- [ ] Lighthouse on / is 100/100/100/100
- [ ] OpenAPI spec matches /api/v1/*
- [ ] Backup strategy documented
- [ ] Rollback runbook current
- [ ] On-call rotation defined (even if it's just you)
```

---

## End-of-chapter checkpoint

- [ ] Streak is live at a real URL.
- [ ] You signed up on your own production site.

---

# Chapter 63 — Observability — logs, metrics, traces, on-call

## Lesson 63.1 — Structured logger

```bash
pnpm add pino
```

```ts
// src/lib/logger.ts
import pino from 'pino';

const SENSITIVE = ['password', 'passwordHash', 'token', 'cookie', 'sessionToken', 'authorization'];

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  redact: {
    paths: SENSITIVE,
    remove: true,
  },
});
```

---

## Lesson 63.2 — Request IDs

```ts
// hooks.server.ts handle:
const requestId = crypto.randomUUID();
event.locals.requestId = requestId;
const start = Date.now();
const response = await resolve(event);
response.headers.set('x-request-id', requestId);
logger.info('request', {
  requestId,
  method: event.request.method,
  path: event.url.pathname,
  status: response.status,
  durationMs: Date.now() - start,
});
return response;
```

---

## Lesson 63.3 — Prometheus metrics with bounded labels

```bash
pnpm add prom-client
```

```ts
// src/lib/metrics.ts
import { Counter, Histogram, register } from 'prom-client';

export const requestCounter = new Counter({
  name: 'streak_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_class'] as const,
});

export const requestLatency = new Histogram({
  name: 'streak_request_duration_seconds',
  help: 'Request latency',
  labelNames: ['method', 'route'] as const,
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
});

const KNOWN_ROUTES = new Set(['/', '/login', '/signup', '/app/habits', '/billing']);

export function safeRouteLabel(path: string): string {
  return KNOWN_ROUTES.has(path) ? path : 'other';
}

export function safeStatusClass(status: number): string {
  return `${Math.floor(status / 100)}xx`;
}
```

The `safeRouteLabel` is the **bounded-label rule** applied — never feed raw user input as a label.

Expose via `/metrics/+server.ts`:

```ts
import { register } from 'prom-client';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async () => {
  return new Response(await register.metrics(), { headers: { 'content-type': register.contentType } });
};
```

(Gate it; only allow Prometheus scrapers.)

---

## Lesson 63.4 — Reading prod logs

The drill: a synthetic 200-line log dump with planted bugs. Reader spots:
- a 500 spike at minute 12,
- a missing `requestId` on one log line (bug),
- a slow query (`durationMs > 1000`) recurring,
- the missing audit-log-on-cancel bug.

This is the chapter that produces the on-call mindset.

---

## End-of-chapter checkpoint

- [ ] Logs are JSON, structured, with request IDs.
- [ ] Metrics endpoint serves Prometheus format.
- [ ] You debugged the planted log dump.

---

# Chapter 64 — Public REST API, emails, cron

## Lesson 64.1 — REST versioning

```
/api/v1/habits        GET (list), POST (create)
/api/v1/habits/[id]   GET (one), PATCH (update), DELETE
```

---

## Lesson 64.2 — Personal access tokens

```ts
export const personalAccessTokens = pgTable('personal_access_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  hashedToken: text('hashed_token').notNull().unique(),
  name: text('name').notNull(),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Token shown once at creation: `pat_<base64>`. Stored hashed.

---

## Lesson 64.3 — RFC 7807 errors

```ts
function problem(status: number, title: string, detail: string): Response {
  return new Response(JSON.stringify({ type: 'about:blank', title, detail, status }), {
    status,
    headers: { 'content-type': 'application/problem+json' },
  });
}
```

---

## Lesson 64.4 — Resend for email

```bash
pnpm add resend
```

```ts
// src/lib/mail/client.ts
import { Resend } from 'resend';
import { RESEND_API_KEY } from '$env/static/private';

export const resend = new Resend(RESEND_API_KEY);
```

Send:

```ts
await resend.emails.send({
  from: 'Streak <noreply@streak.example.com>',
  to: user.email,
  subject: 'Verify your email',
  html: emailVerificationHtml(token),
});
```

---

## Lesson 64.5 — Cron via `vercel.json`

```json
{
  "crons": [
    { "path": "/api/cron/daily-reminders", "schedule": "0 9 * * *" }
  ]
}
```

In the handler, check a shared secret in the `Authorization` header so randoms can't trigger cron.

---

## End-of-chapter checkpoint

- [ ] `/api/v1/habits` works with PAT auth.
- [ ] Emails actually send.
- [ ] Cron job fires.

---

# Chapter 65 — Code review, architecture, scaling, ADRs, boring tech

## Lesson 65.1 — The structured-review framework

For every PR:

| Category | Questions |
|---|---|
| Correctness | Does this match the spec? Off-by-one? Boundary conditions? |
| Security | New attack surface? Inputs trusted? Secrets logged? |
| Performance | New query without index? N+1? Bundle bloat? |
| Maintainability | Names? Comments lying? Repeated logic that should be extracted? |
| Accessibility | ARIA? Keyboard? Contrast? |
| Observability | Errors logged? Metrics? Useful breadcrumbs? |
| Tests | Unit + integration + e2e where appropriate? |
| Docs | API change documented? Runbook updated? |

---

## Lesson 65.2 — The 600-line PR fixture

A PR with planted bugs:
- A SELECT-then-UPDATE (TOCTOU).
- A console.log of `password`.
- A new metric with raw user input as a label.
- An `<img>` without width/height.
- A `// O(1) lookup` comment on an O(n) implementation.
- A `.catch(() => {})` swallow.
- A migration that drops a column without a deprecation period.
- An unbounded retry loop.

Reader writes a structured review naming each. Cite the Bible rule per finding.

---

## Lesson 65.3 — Scaling plan

*"At 100k users with 30 habits each, where does Streak break first?"*

The reader works through:
- DB connections (pool sizing).
- Read vs write ratio (do we need read replicas?).
- Query plans for the 10 hottest queries.
- Hot tables that might need partitioning (audit_log).
- Cache opportunities (immutable habit data).
- Edge vs origin tradeoffs.

---

## Lesson 65.4 — ADR template

```
# ADR-NNN: <Decision>

## Status
Accepted | Rejected | Superseded by ADR-XXX

## Context
What forced the decision.

## Decision
What we chose.

## Alternatives considered
What we rejected, why.

## Consequences
What this enables and constrains.
```

Reader writes ADR-001 for *"Why Postgres, not Mongo."*

---

## Lesson 65.5 — The boring-tech doctrine

Pick technologies whose failure modes are well-understood. Postgres > exotic-DB-of-the-month. Stripe > rolling your own payments. Boring is fast in 6 months, even when novel feels fast today.

---

## End-of-chapter checkpoint

- [ ] You wrote a 600-line review.
- [ ] You wrote ADR-001.
- [ ] You wrote a 1-page scaling plan.

---

# Chapter 66 — Graduation Part I: the brief

> *Build Streak Freezes. From a one-paragraph brief. Alone.*

> **Streak Freezes — Pro feature.**
> Pro users get one streak freeze per calendar month: a one-day skip that doesn't break their streak. The home screen shows the count of remaining freezes for the current month. Consuming a freeze is a deliberate user action — confirm dialog and audit row. The reset to 1 happens at 00:00 UTC on the first of every month. Safe under concurrent consumption (a user clicking twice cannot consume two freezes). Roll out behind a feature flag. Fully observable.

Required deliverables:

1. Migration: `users.freezes_remaining_month` (integer, default 1, NOT NULL) — or a `freeze_consumption` table; reader's call, defended in ADR.
2. ADR explaining the schema decision.
3. `consumeFreeze` form action with atomic conditional `UPDATE`.
4. `withAudit` wrapper for the action.
5. Pro-status check at action and UI.
6. `<ConfirmDialog>` using the existing primitive.
7. `/api/cron/reset-freezes/+server.ts` — idempotent monthly reset, with `withAudit`.
8. Unit tests; property-based where applicable.
9. Integration test: 10 simultaneous consume calls; assert ≤ 1 succeeds.
10. Playwright e2e: happy path + "no freezes left" path.
11. Feature flag: `PUBLIC_ENABLE_STREAK_FREEZES`.
12. Prometheus metric `streak_freezes_consumed_total{plan}` (bounded label).
13. Runbook at `docs/runbooks/streak-freezes.md`.
14. PR description citing the Bible rules followed.
15. Runtime evidence for every claim.

Estimated 6–10 hours of focused work.

There are no lessons in this chapter. Only the brief.

When you commit and merge, return for Chapter 67.

---

# Chapter 67 — Graduation Part II: the post-mortem

> *Write the post-mortem of your own feature.*

Format:

```markdown
# Streak Freezes — Post-mortem (YYYY-MM-DD)

## Context
Why we shipped this, what the user need was.

## What I built
3-line summary.

## Decisions and alternatives
For each significant decision: what I chose, what I rejected, why.

## Surprises
What I didn't expect — both helpful and harmful.

## What I'd do differently
Honest reflection.

## What's still scary
The tail risks I haven't covered.

## Bible rules cited
List of rules followed (and any deliberately bent, with rationale).

## What I'd plant for the next learner
If I were teaching this feature, where would I put a deliberate bug for them to find?
```

Then:

1. Reread the 21 non-negotiables at the front of the book. Circle the ones that hit hardest in retrospect.
2. Inventory what's still scary. That's your next learning agenda.

**You are now a Svelte 5 + SvelteKit + TypeScript + Postgres + Stripe principal engineer as of May 5, 2026.**

---

# Appendix A — Remote Functions, the experimental future

`$app/server` exports `query`, `query.batch`, `query.live`, `form`, `command`. Experimental as of May 2026.

```ts
// src/routes/(app)/habits.remote.ts
import { query, form } from '$app/server';
import * as v from 'valibot';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';

export const getHabits = query(async () => db.select().from(habits));

export const addHabit = form(
  v.object({ name: v.pipe(v.string(), v.minLength(1)) }),
  async ({ name }) => {
    await db.insert(habits).values({ userId: 'demo', name });
    void getHabits().refresh();
  },
);
```

In the page:

```svelte
<script lang="ts">
  import { getHabits, addHabit } from './habits.remote';
</script>

<form {...addHabit}>
  <input {...addHabit.fields.name.as('text')} />
  <button>Add</button>
</form>

{#each await getHabits() as h}<li>{h.name}</li>{/each}
```

Experimental. Don't bet a production app on it yet, but know it exists. The book taught you the stable `+page.server.ts` + `actions` story; remote functions are where it's heading.

---

# Appendix B — Reading list

- **Svelte docs** — `https://svelte.dev/docs` (primary). Section anchors: runes, snippets, transitions, motion.
- **SvelteKit docs** — `https://kit.svelte.dev/docs`. Section anchors: routing, load, form actions, hooks, adapters.
- **Markus Winand — *Use The Index, Luke!*** Free online. Read it once; reread the EXPLAIN chapters yearly.
- **Addy Osmani — *Image Optimization*.** The CLS chapter especially.
- **OWASP Cheat Sheets** — Authentication, Session Management, Input Validation, Logging, REST Security.
- **Google SRE Workbook** — chapters on alerting, error budgets, post-mortems.
- **Stripe docs** — Idempotency, Webhook signing, Best practices.
- **Conventional Commits** — `https://conventionalcommits.org`.
- **Vercel docs** — Adapters, ISR, Edge Functions.
- **Cloudflare R2 docs** — presigned URLs, multipart uploads.

---

# Appendix C — The Bible card

Print this. Stick it above your monitor.

```
THE STREAK BIBLE — May 5, 2026

TOOLING
1. pnpm only.
2. .svelte and .ts only — never .js.
3. TypeScript strict from line one. No any. No !. No @ts-ignore.
4. Svelte 5 runes only — no export let, no $:, no on:click, no <slot>.

IDIOMS
5. += not = x + 1. === not ==. const by default. ??not ||. ?. over manual null checks.
6. Engineer-English read-aloud on every snippet.
7. Every example is load-bearing.
8. English sentence first, code second.
9. No comment lies.

CORRECTNESS
10. Money is integer cents end-to-end. No floats.
11. Atomic UPDATE … WHERE … RETURNING. Never SELECT-then-UPDATE.
12. Idempotency on every external-side-effect handler.
13. .timeout() and .connect_timeout() on every external client.
14. Bounded label sets on every metric.
15. No password/token/secret/PII ever logged.
16. No <img> without width and height.
17. No silent .catch(() => {}).
18. Migrations are forward-only.
19. Server-only secrets via $env/static/private. Build error if leaked.
20. Every chapter delivers a visible win.

EVIDENCE
21. Compilation and tests are necessary, not sufficient. Demand runtime evidence.
```

---

**End of book.**

The reader has built Streak from `pnpm create` to a deployed, audited, observable, paid product, and shipped a feature on their own. They are a principal-engineer-level-7 specialist in this stack as of May 5, 2026.

Welcome to the trade.
