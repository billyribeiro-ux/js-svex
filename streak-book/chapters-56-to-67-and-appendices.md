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

function redact(err: unknown): { name: string; message: string; stack?: string } {
  if (!(err instanceof Error)) {
    return { name: 'Unknown', message: 'Unknown error' };
  }
  const message = err.message
    .replace(/[a-zA-Z0-9._-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]+/g, '[redacted-email]')
    .replace(/\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b/g, '[redacted-card]');
  return { name: err.name, message, stack: err.stack };
}
```

Postgres errors leak PII via *"duplicate key value"* messages — the redactor handles the common shapes (emails, card numbers); for production, add domain-specific patterns as you discover them in your logs.

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

Why `console.error` and not `pino`? The client side doesn't have pino — bundling pino into the browser would balloon the JS payload. Server-side `handleError` uses pino through `$lib/logger`; client-side uses `console.error` with a structured object payload. The client can't reach the structured logger because doing so would ship pino to the browser bundle. In production, replace this `console.error` with a thin wrapper around your telemetry SDK (Sentry's `captureException`, Logflare's HTTP API, etc.) — those *are* designed to ship to the browser.

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
  {#snippet failed(error, reset)}
    <div class="boundary-fail">
      <p>This chart couldn't load.</p>
      <button type="button" onclick={reset}>Retry</button>
    </div>
  {/snippet}
</svelte:boundary>
```

Read aloud: *"render the chart; if it throws during render, swap in the failed snippet, which lets the user retry."*

The rest of the page (header, nav, other widgets) keeps working. This is the *graceful-degradation* pattern.

> **`<svelte:boundary>`** — Svelte 5 element (added in 5.3) that catches errors thrown during render of its children. The `failed` snippet receives `(error, reset)`; `reset` re-creates the boundary's contents. Pinned to the May 2026 docs at `svelte/svelte-boundary`; verify before adopting if the API has shifted in a later minor.

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

  {#snippet failed(error, reset)}
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

> **`PUBLIC_` prefix** — SvelteKit's signal that a value is browser-safe. Env vars imported via `$env/static/private` (or `$env/dynamic/private`) are server-only; the build refuses to ship them to the client. The `PUBLIC_` prefix is what gates `static/public` and `dynamic/public` access. A non-prefixed env var without an explicit private import isn't auto-private — it's just unimported. The prefix is configurable via `kit.env.publicPrefix` in `svelte.config.ts`; we use the default.

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
import { env } from '$lib/env'; // top-level import → validator runs at module load
```

Top-level imports run at module load. On Vercel serverless, that's per-cold-start. The validator throws synchronously at import; the function fails to boot if env vars are missing. The validation cost is paid once per cold start, not per request.

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

`SENTRY_DSN` is `optional` because dev environments don't ship to Sentry. In production, the running server fails to boot if it's malformed. Dynamic-private env vars are populated at runtime, not build time — the build won't fail on a missing optional dynamic var; only the running server validates. Add to `.env.example` (commented):

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
        // `Cents extends number`, so the sum works without a cast.
        const sum = parts.reduce<number>((a, b) => a + b, 0);
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
  expect(result.success).toBe(true);
});
```

What's wrong? (Two bugs — one design, one assertion.)

<details>
<summary>Answer</summary>

1. **`mockDb`** — Bible rule #5: *don't mock the database*. Mocked DBs lie about transactional semantics, foreign-key behaviour, atomic UPDATEs, race conditions. The integration test against a real Postgres (Ch 59) is the right home for this. A unit test should test a *pure* function. If `addHabit` touches the DB, push the DB-touching part down to a thin layer and unit-test the *pure logic* (validation, name normalisation, the `Result` shape).

2. **`expect(result.success).toBe(true)`** — the test reads the wrong field. The book's `Result` API uses `.ok`, not `.success`. The test would *fail* — `result.success` is `undefined`, not `true` — but the failure message points at the assertion, not at the typo. A senior reviewer catches this in review and saves the round-trip through CI. Fix: `expect(result.ok).toBe(true)`.
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

**Locale trap.** `formatCents` uses `Intl.NumberFormat`; the output depends on the runtime's locale. For en-US, `$12,345.67`; for de-DE, `12.345,67 €`. The regex `[^\d.]` strips `$` and `,` — fine for en-US, *broken* for de-DE (which swaps the period and comma). For round-trip tests, either pin the locale (`formatCents(c, { locale: 'en-US' })`) or use the simpler `(c / 100).toFixed(2)` form that doesn't depend on locale at all. The book's snippet pins en-US implicitly via the test runtime; document the assumption so the test isn't a flake-on-CI in a different locale.
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

The `projects` array is the recommended form in Vitest 1.6+; older versions used `test.workspace` in a separate `vitest.workspace.ts` file. Pin to your installed version — verify against May 2026 Vitest before copy-pasting.

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
| `screen.getByRole('button', { name: /remove read/i })` | *"Find the button accessible-named 'Remove Read' (case-insensitive)."* (the `aria-label="Remove ${habit.name}"` was wired in Chapter 8 / Chapter 13's `HabitRow`.) |
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
  // Apply migrations to the test DB. `stdio: 'pipe'` keeps the test output
  // clean; we only print drizzle's chatter when the migration actually fails.
  try {
    execSync('pnpm drizzle-kit migrate', {
      env: { ...process.env, DATABASE_URL: process.env.TEST_DATABASE_URL },
      stdio: 'pipe',
    });
  } catch (e) {
    console.error(e);
    throw e;
  }
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
// `client.end()` returns `Promise<void>` in `postgres-js` 3.x (verified May 2026).
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
// Pin to ajv 8.x in package.json: `"ajv": "^8.17.1"`, `"ajv-formats": "^3.0.1"`.
// ajv 8.x ships a default export (`import Ajv from 'ajv'`); some prereleases
// shipped a named export (`import { Ajv } from 'ajv'`). Verify against the
// version in your lockfile before copying.
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
  // CI flakes are the only place retries help — same machine, same code, but the
  // ephemeral runner sometimes hits a transient network / browser hiccup. Retry
  // twice on CI; never locally (a local flake is a real bug, fix it).
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
| `retries: process.env.CI ? 2 : 0` | *"Retry twice on CI, never locally."* — CI flakes (network, browser-startup hiccups on ephemeral runners) are the only place retries help; a local flake is a real bug. |
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

> **The two-process trap.** Playwright tests run against the *running preview server*; the truncate fixture connects to the DB *from the test process*. Both must point at the same database. Set `DATABASE_URL` in `.env.test` and have both Playwright's `webServer` config and the test fixture's DB client read from `.env.test` (e.g. via `dotenv/config` in a setup file). If the preview server uses prod and the truncate uses test, the test wipes nothing the server reads — and your fresh-user-per-test guarantee silently breaks.

---

## Lesson 60.5 — Testing flows that hit external services

Stripe and Resend can't run in the test DB. Two senior patterns:

1. **Stripe test mode** — use real Stripe APIs with `sk_test_*` keys; the `4242 4242 4242 4242` test card always succeeds.
2. **Test-mode bypass flags** — env vars like `TEST_MODE=1` that the *server* respects only in test environments. Add a check inside the signup action:

```ts
const skipVerify = process.env.TEST_MODE === '1';
if (skipVerify) {
  await db.update(users).set({ emailVerifiedAt: new Date() }).where(eq(users.id, created.id));
}
```

Use `TEST_MODE=1` instead of `dev` from `$app/environment`: Playwright runs against the built *preview* server (`pnpm preview`), where `dev` is `false`, so the bypass would never fire. The env var is set only in `.env.test` and never in production — production deploys never see `TEST_MODE`, so the bypass is unreachable there.

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

Visual tests are notoriously flaky if you don't control four things:

1. **Viewport size** — different runners default differently.
2. **Animations** — running animations make screenshots non-deterministic.
3. **Font loading** — first render before fonts swap looks different.
4. **Dynamic dates** — relative timestamps ("2 minutes ago") move on every run.

A senior visual test pins all four:

```ts
// tests/e2e/visual.spec.ts
import { test, expect } from '@playwright/test';

test.describe('visual regression', () => {
  test.beforeEach(async ({ page }) => {
    // 1. Pin viewport.
    await page.setViewportSize({ width: 1280, height: 800 });
    // 2. Disable animations + reduce motion.
    await page.emulateMedia({ reducedMotion: 'reduce' });
    // 4. Pin Date.now() so relative-time strings are deterministic.
    await page.addInitScript(() => {
      const FIXED = new Date('2026-05-05T12:00:00Z').valueOf();
      Date.now = () => FIXED;
    });
  });

  test('home matches snapshot', async ({ page }) => {
    await page.goto('/');
    // 3. Wait for fonts to load before screenshotting. `waitForFunction` is more
    //    explicit than `evaluate(() => document.fonts.ready)` — it polls until the
    //    predicate is true and fails the test if the condition never holds.
    await page.waitForFunction(() => document.fonts.status === 'loaded');
    await expect(page).toHaveScreenshot('home.png', {
      maxDiffPixelRatio: 0.01,
      fullPage: true,
    });
  });

  test('pricing matches snapshot', async ({ page }) => {
    await page.goto('/pricing');
    await page.waitForFunction(() => document.fonts.status === 'loaded');
    await expect(page).toHaveScreenshot('pricing.png', {
      maxDiffPixelRatio: 0.01,
      fullPage: true,
    });
  });
});
```

First run with `pnpm exec playwright test --update-snapshots` creates the baseline images. Subsequent runs compare. When a CSS change shifts the layout, the test fails *and* writes a diff image to `test-results/` you can inspect.

> **visual regression** — automated detection of pixel-level layout/color changes between runs.
>
> **The four stability levers** — pin viewport, disable animations, wait for fonts, freeze time. Forgetting any one produces a flaky test, and a flaky test is worse than no test (it teaches the team to ignore failures).

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
  # Lock the pnpm action and version to your `package.json#packageManager` field
  # so CI and local dev never drift. As of May 2026, pnpm 10 is current; pin to
  # whatever the lockfile + packageManager specify, not whatever's "latest."
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

> **The dummy-env-var trap.** The boot validator (Ch 57) regex-checks each value. Make sure each CI dummy actually passes the check, or the e2e job fails before the first test runs:
>
> | Var | Regex | Dummy that passes |
> |---|---|---|
> | `STRIPE_SECRET_KEY` | `^sk_(test\|live)_` | `sk_test_dummy` |
> | `STRIPE_WEBHOOK_SECRET` | `^whsec_` | `whsec_dummy` |
> | `RESEND_API_KEY` | `^re_` | `re_dummy` |
> | `R2_ACCOUNT_ID` / `R2_ACCESS_KEY_ID` / `R2_SECRET_ACCESS_KEY` / `R2_BUCKET` | `minLength(1)` only | `dummy` |
>
> If you tighten any regex later, update both the validator and the CI dummies in the same PR.

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
    # `'Preview'` is capitalised — Vercel's deployment-event payload sends the
    # environment name in title case (`'Preview'`, `'Production'`). The match
    # is case-sensitive; verify against current Vercel docs before relying on it.
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

> *Today's job:* Streak is live at `streak.example.com` (or whatever you chose) with HTTPS, custom domain, real Postgres, real Stripe, real R2, real Resend. *Visible win:* you sign up on your own production URL with a real email.

A senior engineer treats first deploy as the *easy* part. The hard part is everything that *follows* — the runbook, the rollback plan, the env-var rotation, the on-call expectation. We set those up at deploy time, not afterward.

---

## Lesson 62.1 — `@sveltejs/adapter-vercel`

```bash
pnpm add -D @sveltejs/adapter-vercel
```

```ts
// svelte.config.ts
import adapter from '@sveltejs/adapter-vercel';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
import type { Config } from '@sveltejs/kit';

const config: Config = {
  preprocess: vitePreprocess(),
  kit: {
    adapter: adapter({
      runtime: 'nodejs22.x',
      regions: ['iad1'], // pick the region closest to your DB
      memory: 1024,
      maxDuration: 30,
    }),
  },
};

export default config;
```

Read aloud:

| Field | Read aloud as |
|---|---|
| `runtime: 'nodejs22.x'` | *"Run as a Node.js 22 serverless function."* (Node 22 is LTS as of Oct 2024 and current on Vercel as of May 2026; if Vercel adds a Node 24 LTS slot later, switch.) |
| `regions: ['iad1']` | *"Pin the function to the us-east-1 region — same region as the Postgres."* |
| `memory: 1024` | *"1 GB of RAM per invocation."* |
| `maxDuration: 30` | *"Hard-cap any one request at 30 seconds."* |

Pinning the region matters: if your function runs in `iad1` and your DB is in `eu-west-1`, every query crosses the Atlantic. Latency dies. Pick one region and stay there until you genuinely need geographic distribution.

---

## Lesson 62.2 — Vercel project setup

1. **Push to GitHub.** The repo is already there.
2. **Vercel → New Project → Import Git Repository.** Pick `streak`.
3. **Framework preset: SvelteKit** — Vercel auto-detects.
4. **Root directory: `./`**. Build command: `pnpm build`. Output: `.vercel/output` (auto).
5. **Environment Variables** — paste from `.env`, but **per environment**: Production, Preview, Development. Set each:
   - `DATABASE_URL` (production points to prod DB; preview points to a separate preview DB)
   - `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `RESEND_API_KEY`, `R2_*`
   - `PUBLIC_SITE_URL` (production: `https://streak.example.com`; preview: leave blank — Vercel auto-injects `VERCEL_URL`)
6. **Deploy.** First build takes 60–120 s. You get a `.vercel.app` URL.
7. Visit it. Sign up. Confirm.

---

## Lesson 62.3 — Database hosting

Two senior choices for serverless-friendly Postgres in May 2026:

- **[Neon](https://neon.tech)** — branchable; per-PR Postgres branches. Pooler URL recommended for serverless.
- **[Supabase](https://supabase.com)** — Postgres + auth + storage + realtime; you can use just the DB.

Both expose a PgBouncer-style pooler URL. Use the **pooled** URL for serverless functions, not the direct one — Vercel's serverless runtime spins up many short-lived processes that would exhaust direct connections.

Set `DATABASE_URL` to the *pooled* URL. Apply migrations against the *unpooled* URL (some DDL statements don't work over PgBouncer):

```bash
DATABASE_URL=$UNPOOLED_URL pnpm db:migrate
```

> **connection pooler** *(noun)* — a process that maintains a pool of long-lived connections to Postgres and lets short-lived clients (serverless functions) borrow them. Without one, serverless apps hit Postgres's `max_connections` limit fast.

The pooler earns its keep here, but file away one thing for later: it abstracts *which* backend serves a connection. The day you add a read replica for scale (Lesson 65.5a), that same abstraction will happily serve a read off a stale replica milliseconds after a write — and the code will look completely finished while doing it. Single primary today; the consistency reckoning comes when you split reads off.

---

## Lesson 62.4 — Custom domain and DNS

Buy a domain (Namecheap, Cloudflare, wherever). In Vercel project settings → Domains → Add `streak.example.com`. Vercel gives you the DNS records:

- `A` record on `streak.example.com` → `76.76.21.21` (Vercel's anycast IP), or
- `CNAME` on `www.streak.example.com` → `cname.vercel-dns.com`.

Add at your DNS provider. Wait 5–60 minutes for propagation. Vercel auto-provisions a Let's Encrypt cert.

Once HTTPS works, **wait two weeks** with HSTS active (you set the header in Ch 48 already). Then submit to **HSTS preload**:

```
https://hstspreload.org/?domain=streak.example.com
```

Once accepted, every major browser will refuse HTTP for your domain *forever*. Don't preload until you're sure HTTPS is solid; the preload list is hard to leave.

The HSTS header itself must include the `preload` directive (already in the Chapter 48 header — `Strict-Transport-Security: max-age=63072000; includeSubDomains; preload`). The preload list site (`hstspreload.org`) requires this directive plus a fully HTTPS redirect chain from the apex domain. The two-week wait is conservative — the real requirement is *"survive the 30-day max-age long enough to be confident HTTPS is solid before locking the door."* If you're certain on day three, submit on day three; if you're shaky on day fourteen, wait longer.

---

## Lesson 62.5 — The pre-launch checklist

`docs/checklists/launch.md`:

```markdown
# Streak — Launch Checklist

## Infrastructure
- [ ] All env vars set in production *and* preview (boot validator confirms)
- [ ] DB pooler URL used for `DATABASE_URL`
- [ ] DB migrations applied to production
- [ ] DB backup schedule confirmed (Neon auto-snaps; verify retention)
- [ ] Stripe webhook endpoint registered with prod URL + signing secret
- [ ] R2 bucket created; CORS configured for the production origin
- [ ] Resend domain verified (SPF + DKIM + DMARC records)

## Security
- [ ] All security headers present (`pnpm security:headers` against prod)
- [ ] HSTS preload submitted (after 2 weeks HTTPS-only)
- [ ] CSP audited; no `'unsafe-inline'` for scripts
- [ ] Rate-limits live on `/login` and `/signup`

## Observability
- [ ] Logger DSN active (Sentry or chosen vendor)
- [ ] Request IDs in every log line
- [ ] Vercel Analytics enabled
- [ ] On-call rotation defined (even if it's just you for now)
- [ ] Status page or runbook bookmark

## Content
- [ ] `robots.txt` allows what it should; disallows `/app/`, `/admin/`
- [ ] `sitemap.xml` validates (https://www.xml-sitemaps.com/validate-xml-sitemap.html)
- [ ] OpenGraph image renders correctly when the URL is shared
- [ ] Lighthouse: marketing pages 100/100/100/100

## API
- [ ] OpenAPI spec matches `/api/v1/*` (contract tests green)
- [ ] CORS allowlist for the API matches expected callers

## Operations
- [ ] Rollback runbook current (`docs/runbooks/rollback.md`)
- [ ] Secret-rotation runbook current (`docs/runbooks/secret-rotation.md`)
- [ ] DB-restore runbook drafted

## Sign-off
- [ ] You signed up on production with a real email; everything works.
```

The checklist is a *contract*. Going live before every box is ticked is how teams ship Friday-night incidents.

---

## Lesson 62.6 — First deploy ceremony

The literal sequence:

1. **PR is green.** All CI jobs passed.
2. **Tag a release.** `git tag v0.1.0 && git push origin v0.1.0` (or let `release-please` do it).
3. **Merge to main.** Vercel deploys to production.
4. **Smoke test.** Visit the production URL. Sign up. Log a habit. Logout. Login. Add a Stripe Pro subscription with the test card. Verify the webhook fired.
5. **Watch logs for 5 minutes.** Vercel → Logs → confirm no errors in the request stream.
6. **Tweet about it.** Optional but earned.

Senior habit: **don't ship Friday afternoon.** Ship Tuesday morning when you have the rest of the day to fix anything that breaks.

---

## Lesson 62.7 — Rollback runbook

`docs/runbooks/rollback.md`:

```markdown
# Rollback runbook

Triggered when production is broken and a fix-forward isn't fast.

## 1. Promote the previous deployment in Vercel
- Vercel dashboard → Deployments → find the last green deployment → "..." → Promote to Production.
- This takes ~30 seconds; users see no downtime.

## 2. Verify
- Open the production URL.
- Log in; verify the broken behaviour is gone.
- Check `/api/v1/health` returns 200.

## 3. Communicate
- If users were impacted >5 minutes, post in #status (or your status page).
- Open a GitHub issue tagged `incident`.

## 4. Post-mortem
- Within 24h, write a blameless post-mortem (template: `docs/post-mortems/_template.md`).
- Land a fix forward; add a regression test.
```

Test the rollback once *before* you need it. Senior pattern: **the runbook you've never tested doesn't work.**

> **The migration trap.** If the bad deploy ran a migration, rollback recovers code only — DB schema changes are *not* reverted by promoting an older deployment. The DB-restore runbook (`docs/runbooks/db-restore.md`) covers schema rollback via point-in-time recovery for the destructive case. For *additive* migrations (new columns, new tables) the rollback is safe because the old code ignores new columns; for *destructive* migrations (dropped columns, renamed tables, narrowed types) the rollback is broken until the schema is restored. This is the strongest argument for Bible rule #18 (forward-only) — *additive forever* makes rollback simple.

---

## Lesson 62.8 — Read this code

```ts
// svelte.config.ts
adapter: adapter({ runtime: 'edge' }),
```

When is this *wrong* for Streak?

<details>
<summary>Answer</summary>

`runtime: 'edge'` runs your function on Vercel's edge network — closer to users, but with a *different runtime* (Cloudflare Workers, V8 isolates) that doesn't support Node-only APIs.

Streak uses:
- **`@node-rs/argon2`** — native Node binding, doesn't run on edge.
- **`postgres-js`** — uses Node's `net` module.
- **Worker threads** — Node-only.

Edge runtime would silently break all three. We use `'nodejs22.x'` for Streak. `'edge'` is right for *some* read-heavy GET handlers (geo-aware redirects, content-only marketing pages); we'd opt in per-route via page-level `export const config = { runtime: 'edge' }`.
</details>

---

## Lesson 62.9 — Now you write it

**The English sentence first:**

> *"Add a `/api/v1/health` endpoint that returns 200 + a JSON body with `{ ok: true, version, time }` — used by the rollback runbook to verify a deployment."*

<details>
<summary>Worked answer</summary>

```ts
// src/routes/api/v1/health/+server.ts
import type { RequestHandler } from './$types';
import { json } from '@sveltejs/kit';

const VERSION = process.env.VERCEL_GIT_COMMIT_SHA?.slice(0, 7) ?? 'dev';

export const GET: RequestHandler = () => {
  return json({ ok: true, version: VERSION, time: new Date().toISOString() });
};
```

`VERCEL_GIT_COMMIT_SHA` is auto-injected by Vercel — gives you a commit SHA in the response so you can verify *which* version is live. Bookmark `https://streak.example.com/api/v1/health` and refresh after every deploy.
</details>

---

## Lesson 62.10 — Recurring concepts from earlier chapters

- **Boot validator** (Ch 57) — the deploy fails fast if any env var is wrong.
- **Security headers** (Ch 48) — applied at the edge via `handle`.
- **HSTS preload** — the two-week soak is the senior caution.
- **The runbook discipline** (Ch 57) — secret rotation and rollback both shipped before launch.

---

## Lesson 62.11 — What you can now read in the wild

After Chapter 62 you can:

- Read **`adapter-vercel`** options (`runtime`, `regions`, `memory`, `maxDuration`).
- Read **a launch checklist** and tell what's missing.
- Read **`docs/runbooks/*.md`** as senior operational artifacts.
- Tell **edge from node** runtimes and pick correctly per-route.
- Spot **direct DB connections in serverless** as a connection-exhaustion bug waiting.

---

## Glossary added in Chapter 62

| Term | Definition |
|---|---|
| `adapter-vercel` | SvelteKit adapter that builds for Vercel's serverless runtime. |
| connection pooler | Process that lends DB connections to short-lived clients. |
| HSTS preload | Permanent HTTPS-only listing in browsers. |
| smoke test | Quick post-deploy manual verification. |
| rollback runbook | Documented steps to revert a bad deploy. |

---

## End-of-chapter checkpoint

- [ ] Streak is live at a real URL.
- [ ] You signed up on your own production site.
- [ ] HSTS preload submitted (2 weeks after launch).
- [ ] Rollback runbook tested at least once.
- [ ] `/api/v1/health` returns the deployed commit SHA.

---

# Chapter 63 — Observability — structured logs, metrics, traces, on-call

> *Today's job:* every request emits a structured JSON log line with a request ID; Prometheus metrics expose latency + traffic + errors with bounded labels; you read a 200-line log dump and find a planted bug. *Visible win:* you point at a `p99 > 2s` line on a Grafana panel and trace it back through the request ID to the slow query.

The on-call engineer's mindset — *"the user said it's broken; my gates are green; my gates must be insufficient"* — is what this chapter installs.

---

## Lesson 63.1 — Why structured

`console.log('error: ' + message)` is *unstructured*. Greppable for a short string, useless for filtering by user, route, severity, or time window. **Structured logs are JSON, one object per line.**

```bash
pnpm add pino
```

```ts
// src/lib/logger.ts
import pino from 'pino';

// pino's `redact.paths` takes literal paths or single-level wildcards. Deep
// nesting needs explicit paths — `password` does NOT match `user.password`.
// Spell each nested path out, then add `*.password`-style wildcards for the
// common shallow shapes. Test with a sample log to confirm redaction fires
// (a debug `logger.info({ user: { password: 'x' } }, ...)` should print
// without the `password` field).
const SENSITIVE = [
  'user.password',
  'user.passwordHash',
  'token',
  '*.password',
  '*.passwordHash',
  'authorization',
  'cookie',
];

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',
  base: {
    app: 'streak',
    env: process.env.NODE_ENV ?? 'development',
    version: process.env.VERCEL_GIT_COMMIT_SHA?.slice(0, 7) ?? 'dev',
  },
  redact: { paths: SENSITIVE, remove: true },
  timestamp: pino.stdTimeFunctions.isoTime,
  formatters: {
    level: (label) => ({ level: label }),
  },
});
```

Read aloud:

| Field | Read aloud as |
|---|---|
| `level` | *"Log level — info or above by default."* |
| `base` | *"Every line includes app + env + commit SHA."* |
| `redact: { paths: SENSITIVE, remove: true }` | *"For each path in the deny-list, delete the field entirely from the output."* |
| `timestamp: ...isoTime` | *"ISO 8601 timestamps so log aggregators can sort."* |

The `SENSITIVE` deny-list is the application of Bible rule #15. The `'*.password'` glob form catches `user.password`, `body.password`, etc. — anywhere the field shows up in a nested object.

> **structured logging** *(noun)* — JSON-line logs with consistent fields. The format every modern aggregator (Datadog, Logflare, Better Stack, Grafana Loki) ingests directly.

---

## Lesson 63.2 — Request IDs

The bridge between user complaint ("error ID `f3a1`") and operator query ("show me logs for `f3a1`"). Wire it in `handle`:

```ts
// src/hooks.server.ts (extended)
import type { Handle } from '@sveltejs/kit';
import { sequence } from '@sveltejs/kit/hooks';
import { logger } from '$lib/logger';

const logHandle: Handle = async ({ event, resolve }) => {
  const requestId = event.request.headers.get('x-request-id') ?? crypto.randomUUID();
  event.locals.requestId = requestId;

  const start = Date.now();
  const log = logger.child({ requestId });
  event.locals.log = log;

  let response: Response;
  try {
    response = await resolve(event);
  } catch (err) {
    log.error({ err }, 'request.failed');
    throw err;
  }

  response.headers.set('x-request-id', requestId);
  log.info({
    method: event.request.method,
    path: event.url.pathname,
    status: response.status,
    durationMs: Date.now() - start,
    userId: event.locals.user?.id ?? null,
  }, 'request');
  return response;
};

export const handle = sequence(authHandle, logHandle /* , ... */);
```

Read aloud: *"For every request, mint a request ID — or accept one from the upstream proxy. Attach it to a child logger. Time the response. Log the request line with method, path, status, duration, user. Echo the ID in the response header so the client can quote it."*

Update `App.Locals` in `app.d.ts`:

```ts
declare global {
  namespace App {
    interface Locals {
      user?: { id: string; email: string; role: 'user' | 'admin' };
      requestId: string;
      log: import('pino').Logger;
    }
  }
}
```

Now any `event.locals.log.info(...)` inside actions, loads, or `+server.ts` handlers automatically carries the request ID. Senior pattern: **never pass `requestId` by hand** — let the request-scoped child logger do it.

---

## Lesson 63.3 — The four golden signals

Google's SRE book named four signals every service must monitor:

1. **Latency** — how long requests take (especially p50 and p99).
2. **Traffic** — how many requests per second.
3. **Errors** — what fraction fail.
4. **Saturation** — how full the system is (CPU, memory, queue depth, DB pool).

A senior dashboard has at least one panel for each. A senior alert fires when *any* of them crosses a threshold sustained across the alert window.

> **p99 latency** — the 99th-percentile latency. *"99% of requests complete in under X ms."* The median (`p50`) lies about user experience; `p99` tells the truth.

---

## Lesson 63.4 — Prometheus metrics with bounded labels

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
  help: 'Request latency in seconds',
  labelNames: ['method', 'route'] as const,
  buckets: [0.01, 0.05, 0.1, 0.25, 0.5, 1, 2, 5, 10],
});

export const dbQueryLatency = new Histogram({
  name: 'streak_db_query_duration_seconds',
  help: 'Database query latency',
  labelNames: ['operation'] as const,
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1, 5],
});

// Bounded label set — never accept raw user input as a label.
const KNOWN_ROUTES = new Set([
  '/',
  '/about',
  '/pricing',
  '/login',
  '/signup',
  '/dashboard',
  '/billing',
  '/admin/users',
  '/api/v1/habits',
  '/api/stripe/webhook',
]);

export function safeRouteLabel(path: string): string {
  if (KNOWN_ROUTES.has(path)) return path;
  // Collapse known dynamic shapes into a single bucket.
  if (path.startsWith('/habits/')) return '/habits/[id]';
  if (path.startsWith('/api/v1/habits/')) return '/api/v1/habits/[id]';
  // `'other'` is itself a label — attackers piling counts under it cause no
  // cardinality bomb because it's a fixed string. Bible #14 satisfied: every
  // label value comes from a static set.
  return 'other';
}

export function safeStatusClass(status: number): string {
  return `${Math.floor(status / 100)}xx`; // 2xx, 3xx, 4xx, 5xx
}

export { register };
```

Wire metrics into the log handle:

```ts
import { requestCounter, requestLatency, safeRouteLabel, safeStatusClass } from '$lib/metrics';

// ... inside logHandle, after `response = await resolve(event)`:
const route = safeRouteLabel(event.url.pathname);
const statusClass = safeStatusClass(response.status);
const durationS = (Date.now() - start) / 1000;

requestCounter.inc({ method: event.request.method, route, status_class: statusClass });
requestLatency.observe({ method: event.request.method, route }, durationS);
```

Why the bounded-label discipline matters: Prometheus stores **one time series per unique label combination**. If you used the raw `path` as a label and a malicious actor hit `/abc-123-def`, `/abc-124-def`, `/abc-125-def`, ..., you'd grow a million time series in seconds. Memory blows up. Scrape fails. Alerting goes dark. Prod dies. **This is a real production-incident class** — Bible rule #14.

> **cardinality explosion** *(noun)* — uncontrolled growth in unique time series due to high-cardinality labels. The single most common Prometheus failure mode in production.

---

## Lesson 63.5 — Exposing `/metrics`

```ts
// src/routes/_metrics/+server.ts
import { error } from '@sveltejs/kit';
import { timingSafeEqual } from 'node:crypto';
import { register } from '$lib/metrics';
import { METRICS_TOKEN } from '$env/static/private';
import type { RequestHandler } from './$types';

export const GET: RequestHandler = async ({ request }) => {
  // Gate with a shared token; only Prometheus scrapers should hit this.
  const auth = request.headers.get('authorization');
  if (auth === null || !auth.startsWith('Bearer ')) {
    throw error(401, 'Unauthorized');
  }
  const provided = auth.slice('Bearer '.length);

  // `timingSafeEqual` throws on different-length buffers. Length-mismatch is
  // already a "wrong token" so we reject without timing the comparison; the
  // only constant-time path runs on equal-length tokens.
  if (provided.length !== METRICS_TOKEN.length) {
    throw error(401, 'Unauthorized');
  }
  if (!timingSafeEqual(Buffer.from(provided), Buffer.from(METRICS_TOKEN))) {
    throw error(401, 'Unauthorized');
  }

  return new Response(await register.metrics(), {
    headers: { 'content-type': register.contentType },
  });
};
```

The `_metrics` route prefix (with leading underscore) is a senior convention for *"endpoint not for end users"*. Configure your Prometheus scraper to hit it with the bearer token:

```yaml
# prometheus.yml
scrape_configs:
  - job_name: streak
    metrics_path: /_metrics
    bearer_token: "<METRICS_TOKEN value>"
    static_configs:
      - targets: ['streak.example.com']
```

---

## Lesson 63.6 — OpenTelemetry tracing

In May 2026, SvelteKit ships **experimental tracing** that auto-instruments `handle`, `load`, actions, and remote functions. Enable in `svelte.config.ts`:

```ts
const config: Config = {
  kit: {
    adapter: adapter(/* ... */),
    experimental: {
      tracing: { server: true },
    },
  },
};
```

Add an OTLP exporter (or any compatible vendor):

```bash
pnpm add @opentelemetry/sdk-node @opentelemetry/exporter-trace-otlp-http \
         @opentelemetry/instrumentation-http @opentelemetry/instrumentation-fetch
```

```ts
// src/instrumentation.server.ts
import { NodeSDK } from '@opentelemetry/sdk-node';
import { OTLPTraceExporter } from '@opentelemetry/exporter-trace-otlp-http';
import { HttpInstrumentation } from '@opentelemetry/instrumentation-http';
import { FetchInstrumentation } from '@opentelemetry/instrumentation-fetch';

// Pin to HTTP + fetch only. `getNodeAutoInstrumentations()` includes packages
// that conflict with Vercel's serverless runtime (`fs`, `net`, `dns`) and
// silently inflate the function bundle. Explicit instrumentations are the
// senior pattern for serverless: opt in to what you need; nothing else.
const sdk = new NodeSDK({
  serviceName: 'streak',
  traceExporter: new OTLPTraceExporter({ url: process.env.OTEL_EXPORTER_OTLP_ENDPOINT }),
  instrumentations: [new HttpInstrumentation(), new FetchInstrumentation()],
});
sdk.start();
```

Now every request produces a *trace* — a tree of spans showing how time was spent: handle → load → db.query → external.fetch → render. When p99 spikes, you click the slowest trace and see *exactly* which span ate the budget.

> **trace** (in observability) *(noun)* — a tree of timed spans across one request. Distinguishes *"DB was slow"* from *"render was slow"* without guessing.

Sample at 10% in production to keep cost reasonable. Senior heuristic: 100% sampling for first month after launch (you need the data); ramp down once dashboards are stable.

---

## Lesson 63.7 — Alerts — when to wake someone up

A senior alert is **actionable, specific, and non-flaky**. Bad: *"errors > 0"*. Good: *"5xx error rate > 1% sustained for 5 minutes"*.

Three pages-the-on-call alerts for Streak:

1. **5xx rate > 1% over 5 min** — something's broken; the user notices.
2. **p99 latency > 2s over 5 min** — degraded experience.
3. **DB connection pool > 80% over 5 min** — saturation; an outage is approaching.

Three *non-paging* alerts (open a ticket, don't wake anyone):

1. **Successful signups < expected weekly average × 0.5** — funnel problem.
2. **Webhook processing lag > 5 min** — Stripe is OK; we're behind.
3. **Cert expiring in < 7 days** — renew it.

> **alert fatigue** — when on-call ignores alerts because too many fire. The death of an observability culture. *Alerts must be rare and meaningful.*

---

## Lesson 63.8 — Reading a prod log dump

Drill. Below is a synthetic 30-line log dump (real ones run thousands of lines; the principle's the same). Find the planted bugs.

```
{"level":"info","time":"2026-05-05T10:00:00Z","requestId":"a1","method":"GET","path":"/dashboard","status":200,"durationMs":42,"userId":"u1"}
{"level":"info","time":"2026-05-05T10:00:01Z","requestId":"a2","method":"POST","path":"/dashboard?/addHabit","status":200,"durationMs":78,"userId":"u1"}
{"level":"info","time":"2026-05-05T10:00:03Z","requestId":"a3","method":"GET","path":"/api/v1/habits","status":200,"durationMs":35,"userId":"u2"}
{"level":"info","time":"2026-05-05T10:00:12Z","requestId":"a4","method":"POST","path":"/login","status":200,"durationMs":201,"userId":null}
{"level":"info","time":"2026-05-05T10:01:33Z","requestId":"a5","method":"POST","path":"/login","status":429,"durationMs":3,"userId":null}
{"level":"error","time":"2026-05-05T10:02:01Z","requestId":"a6","method":"POST","path":"/billing?/upgrade","status":500,"durationMs":10042,"userId":"u3","err":{"name":"Error","message":"Stripe checkout.sessions.create timed out"}}
{"level":"error","time":"2026-05-05T10:02:14Z","requestId":"a7","method":"POST","path":"/billing?/upgrade","status":500,"durationMs":10039,"userId":"u4","err":{"name":"Error","message":"Stripe checkout.sessions.create timed out"}}
{"level":"error","time":"2026-05-05T10:02:25Z","requestId":"a8","method":"POST","path":"/billing?/upgrade","status":500,"durationMs":10044,"userId":"u5","err":{"name":"Error","message":"Stripe checkout.sessions.create timed out"}}
{"level":"info","time":"2026-05-05T10:03:01Z","method":"GET","path":"/dashboard","status":200,"durationMs":51,"userId":"u1"}
{"level":"info","time":"2026-05-05T10:04:00Z","requestId":"a9","method":"GET","path":"/dashboard","status":200,"durationMs":1840,"userId":"u9"}
{"level":"info","time":"2026-05-05T10:04:12Z","requestId":"a10","method":"GET","path":"/dashboard","status":200,"durationMs":2105,"userId":"u9"}
{"level":"info","time":"2026-05-05T10:04:30Z","requestId":"a11","method":"GET","path":"/dashboard","status":200,"durationMs":1998,"userId":"u9"}
```

What three problems do you spot?

<details>
<summary>Answer</summary>

1. **Three 500s in a row at 10:02 (lines 6–8) all from `/billing?/upgrade` with the same Stripe-timeout error.** The 10s `durationMs` matches the `timeout: 10_000` we set in Ch 49. Stripe's API is degraded. Action: post a status update; consider a circuit breaker; check Stripe's status page.

2. **Line 9 is missing `requestId`.** Every log line should have one (Lesson 63.2). This indicates a code path that bypasses the `logHandle` middleware — possibly a direct `console.log` someone forgot to remove. Action: grep for `console.log` in src/.

3. **Lines 10–12: dashboard requests for user `u9` taking 1.8–2.1 seconds.** That's a tail-latency outlier — most are <100 ms. Action: check that user's data; do they have unusually many habits triggering an N+1 query? Add an `EXPLAIN ANALYZE` for that query.

The senior on-call mindset: *every weird line is a signal*. You don't ignore them — you investigate or you file them.
</details>

---

## Lesson 63.9 — Read this code

```ts
metrics.requestCounter.inc({
  method: req.method,
  route: req.url, // ❌
  user_id: req.userId, // ❌
});
```

Two cardinality bombs. Find them.

<details>
<summary>Answer</summary>

1. **`route: req.url`** — the full URL includes query strings. `?q=foo`, `?q=bar`, `?q=...` each become unique label combinations. After a day, you have a million series.

2. **`user_id: req.userId`** — every user is a unique label value. With 100k users, that's 100k series per other-label-combination. Cardinality death.

The fix: `safeRouteLabel(pathname)` + drop `user_id` (it goes in *logs*, not *metrics*).
</details>

---

## Lesson 63.10 — Now you write it

**The English sentence first:**

> *"Add a `streak_signups_total{plan}` counter that increments after every successful signup, with `plan` bounded to `{free, pro}`. Drop the user's email — that would be a cardinality bomb."*

<details>
<summary>Worked answer</summary>

```ts
// src/lib/metrics.ts (add)
export const signupCounter = new Counter({
  name: 'streak_signups_total',
  help: 'Total successful signups',
  labelNames: ['plan'] as const,
});

const KNOWN_PLANS = new Set(['free', 'pro']);

export function safePlanLabel(plan: string): string {
  return KNOWN_PLANS.has(plan) ? plan : 'unknown';
}
```

In the signup action:

```ts
import { signupCounter, safePlanLabel } from '$lib/metrics';

// ... after successful signup:
signupCounter.inc({ plan: safePlanLabel('free') });
event.locals.log.info({ userId: created.id }, 'signup');
```

Email goes in the **log line** (where it can be searched per-user); plan goes in the **metric** (where the bounded label is safe).
</details>

---

## Lesson 63.11 — Recurring concepts from earlier chapters

- **`crypto.randomUUID()`** (Ch 9) — every request mints one.
- **Bible rule #14** — bounded label sets, applied as `safeRouteLabel`/`safePlanLabel`.
- **Bible rule #15** — pino's `redact` paths apply the deny-list.
- **`event.locals`** (Ch 45) — request-scoped logger lives there.

---

## Lesson 63.12 — What you can now read in the wild

After Chapter 63 you can:

- Read **`pino`** config with `redact`, `base`, `formatters`.
- Read **`logger.child({ requestId })`** as the per-request scoping pattern.
- Read **`Counter` / `Histogram` / `Gauge`** from prom-client and explain the difference.
- Read a **Grafana dashboard** with the four golden signals.
- Read an **OpenTelemetry trace** and identify the slow span.
- Spot a **cardinality bomb** in unbounded metric labels.

---

## Glossary added in Chapter 63

| Term | Definition |
|---|---|
| structured log | JSON-line log with consistent fields. |
| `pino` | Fast structured-logging library for Node. |
| request ID | UUID minted per request; carried through logs and the `x-request-id` header. |
| four golden signals | Latency, traffic, errors, saturation. |
| p99 latency | 99th-percentile latency; the truth-teller about user experience. |
| cardinality explosion | Unbounded growth in unique time series due to high-cardinality labels. |
| trace | Tree of timed spans across one request. |
| OpenTelemetry (OTel) | Vendor-neutral standard for traces / metrics / logs. |
| alert fatigue | When too many alerts make on-call ignore real ones. |

---

## End-of-chapter checkpoint

- [ ] Logs are JSON, structured, with request IDs.
- [ ] `/_metrics` serves Prometheus format, gated by token.
- [ ] OpenTelemetry traces export to your chosen backend.
- [ ] You debugged the planted log dump.
- [ ] You can articulate the four golden signals out loud.

---

# Chapter 64 — Public REST API, emails, cron

> *Today's job:* Streak has a versioned public REST API at `/api/v1/*` documented in OpenAPI; signups send a real verification email via Resend; a daily cron job sends streak-reminders. *Visible win:* `curl -H "Authorization: Bearer pat_..." https://streak.example.com/api/v1/habits` returns JSON; signing up triggers a real email; tomorrow at 09:00 UTC, cron fires.

This is the operational tail of Streak — the last set of capabilities before the principal-engineer chapter and graduation.

---

## Lesson 64.1 — REST API design

| Path | Method | Purpose |
|---|---|---|
| `/api/v1/habits` | `GET` | List habits (paginated) |
| `/api/v1/habits` | `POST` | Create a habit |
| `/api/v1/habits/[id]` | `GET` | Read one habit |
| `/api/v1/habits/[id]` | `PATCH` | Update a habit |
| `/api/v1/habits/[id]` | `DELETE` | Delete a habit |

Senior conventions:

- **Version in the path** (`/v1/`). Easier to reason about than headers; cache-friendly.
- **Cursor-based pagination**, never offset. Cursors don't shift when items are inserted/deleted between pages.
- **`PATCH` for partial updates**, `PUT` reserved for full replacement (we use `PATCH` here).
- **Plural resource names** (`/habits`, not `/habit`).
- **Lowercase paths**, hyphens not underscores (`/habit-categories`, not `/habitCategories`).

> **cursor pagination** — pass back a `cursor` token; client sends it on the next request. Stable across inserts/deletes. Offset pagination (`?page=3`) reorders items as the data shifts.

---

## Lesson 64.2 — Personal access tokens (PATs)

API auth via short tokens the user generates in Settings. Show once, store hashed.

```ts
// schema.ts additions
import { pgTable, uuid, text, timestamp } from 'drizzle-orm/pg-core';

export const personalAccessTokens = pgTable('personal_access_tokens', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  hashedToken: text('hashed_token').notNull().unique(),
  name: text('name').notNull(),
  scopes: text('scopes').array().notNull().default([]),
  lastUsedAt: timestamp('last_used_at', { withTimezone: true }),
  expiresAt: timestamp('expires_at', { withTimezone: true }),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

Creation flow:

```ts
// src/lib/pat.ts
import { randomBytes, createHash } from 'node:crypto';
import { db } from '$lib/db/client';
import { personalAccessTokens } from '$lib/db/schema';
import { eq, and, gt, isNull } from 'drizzle-orm';

const PREFIX = 'pat_';

export function newPat(): { token: string; hash: string } {
  const raw = randomBytes(32).toString('base64url');
  const token = `${PREFIX}${raw}`;
  const hash = sha256(token);
  return { token, hash };
}

function sha256(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export async function createPat(userId: string, name: string): Promise<{ token: string; id: string }> {
  const { token, hash } = newPat();
  const [row] = await db.insert(personalAccessTokens)
    .values({ userId, name, hashedToken: hash })
    .returning({ id: personalAccessTokens.id });
  if (row === undefined) throw new Error('failed to insert PAT');
  return { token, id: row.id };
}

export async function findUserByPat(token: string): Promise<{ userId: string; scopes: string[] } | null> {
  if (!token.startsWith(PREFIX)) return null;
  const hash = sha256(token);
  const now = new Date();
  const [row] = await db.select({
    userId: personalAccessTokens.userId,
    scopes: personalAccessTokens.scopes,
    lastUsedAt: personalAccessTokens.lastUsedAt,
  })
    .from(personalAccessTokens)
    .where(and(
      eq(personalAccessTokens.hashedToken, hash),
      or(
        isNull(personalAccessTokens.expiresAt),
        gt(personalAccessTokens.expiresAt, now),
      ),
    ))
    .limit(1);
  if (row === undefined) return null;

  // Touch lastUsedAt at most once per minute to avoid hammering the row's lock under high QPS.
  const ONE_MIN = 60_000;
  if (row.lastUsedAt === null || now.getTime() - row.lastUsedAt.getTime() > ONE_MIN) {
    void db.update(personalAccessTokens)
      .set({ lastUsedAt: now })
      .where(eq(personalAccessTokens.hashedToken, hash));
  }

  return { userId: row.userId, scopes: row.scopes };
}
```

We use `sha256` (not Argon2) for PATs because the token is high-entropy (256 bits) — the slow-hash defence against brute-force isn't needed when the input space is already astronomical. Argon2 is for *human-typed* passwords with low entropy.

> **personal access token (PAT)** — a long random string the user creates and pastes into clients to authenticate API calls. Equivalent to a password but for machines.

---

## Lesson 64.3 — RFC 7807 problem details

The standardised error shape for JSON APIs:

```ts
// src/lib/api/errors.ts
type Problem = {
  type: string;
  title: string;
  status: number;
  detail?: string;
  instance?: string;
};

export function problem(status: number, title: string, detail?: string): Response {
  const body: Problem = {
    type: 'about:blank',
    title,
    status,
    ...(detail !== undefined && { detail }),
  };
  return new Response(JSON.stringify(body), {
    status,
    headers: { 'content-type': 'application/problem+json' },
  });
}
```

Use across all API endpoints:

```ts
// inside any /api/v1/* handler:
if (!user) return problem(401, 'Unauthorized', 'Missing or invalid token');
if (!habit) return problem(404, 'Not Found', `No habit with id ${id}`);
```

> **RFC 7807** — IETF spec for JSON-shaped error bodies. Standard across mature APIs.

---

## Lesson 64.4 — A complete `/api/v1/habits` endpoint

```ts
// src/routes/api/v1/habits/+server.ts
import type { RequestHandler } from './$types';
import { json } from '@sveltejs/kit';
import * as v from 'valibot';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
// All of these are used somewhere in the file — the cursor logic uses `sql`,
// `and`, `eq`, `desc`; the user lookup in `findUserByPat` (in `pat.ts`) uses
// `lt`, `or`, `isNull`. Audit every `+server.ts` snippet for missing imports
// before copy-paste; an unimported helper is the most common copy-paste bug.
import { sql, and, eq, lt, or, isNull, desc } from 'drizzle-orm';
import { findUserByPat } from '$lib/pat';
import { problem } from '$lib/api/errors';
import { addHabitForUser } from '$lib/habits-server';

const PAGE_SIZE = 50;

const ListQuery = v.object({
  cursor: v.optional(v.string()),
});

const CreateBody = v.object({
  name: v.pipe(v.string(), v.trim(), v.minLength(1), v.maxLength(100)),
  description: v.optional(v.pipe(v.string(), v.maxLength(500))),
});

async function authUser(request: Request, requiredScope?: string): Promise<{ userId: string; scopes: string[] } | Response> {
  const auth = request.headers.get('authorization');
  if (auth === null || !auth.startsWith('Bearer ')) {
    return problem(401, 'Unauthorized', 'Missing Bearer token');
  }
  const token = auth.slice(7);
  const result = await findUserByPat(token);
  if (result === null) return problem(401, 'Unauthorized', 'Invalid or expired token');
  if (requiredScope !== undefined && !result.scopes.includes(requiredScope)) {
    return problem(403, 'Forbidden', `Missing scope: ${requiredScope}`);
  }
  return result;
}

export const GET: RequestHandler = async ({ request, url }) => {
  const auth = await authUser(request, 'habits:read');
  if (auth instanceof Response) return auth;
  const { userId } = auth;

  // Cursor pagination tie-broken by id — handles equal-millisecond inserts.
  const cursor = url.searchParams.get('cursor');
  let cursorTs: Date | null = null;
  let cursorId: string | null = null;
  if (cursor !== null) {
    const [tsStr, id] = cursor.split(':', 2);
    const ts = Number(tsStr);
    if (!Number.isFinite(ts) || id === undefined) {
      return problem(400, 'Bad Request', 'Invalid cursor');
    }
    cursorTs = new Date(ts);
    cursorId = id;
  }

  // (created_at, id) < (cursor.ts, cursor.id) — Postgres row comparison.
  // For two rows with identical timestamps, id breaks the tie deterministically.
  const where = cursorTs !== null && cursorId !== null
    ? and(
        eq(habits.userId, userId),
        sql`(${habits.createdAt}, ${habits.id}) < (${cursorTs}, ${cursorId})`,
      )
    : eq(habits.userId, userId);

  const rows = await db.select()
    .from(habits)
    .where(where)
    .orderBy(desc(habits.createdAt), desc(habits.id))
    .limit(PAGE_SIZE + 1);

  const hasMore = rows.length > PAGE_SIZE;
  const page = hasMore ? rows.slice(0, PAGE_SIZE) : rows;
  const last = page[page.length - 1];
  // Cursor format: `<unix-ms>:<uuid>`. The decoder splits on `:`; we rely on
  // UUIDs containing no colons. If we ever change the ID type (e.g. to
  // base32-with-dashes), update both the encoder and the decoder.
  const nextCursor = hasMore && last !== undefined
    ? `${last.createdAt.getTime()}:${last.id}`
    : null;

  return json({
    data: page,
    cursor: nextCursor,
  });
};

export const POST: RequestHandler = async ({ request }) => {
  const auth = await authUser(request, 'habits:write');
  if (auth instanceof Response) return auth;
  const { userId } = auth;

  const raw: unknown = await request.json();
  const parsed = v.safeParse(CreateBody, raw);
  if (!parsed.success) {
    return problem(400, 'Bad Request', parsed.issues[0]?.message ?? 'Invalid body');
  }

  const result = await addHabitForUser(userId, parsed.output.name);
  if (!result.ok) return problem(409, 'Conflict', `Habit limit reached`);

  return json({ id: result.value.id }, { status: 201 });
};
```

Three senior touches in the cursor logic:

1. **Composite cursor `<ts>:<id>`** — two habits inserted in the same millisecond are rare in practice but real. A pure timestamp cursor loses one when paginating across the boundary. Row comparison `(created_at, id) < (cursor)` is Postgres's idiomatic fix.
2. **Scope check on the PAT** — `'habits:read'` for GET, `'habits:write'` for POST. Without this, every PAT is admin.
3. **`isNull(expiresAt) OR expiresAt > now`** in `findUserByPat` — expired tokens are rejected, not just lurking in the table.

For `findUserByPat` (which uses `or` and `isNull` for the expiry check), the imports look like:

```ts
import { and, eq, gt, isNull, or } from 'drizzle-orm';
```

Together, the two files (`+server.ts` for the API and `pat.ts` for the auth helper) have everything they need. No mid-snippet import surprises.

Read aloud:

| Line | Read aloud as |
|---|---|
| `authUser` returns `string \| Response` | *"Either a user-ID string (auth succeeded), or a Response object (auth failed and we want to return that response immediately)."* |
| `if (userId instanceof Response) return userId;` | *"If auth returned a Response, short-circuit out."* |
| `lt(habits.createdAt, new Date(cursorTime))` | *"Cursor pagination: 'less than the cursor's timestamp', plus desc ordering."* |
| `rows.slice(0, PAGE_SIZE + 1)` then `hasMore` | *"Fetch one extra row; if we got it, there's another page."* |

This is a real, runnable endpoint. Test it with `curl`:

```bash
TOKEN="pat_..."
curl -H "Authorization: Bearer $TOKEN" https://streak.example.com/api/v1/habits
curl -H "Authorization: Bearer $TOKEN" -H "Content-Type: application/json" \
     -d '{"name": "Read"}' https://streak.example.com/api/v1/habits
```

---

## Lesson 64.5 — OpenAPI spec

`docs/openapi.yaml` (excerpt):

```yaml
openapi: 3.1.0
info:
  title: Streak API
  version: 1.0.0
servers:
  - url: https://streak.example.com/api/v1
security:
  - bearerAuth: []
components:
  securitySchemes:
    bearerAuth:
      type: http
      scheme: bearer
  schemas:
    Habit:
      type: object
      required: [id, name, createdAt]
      properties:
        id: { type: string, format: uuid }
        name: { type: string, minLength: 1, maxLength: 100 }
        description: { type: string, maxLength: 500 }
        createdAt: { type: string, format: date-time }
    HabitList:
      type: object
      required: [data]
      properties:
        data:
          type: array
          items: { $ref: '#/components/schemas/Habit' }
        cursor:
          type: string
          nullable: true
    Problem:
      # RFC 7807 — every error response uses this shape.
      type: object
      required: [type, title, status]
      properties:
        type: { type: string, format: uri, default: 'about:blank' }
        title: { type: string }
        status: { type: integer }
        detail: { type: string }
        instance: { type: string, format: uri }
  responses:
    BadRequest:
      description: Invalid request
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
    Unauthorized:
      description: Missing or invalid token
      content:
        application/problem+json:
          schema: { $ref: '#/components/schemas/Problem' }
paths:
  /habits:
    get:
      summary: List habits
      parameters:
        - name: cursor
          in: query
          schema: { type: string }
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema: { $ref: '#/components/schemas/HabitList' }
        '401':
          $ref: '#/components/responses/Unauthorized'
    post:
      summary: Create a habit
      requestBody:
        required: true
        content:
          application/json:
            schema:
              type: object
              required: [name]
              properties:
                name: { type: string }
                description: { type: string }
      responses:
        '201':
          description: Created
          content:
            application/json:
              schema:
                type: object
                properties:
                  id: { type: string }
        '400':
          $ref: '#/components/responses/BadRequest'
```

The contract test from Chapter 59 validates the live API against this spec. Drift is caught immediately.

---

## Lesson 64.6 — CORS

If your API is going to be called from third-party browsers (a mobile app you don't control, a partner's web app), set CORS:

```ts
// inside any /api/v1/* handler:
import { dev } from '$app/environment';

const ALLOWED_ORIGINS = dev
  ? ['*']
  : ['https://streak.example.com', 'https://app.partner.com'];

function corsHeaders(origin: string | null): HeadersInit {
  if (origin === null) return {};
  const allowed = ALLOWED_ORIGINS.includes(origin) || ALLOWED_ORIGINS.includes('*');
  if (!allowed) return {};
  return {
    'Access-Control-Allow-Origin': origin,
    'Access-Control-Allow-Methods': 'GET, POST, PATCH, DELETE, OPTIONS',
    'Access-Control-Allow-Headers': 'Authorization, Content-Type',
    'Access-Control-Max-Age': '86400',
  };
}

export const OPTIONS: RequestHandler = async ({ request }) => {
  return new Response(null, { status: 204, headers: corsHeaders(request.headers.get('origin')) });
};
```

Allowlist explicitly. Wildcards are fine in dev; never in prod.

---

## Lesson 64.7 — Resend for email

```bash
pnpm add resend
```

```ts
// src/lib/mail/client.ts
import { Resend } from 'resend';
import { RESEND_API_KEY } from '$env/static/private';

export const resend = new Resend(RESEND_API_KEY);
```

Sending:

```ts
// src/lib/mail/send.ts
import { resend } from './client';
import { logger } from '$lib/logger';

const FROM = 'Streak <noreply@streak.example.com>';

export async function sendVerificationEmail(to: string, link: string): Promise<void> {
  const { error } = await resend.emails.send({
    from: FROM,
    to,
    subject: 'Verify your email',
    html: `
      <p>Welcome to Streak.</p>
      <p>Click to verify: <a href="${link}">${link}</a></p>
      <p>This link expires in 24 hours.</p>
    `,
  });
  if (error !== null && error !== undefined) {
    logger.error({ err: error, to }, 'mail.failed.verify');
    throw new Error('Email send failed');
  }
}
```

We never use `==` (Bible rule #5b). The explicit `!== null && !== undefined` is verbose but matches the Bible — `error == null` is the one-liner that catches both, and it's banned. `?? null` is the one-line shortcut where the value is *used* rather than *checked* (`const x = maybe ?? null`).

To deliver to inboxes (not spam folders), set up SPF, DKIM, and DMARC records on your domain. Resend's dashboard walks you through it.

---

## Lesson 64.8 — Email idempotency

A user retries signup or a network blip causes a duplicate; the verification email shouldn't go out twice. Track sends:

```ts
export const emailSendLog = pgTable('email_send_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  recipient: text('recipient').notNull(),
  kind: text('kind', { enum: ['verify', 'reminder', 'reset', 'summary'] }).notNull(),
  sentAt: timestamp('sent_at', { withTimezone: true }).notNull().defaultNow(),
  // dedupe key: same (recipient, kind, contextId) won't re-send for an hour
  dedupeKey: text('dedupe_key').notNull().unique(),
});
```

Around the actual send:

```ts
export async function sendOnce(
  kind: 'verify' | 'reminder' | 'reset' | 'summary',
  recipient: string,
  dedupeKey: string,
  fn: () => Promise<void>,
): Promise<void> {
  const inserted = await db.insert(emailSendLog)
    .values({ recipient, kind, dedupeKey })
    .onConflictDoNothing()
    .returning();
  if (inserted.length === 0) return; // already sent
  await fn();
}
```

Call sites pass the recipient explicitly so the dedupe row records who the email went to, e.g. `await sendOnce('verify', user.email, \`verify:${user.id}\`, async () => sendVerificationEmail(user.email, link))`.

Senior pattern: emails to external providers are *external side effects* — Bible rule #12 applies.

---

## Lesson 64.9 — Cron via `vercel.json`

```json
{
  "crons": [
    { "path": "/api/cron/daily-reminders", "schedule": "0 9 * * *" },
    { "path": "/api/cron/weekly-summary", "schedule": "0 14 * * 0" }
  ]
}
```

Cron expressions: minute / hour / day-of-month / month / day-of-week. `0 9 * * *` is "every day at 09:00 UTC".

Handler:

```ts
// src/routes/api/cron/daily-reminders/+server.ts
import type { RequestHandler } from './$types';
import { error } from '@sveltejs/kit';
import { CRON_SECRET } from '$env/static/private';
import { sendOnce } from '$lib/mail/send';
import { logger } from '$lib/logger';

export const GET: RequestHandler = async ({ request }) => {
  // Vercel sends `Authorization: Bearer <CRON_SECRET>` for crons. Headers are
  // case-insensitive in fetch's API, but the canonical form is `Authorization`;
  // tighten for readability so reviewers don't wonder if we meant `lowercase`.
  const auth = request.headers.get('Authorization');
  if (auth !== `Bearer ${CRON_SECRET}`) error(401, 'Unauthorized');

  const today = new Date().toISOString().slice(0, 10); // YYYY-MM-DD
  // ... fetch users who haven't logged today, send reminders, with sendOnce(...)
  logger.info({ date: today }, 'cron.daily-reminders.start');

  // ... actual sending logic ...

  logger.info({ date: today }, 'cron.daily-reminders.done');
  return new Response('ok');
};
```

The `Authorization` check is critical — without it, anyone hitting `/api/cron/daily-reminders` would trigger your cron. Bible rule #12 again: idempotency, dedupe, auth.

> **CRON_SECRET** — random string set in Vercel env vars; Vercel uses it to authenticate cron invocations.

---

## Lesson 64.10 — Read this code

```ts
const auth = request.headers.get('authorization');
if (auth !== `Bearer ${CRON_SECRET}`) error(401);
```

Two issues. Find them.

<details>
<summary>Answer</summary>

1. **`auth !== \`Bearer ${CRON_SECRET}\`` is a non-constant-time comparison.** In theory, an attacker could time the comparison to leak characters of `CRON_SECRET` byte by byte. Use `crypto.timingSafeEqual` (Ch 44):

```ts
import { timingSafeEqual } from 'node:crypto';
const expected = `Bearer ${CRON_SECRET}`;
if (auth === null) error(401);
const a = Buffer.from(auth);
const b = Buffer.from(expected);
if (a.length !== b.length || !timingSafeEqual(a, b)) error(401);
```

2. **No `null` check.** If `auth` is `null` (header missing), the `!==` is `true`, but a more explicit check tells the reader you thought about it.
</details>

---

## Lesson 64.11 — Now you write it

**The English sentence first:**

> *"Add a `weekly-summary` cron that fires Sundays at 14:00 UTC. For each Pro user who logged at least one habit in the past 7 days, send a summary email with the count. Use `sendOnce` so a retry doesn't double-send."*

<details>
<summary>Worked answer (sketch)</summary>

```ts
// src/routes/api/cron/weekly-summary/+server.ts
import type { RequestHandler } from './$types';
import { error } from '@sveltejs/kit';
import { CRON_SECRET } from '$env/static/private';
import { db } from '$lib/db/client';
import { users, habits, subscriptions } from '$lib/db/schema';
import { and, eq, gte } from 'drizzle-orm';
import { sendOnce } from '$lib/mail/send';
import { resend } from '$lib/mail/client';

export const GET: RequestHandler = async ({ request }) => {
  if (request.headers.get('authorization') !== `Bearer ${CRON_SECRET}`) error(401);

  const sevenDaysAgo = new Date(Date.now() - 7 * 24 * 60 * 60 * 1000);
  const weekKey = sevenDaysAgo.toISOString().slice(0, 10);

  const proUsers = await db.select({ id: users.id, email: users.email })
    .from(users)
    .innerJoin(subscriptions, eq(subscriptions.userId, users.id))
    .where(eq(subscriptions.plan, 'pro'));

  for (const user of proUsers) {
    const userHabits = await db.select()
      .from(habits)
      .where(and(eq(habits.userId, user.id), gte(habits.createdAt, sevenDaysAgo)));
    if (userHabits.length === 0) continue;

    await sendOnce(
      'summary',
      user.email,
      `weekly:${user.id}:${weekKey}`,
      async () => {
        await resend.emails.send({
          from: 'Streak <hi@streak.example.com>',
          to: user.email,
          subject: `Your week: ${userHabits.length} habits`,
          html: `<p>You logged ${userHabits.length} habits this week. Keep the streak.</p>`,
        });
      },
    );
  }

  return new Response('ok');
};
```

Idempotent (`sendOnce` keyed per user per week). Auth-gated. Logs not shown but should be added at start/end. Senior shape.

**Scaling note.** For 100 users, sequential `await`s are fine. For 100k users, this loop times out — Vercel cron `maxDuration: 30s` is the hard ceiling, and serial sends can't fit. Two senior moves: (a) `Promise.all` with a concurrency limit (`p-limit` keeps the pool at, say, 10) so latency overlaps; (b) batch the work into a queue (Inngest, Trigger.dev, or your own SQS-shaped table) and let a worker drain it asynchronously. Chapter 65's capacity-planning section covers the math; for the brief, the rule is *"if the loop has > 1k iterations, reach for a queue."*
</details>

---

## Lesson 64.12 — Recurring concepts from earlier chapters

- **`+server.ts`** (Ch 33, 50) — REST endpoints same as webhook handlers.
- **Valibot for input parsing** (Ch 41, 51) — every body parsed.
- **`Result<T, E>`** (Ch 27) — `addHabitForUser` returns one; mapped to HTTP codes.
- **Idempotency-keyed inserts** (Ch 50) — webhooks and email sends use the same dedupe pattern.
- **`crypto.timingSafeEqual`** (Ch 44) — every secret comparison is constant-time.

---

## Lesson 64.13 — What you can now read in the wild

After Chapter 64 you can:

- Read **`/api/vN/...`** as a versioned REST API and tell good from bad versioning.
- Read **cursor pagination** vs offset and explain the trade-off.
- Read **PATs** with hashed-at-rest storage and SHA-256 (because tokens are high-entropy).
- Read **RFC 7807 problem+json** error responses.
- Read **OpenAPI 3.x** specs and trace a path-method to its handler.
- Read **CORS preflight** handlers (`OPTIONS`).
- Read **`vercel.json` cron** entries and explain the schedule.
- Spot **non-constant-time secret comparisons** in cron handlers.

---

## Glossary added in Chapter 64

| Term | Definition |
|---|---|
| cursor pagination | Token-based pagination stable across mutations. |
| PAT | Personal access token; user-generated machine credential. |
| RFC 7807 | Standard JSON shape for API error responses. |
| OpenAPI | Spec format for documenting REST APIs (formerly Swagger). |
| CORS | Cross-Origin Resource Sharing; browser policy for cross-domain requests. |
| `vercel.json` crons | Vercel's scheduled-handler config. |
| email idempotency | Track sends to prevent duplicates on retry. |

---

## End-of-chapter checkpoint

- [ ] `/api/v1/habits` GET + POST work with PAT auth.
- [ ] `docs/openapi.yaml` exists and the contract test is green.
- [ ] Real verification email arrives in your inbox after signup.
- [ ] Daily-reminders cron handler exists and is auth-gated.
- [ ] `email_send_log` dedupes a manually-replayed cron.

---

# Chapter 65 — Code review, architecture, scaling, ADRs, the boring-tech doctrine

> *Today's job:* you do a structured 600-line code review and produce a written report; you draft a one-page scaling plan for *"at 100k users, where does Streak break first?"*; you write ADR-001 explaining *"why Postgres, not Mongo."* *Visible win:* three artifacts in `docs/` that show principal-engineer-level thinking — the kind a hiring committee or a CTO would weight heavily.

This is the chapter where *technical fluency* becomes *engineering judgment*. By the end, you can review code, plan capacity, and document decisions at staff/principal level.

---

## Lesson 65.1 — The structured-review framework

A senior engineer reviews a PR through eight lenses, in order. Skip none. Each lens has its own concrete questions:

| Lens | Questions to ask |
|---|---|
| **Correctness** | Does this match the spec? Off-by-one? Boundary conditions (zero, one, max)? Race conditions? |
| **Security** | New attack surface? Inputs validated? Secrets logged? Auth checks present? RBAC honored? |
| **Performance** | New query without an index? N+1? Bundle bloat? Hot path with a sync I/O call? |
| **Maintainability** | Names true? Comments match code? Repeated logic? Public API stable? |
| **Accessibility** | ARIA labels? Keyboard navigation? Contrast? Screen-reader announcements? |
| **Observability** | Errors logged? Metrics? Trace breadcrumbs? Useful at 3am? |
| **Tests** | Unit + integration + e2e where appropriate? Tests aligned with the change's blast radius? |
| **Docs** | API change in OpenAPI? Runbook updated? CHANGELOG mentions it? |

The senior practice: **scan the diff once for shape (file count, lines, scope creep), then walk each lens.** Drop comments per lens with concrete suggestions, not vague concerns.

---

## Lesson 65.2 — Comment style

Reviews live or die by tone. Senior comments:

- **Specific** — *"`xs[0]` is `T | undefined` under `noUncheckedIndexedAccess`. Either narrow with `if (xs[0] === undefined) return;` or use `xs.at(0)`."*
- **Cite the rule** — *"Bible rule #11: SELECT-then-UPDATE is a TOCTOU. Replace with atomic `UPDATE … WHERE … RETURNING`."*
- **Suggest, don't dictate** — *"Have you considered extracting `parseFooBody` so the integration tests can hit the validator directly?"*
- **Use prefixes** to weight your comments:
  - `BLOCKING:` — must fix before merge.
  - `nit:` — nitpick; author can ignore without explanation.
  - `consider:` — alternative for the author to weigh.
  - `praise:` — yes, call out good code; reviewers who only criticise burn out the team.

---

## Lesson 65.3 — The 600-line PR fixture

Below is a deliberately-flawed PR. **Read it carefully. Find every issue.** Don't peek at the answers.

The PR description: *"Adds a `/admin/reset-password` route. Admin can paste a user email and trigger a password reset. The user gets a magic-link email."*

```ts
// src/routes/(app)/admin/reset-password/+page.server.ts
import type { Actions } from './$types';
import { db } from '$lib/db/client';
import { users, passwordResets } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { sendResetEmail } from '$lib/mail/send';
import { recordCounter } from '$lib/metrics';

export const actions: Actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = data.get('email') as string;
    console.log(`Admin reset for ${email}, password was: ${data.get('admin_password')}`);

    const [user] = await db.select().from(users).where(eq(users.email, email));
    if (!user) return { success: true };

    const existing = await db.select().from(passwordResets).where(eq(passwordResets.userId, user.id));
    if (existing.length > 0) {
      await db.delete(passwordResets).where(eq(passwordResets.userId, user.id));
    }

    const token = Math.random().toString(36).slice(2);
    await db.insert(passwordResets).values({ userId: user.id, token, expiresAt: new Date(Date.now() + 86400000) });

    recordCounter.inc({ kind: 'reset', email });

    while (true) {
      try {
        await sendResetEmail(user.email, `https://streak.example.com/reset?t=${token}`);
        break;
      } catch (e) {
        console.log('retrying email');
      }
    }

    return { success: true };
  },
};
```

```svelte
<!-- src/routes/(app)/admin/reset-password/+page.svelte -->
<script>
  let { form } = $props();
</script>

<form method="POST">
  <input name="email" type="text" />
  <input name="admin_password" type="password" />
  <button>Reset</button>
</form>

<img src="/admin-banner.png" />
```

```sql
-- migrations/0042_drop_legacy_column.sql
ALTER TABLE users DROP COLUMN legacy_email;
```

How many BLOCKING issues do you find? Take your time; write them down with citations.

<details>
<summary>The full review (12 BLOCKING + 4 nits)</summary>

**BLOCKING:**

1. **No auth check.** `/admin/reset-password` is in `(app)/admin/`, but the page itself doesn't call `requireRole(event, 'admin')`. We rely on `(app)/admin/+layout.server.ts`, but if that layout is missing or misconfigured, this leaks. Add an explicit `requireRole(event, 'admin')` in the action body. **Bible rule #19** (defence in depth).

2. **`console.log` of `admin_password`.** Logs the admin's password to the console, which ships to the structured logger and from there to your log aggregator and possibly third parties. **Bible rule #15.** Remove.

3. **`data.get('email') as string`.** `FormData.get` returns `string | File | null`. Casting blindly will yield `null` cast as `string` if the field is missing. Validate with Valibot. **Bible rule #3** (no `as` casts that lie).

4. **Account enumeration.** The action returns `{ success: true }` immediately if user is missing — but the UI shows "Email sent" only if it didn't throw. An attacker could time the response to deduce which emails exist. Also: any visible error (unhandled exception) leaks "user does not exist." Make the path constant-time and behave identically. **Bible rule #19** + Ch 44 threat model.

5. **`Math.random()` for the token.** Not cryptographically secure. Use `crypto.randomBytes(32).toString('base64url')`. Bible rule echo: never invent crypto.

6. **SELECT-then-DELETE pattern.** Two queries instead of one atomic delete. Should be `db.delete(...).where(eq(passwordResets.userId, user.id))` directly — DELETE on no rows is a noop. The select wastes a round-trip. **Performance lens.**

7. **Cardinality bomb in metrics.** `recordCounter.inc({ kind: 'reset', email })` — every email is a unique label. **Bible rule #14.** Drop the email; put it in the *log line*, not the metric.

8. **Unbounded retry loop.** `while (true) ... sendResetEmail(...)` will spin forever if Resend is down. Catastrophic at scale. Bound the retries; back off; eventually fail loud. **Bible rule echo:** every external call needs a budget.

9. **Email is not idempotent.** Hitting submit twice fires two emails. Use `sendOnce` from Ch 64 with a key like `reset:${user.id}:${date}`. **Bible rule #12.**

10. **No audit row.** Admin actions on user accounts must `withAudit`. **Bible rule echo, Ch 47.**

11. **No CSRF protection mentioned, but `admin_password` is in the form.** Why is the admin re-authing via a form field? Sessions exist. Re-auth should use the session's `fresh` flag (Ch 44), not a re-typed password through a form field — and definitely not unencrypted in form data being logged. **Scope-honesty caveat:** `fresh` is set on login (Ch 44.4) and never refreshed on action. To use it for re-auth gating, add a `requireFresh(event)` helper that throws redirect to `/auth/reauth` if `fresh` is `false`. The book doesn't ship the gate; the planted bug here pretends it does. Treat this review note as *"this is what should exist; verify the codebase actually has it."*

12. **Migration drops a column with no deprecation period.** `ALTER TABLE users DROP COLUMN legacy_email;` — if any code still reads it, deploy breaks. **Bible rule #18:** migrations forward-only; column removals require a two-deploy dance (stop reading → next deploy drops). Also missing `IF EXISTS`.

**nits:**

1. **`<img src="/admin-banner.png" />`** — no `width`, `height`, or `alt`. **Bible rule #16** + accessibility.
2. **`<input name="email" type="text" />`** — should be `type="email"`, with `autocomplete="email"`.
3. **`<button>` has no `type`** — defaults to `type="submit"` inside a `<form>`, which is fine here, but **Bible rule #5** says always be explicit.
4. **Action returns `{ success: true }` for "email sent" but doesn't communicate to the UI that the email actually went out.** No toast. Silent UX.

The senior reviewer leaves all 16 comments in the PR. The BLOCKING ones must be fixed before merge; the nits are author's call.
</details>

---

## Lesson 65.4 — Architecture diagrams

A senior PR for any non-trivial change includes a Mermaid diagram in the description. Streak's full architecture, hand-drawn:

```mermaid
flowchart LR
  Browser --> Vercel[Vercel Edge / Functions]
  Vercel --> SK[SvelteKit handler]
  SK --> Pool[PgBouncer pool]
  Pool --> PG[(Postgres / Neon)]
  SK --> Stripe[Stripe API]
  SK --> Resend[Resend API]
  SK --> R2[(Cloudflare R2)]
  Stripe -- webhooks --> Vercel
  Vercel -- cron --> SK
  SK -. metrics .-> Prom[Prometheus]
  SK -. logs .-> Logs[Log aggregator]
  SK -. traces .-> OTel[OpenTelemetry collector]
```

When you propose a new external service, the diagram updates first, then the code. Review what's *drawn* before you review what's *coded*.

---

## Lesson 65.5 — Capacity planning math

Back-of-envelope sizing. *"At 100k users with 30 habits each, where does Streak break first?"*

**Storage:**
- 100k users × 30 habits × ~200 bytes/habit = 600 MB. Postgres can handle that on the cheapest tier.
- Audit log: ~5 events/user/day × 100k × 200 bytes = 100 MB/day. After a year, 36 GB. **First scaling problem: audit_log table partitioning** (Postgres declarative partitioning by month).

**Traffic:**
- 100k DAU × 5 sessions/day × 10 requests/session = 5M requests/day = ~60 RPS average, ~300 RPS peak (5× factor).
- Each request: median ~50ms, p99 ~500ms.
- 300 RPS × 50ms = 15 concurrent requests; 300 RPS × 500ms = 150 concurrent at p99. **Vercel's serverless scales horizontally; Postgres connections become the bottleneck.**

**Database:**
- Pooled URL allows ~500 concurrent connections.
- 150 in-flight × ~3 queries/request = 450 in-use connections at p99 peak. **Margin is thin; connection-pool exhaustion is the next failure mode.**
- Mitigation: read-only replica for `/api/v1/habits GET` traffic (high read volume; doesn't need primary). **But a replica is stale by milliseconds-to-seconds — see Lesson 65.5a before you ship it, or a user will watch their own write vanish.**

**External services:**
- Stripe: rate-limited at 100 RPS by default. Streak's billing actions are <1 RPS. **No issue.**
- Resend: free tier is 100 emails/day; paid is unlimited. At 100k users with daily reminders, that's 100k emails/day. **Need paid tier; need to batch.**

**The summary** (drop into ADR-005 or a `docs/scaling.md`):

> *At 100k users, the first three problems are (1) audit_log size — partition by month at 10 GB; (2) DB connection pool at 80% utilisation peaks — add a read replica for the API (which buys a read-after-write consistency requirement, not just capacity — Lesson 65.5a); (3) Resend pricing — move to paid tier and batch reminders. Storage and per-request latency are not a concern. Vercel scales horizontally; we don't hit a function-runtime ceiling at this size.*

> **back-of-envelope** *(idiom)* — quick capacity math done from rough numbers, in the head or on a napkin. The senior pre-flight before optimising for problems that aren't real yet.

---

## Lesson 65.5a — When the read replica lies: read-after-write consistency

> *Today's job: take the "add a read replica" line from the capacity math and make it **correct**. A replica makes reads cheap; it also makes them **stale**. Visible win: you toggle a habit, the page re-reads from a lagging replica, the checkmark flips back to undone — then you fix it so it never does.*

The capacity plan ended with a tidy one-liner: *"add a read replica for the API."* That sentence hides the most expensive class of bug in distributed systems, and it's exactly what Rule 21 was written for — it **compiles, the tests pass, and it looks finished.**

Stated plainly, the trap that catches senior engineers too:

> **Proxy code can look finished while the real issue sits in geo consistency.**

You wire the pooler, point read traffic at the replica, and every line reviews clean. Nothing in the *code* is wrong. The bug lives in the *gap in time* between the primary accepting a write and the replica catching up.

### The failure, in Streak's own UI

Walk the habit toggle on the stable stack with a replica behind the pooler:

| t (ms) | Where | What happens |
|---|---|---|
| 0 | primary (`iad1`) | `UPDATE` commits — `done_today = true`. Atomic, correct (Rule 11). |
| 5 | browser | Page re-reads the habit list. |
| 5 | replica | Read is routed here by the pooler. Replica is 40 ms behind: still `done_today = false`. |
| 5 | browser | The checkmark the user *just tapped* **flips back to empty.** |
| 45 | replica | Catches up — too late; the user already saw it regress. |

The user did everything right. The primary did everything right. The replica did everything right. The *architecture* is wrong, because nobody told the read path it must not regress a write the same user just made.

> **replication lag** *(noun)* — the delay between a write committing on the primary and that change being visible on a replica. Usually milliseconds; under load or across regions, seconds. Never zero.
> **read-your-writes** *(consistency guarantee)* — a user always sees the effect of *their own* writes, even if other users see them a beat later. The weakest guarantee that still feels correct.

### Why the pooler hides it

The pooled `DATABASE_URL` from Lesson 62.3 is the accomplice. Abstracting *which* backend serves a connection is the whole point of a pooler — but that same abstraction means your read code has **no idea** whether it just talked to the primary or a replica. Looks finished; isn't. The cure is to make the read/write split *explicit and deliberate*, per request — not a thing the proxy decides for you behind a curtain.

### The three fixes, ranked

1. **Read-your-writes via sticky primary (the default).** After any write, route *that user's* reads to the primary for a short window. Cheap, correct for the common case; you forfeit the replica's benefit only briefly and only for the one user who just wrote.
2. **Bounded staleness (where lag is acceptable).** Reads that don't follow a write take the replica and tolerate, say, ≤ 1 s of lag. Document the bound — it's a product decision, not an accident.
3. **Causal / LSN wait (advanced).** Capture the write's log position (LSN) and make the read *wait* until the replica reaches it. Strongest guarantee, most plumbing; reach for it only when the sticky window isn't precise enough.

For Streak, #1 is the right altitude. Here it is.

### The implementation

Split the single client into a writer and a reader:

```ts
// src/lib/db/client.ts
import { drizzle } from 'drizzle-orm/postgres-js';
import postgres from 'postgres';
import { DATABASE_URL, DATABASE_REPLICA_URL } from '$env/static/private';
import * as schema from './schema';

// Rule 13: timeouts on every external client. A hung DB must fail fast,
// not pin a serverless invocation open until maxDuration.
const opts = { connect_timeout: 10, idle_timeout: 20 } as const;

// Every write goes here.
export const dbPrimary = drizzle(postgres(DATABASE_URL, opts), { schema });

// Reads *may* go here. Falls back to the primary URL when no replica is
// configured (local dev, preview), so behaviour is identical everywhere.
export const dbReplica = drizzle(
  postgres(DATABASE_REPLICA_URL ?? DATABASE_URL, opts),
  { schema },
);

// Back-compat: existing call sites that import `db` keep hitting the primary.
// Reads that opt into the replica use `readDb(event)` instead (below).
export const db = dbPrimary;
```

Then the per-request chooser — the whole consistency story in two functions:

```ts
// src/lib/db/route.ts
import type { RequestEvent } from '@sveltejs/kit';
import { dbPrimary, dbReplica } from './client';

const FRESH_COOKIE = 'rw_fresh';
const STICKY_MS = 5_000; // generous cover for worst-case lag; tune to your p99

// Call after any write: pin *this* user's subsequent reads to the primary
// until the window closes — long enough for the replica to catch up.
export function pinPrimaryAfterWrite(event: RequestEvent): void {
  event.cookies.set(FRESH_COOKIE, String(Date.now() + STICKY_MS), {
    path: '/',
    httpOnly: true,
    sameSite: 'lax',
    maxAge: Math.ceil(STICKY_MS / 1000),
  });
}

// Pick the connection for a read. Primary while the window is open (so the
// writer sees their own write); replica otherwise (so we keep the scale win).
export function readDb(event: RequestEvent) {
  const until = Number(event.cookies.get(FRESH_COOKIE) ?? 0);
  return Date.now() < until ? dbPrimary : dbReplica;
}
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `readDb(event)` | *"Give me the connection this read should use — primary if the user just wrote, replica otherwise."* |
| `pinPrimaryAfterWrite(event)` | *"This user just wrote; route their reads to the primary for the next few seconds."* |
| `Date.now() < until` | *"Are we still inside the freshness window?"* |
| `sameSite: 'lax'` | *"The cookie rides along on top-level navigations — exactly the reads that follow a write."* |

Wire it into the page's load and action:

```ts
// src/routes/(app)/dashboard/+page.server.ts
import { error } from '@sveltejs/kit';
import { and, eq, sql } from 'drizzle-orm';
import { dbPrimary } from '$lib/db/client';
import { pinPrimaryAfterWrite, readDb } from '$lib/db/route';
import { habits } from '$lib/db/schema';
import type { Actions, PageServerLoad } from './$types';

export const load: PageServerLoad = async (event) => {
  const user = event.locals.user;
  if (!user) error(401, 'not authenticated');
  // A read — but it respects the freshness window, so a just-toggled habit
  // is served from the primary, not a lagging replica.
  const list = await readDb(event)
    .select()
    .from(habits)
    .where(eq(habits.userId, user.id));
  return { habits: list };
};

export const actions: Actions = {
  toggle: async (event) => {
    const user = event.locals.user;
    if (!user) error(401, 'not authenticated');
    const data = await event.request.formData();
    const id = String(data.get('id'));
    // Writes always hit the primary (Rule 11: atomic UPDATE, owner-scoped).
    await dbPrimary
      .update(habits)
      .set({ doneToday: sql`NOT ${habits.doneToday}` })
      .where(and(eq(habits.id, id), eq(habits.userId, user.id)));
    pinPrimaryAfterWrite(event); // open the read-your-writes window
    return { success: true };
  },
};
```

### Prove it (Rule 21)

You cannot unit-test your way to confidence here — the bug only exists *in time*. Manufacture the lag and watch:

1. **Make the replica visibly slow.** In dev, point `DATABASE_REPLICA_URL` at the same DB but wrap `dbReplica` reads in an artificial `await new Promise((r) => setTimeout(r, 1500))`, *or* (closer to real) use a Neon/Supabase replica and hammer it so lag climbs.
2. **Disable the fix.** Comment out `pinPrimaryAfterWrite(event)`. Toggle a habit. Watch the checkmark flip back when the load re-reads the stale replica. That flip is the bug, reproduced on demand.
3. **Re-enable the fix.** Toggle again. The read now comes from the primary inside the 5 s window; the checkmark stays. *Visible win.*
4. Remove the artificial delay. Ship.

### Where it ties back

- **Rule 11** gets the write right *on the primary*. This lesson gets the read right *across the replica*. They are two halves of one guarantee; one without the other still ships a visible bug.
- **Appendix A's `withOverride`** is the same idea one layer up — read-your-writes *in the browser*. Server and client both owe the user the sight of their own write; the server reconciles, the client paints instantly.
- **Update the ADR.** When you add the replica (next lesson's format), the consequences section must record the consistency requirement, not just the capacity gain — otherwise the next engineer re-introduces the flip the first time they add a "fast" replica read.

---

## Lesson 65.6 — ADRs (Architecture Decision Records)

When you make a decision that constrains future work, write it down. The format:

```markdown
# ADR-NNN: <Decision>

- **Status:** Accepted | Rejected | Superseded by ADR-XXX
- **Date:** YYYY-MM-DD
- **Deciders:** @billy, @reviewer

## Context
What forced the decision. Constraints, prior art, time pressure, team skill, integrations.

## Decision
What we chose. One paragraph.

## Alternatives considered
Each option, why we rejected it. Two or three.

## Consequences
What this enables. What it constrains. What we'll need to revisit and when.

## References
Links: prior PRs, vendor docs, prior art at other companies, blog posts.
```

ADR-001 for Streak — *"Why Postgres, not Mongo"*:

```markdown
# ADR-001: Postgres as the primary datastore

- **Status:** Accepted
- **Date:** 2026-05-01
- **Deciders:** @billy

## Context
Streak needs persistence for users, habits, sessions, subscriptions, audit_log,
webhook_events. We need transactions (Stripe webhooks must update subscriptions
and webhook_events atomically), foreign keys (cascading user→habits delete),
and rich query power for stats/analytics. Tooling on May 2026 includes Drizzle
(typed Postgres ORM), Neon (serverless Postgres), pgvector (future ML), and
mature observability via `pg_stat_statements`.

## Decision
Use Postgres (Neon-hosted in production, Docker locally) as the primary
datastore. Drizzle as the typed ORM.

## Alternatives considered
1. **MongoDB.** Rejected. Subscription updates + webhook dedupe both require
   single-doc transactions across collections, which Mongo only added recently
   and remains awkward. ACID semantics in Postgres are battle-tested.
2. **Supabase.** Equivalent to Neon for our needs; Neon's branching model
   (per-PR branches) is a sharper fit for our preview-deploy pipeline. We can
   migrate later — Postgres-to-Postgres is straightforward.
3. **SQLite + Litestream.** Tempting for solo-dev simplicity but the moment
   we add a worker thread for cron and the Vercel functions can't share file
   storage, the model breaks.

## Consequences
- **Enables:** atomic mutations (Bible rule #11), real foreign keys, structured
  query plans via EXPLAIN, full-text search via `tsvector` if needed.
- **Constrains:** every serverless function must use the pooled connection
  string; some DDL (e.g. CREATE INDEX CONCURRENTLY) requires the unpooled URL
  and a separate runner.
- **Revisit:** if write throughput exceeds ~5k TPS sustained, consider Citus
  or app-level sharding by user_id.

## References
- Neon docs: https://neon.tech/docs
- Drizzle docs: https://orm.drizzle.team
- "Postgres for Everything" — https://gist.github.com/cpursley/...
```

Senior habit: **decisions live longer than the people who made them.** The ADR is the artifact future-you can read in two years and remember why.

> **ADR (Architecture Decision Record)** *(noun)* — a numbered, dated, blameless write-up of a non-trivial technical decision and its alternatives.

---

## Lesson 65.7 — The boring-tech doctrine

Dan McKinley's essay *"Choose Boring Technology"* (March 2015, still required reading — the original talk + transcript at `mcfunley.com/choose-boring-technology`) named the doctrine: **every team has a finite number of innovation tokens.** Spend them where they buy you the most. Use boring, well-understood tech everywhere else.

Boring choices for Streak:
- **Postgres** over Mongo / DynamoDB / FaunaDB.
- **Stripe** over rolling your own payments.
- **Resend** (or SendGrid / Postmark) over an in-house mail server.
- **Vercel** over self-hosting on EC2.
- **Drizzle** over a hand-rolled query builder.
- **TypeScript-strict** over loose-mode-with-aspirations.

Where to spend an innovation token: maybe Svelte 5 (a year old, lightly battle-tested in production but maturing fast). Maybe Drizzle (only ~3 years old as of 2026). The tokens are *deliberate* spending decisions, not accidents.

> **innovation token** *(idiom, McKinley)* — a metaphorical resource. Each team has 3 to spend across all their tech choices. Spend a token = take on the cost of unfamiliar failure modes. Spend zero on the boring choices; spend strategically on the new ones.

---

## Lesson 65.8 — Read this code

A code-review fragment:

```ts
// src/lib/cache.ts
const cache = new Map<string, unknown>();
export function memoize<T>(key: string, fn: () => T): T {
  if (!cache.has(key)) cache.set(key, fn());
  return cache.get(key) as T;
}
```

Three issues a senior reviewer flags. Find them.

<details>
<summary>Answer</summary>

1. **`cache as T` cast.** Bypasses type safety. The `Map` is `unknown`-valued; we trust it stores the right shape because of the `key`. But there's no enforcement. Fix with a generic-parameterised cache class or typed-key.

2. **Module-scoped Map = SSR singleton landmine.** On a server, this map is shared across *all users*. If `key` includes user data, it leaks. **Bible rule echo, Ch 29.** Move to per-request via `event.locals` or context.

3. **No eviction, no size cap.** Memory grows unboundedly. After a long-running deploy, this map could OOM the function. Use an LRU with a size cap (e.g. `lru-cache` package), or expire entries by TTL.

A senior wouldn't merge this without all three addressed.
</details>

---

## Lesson 65.9 — Now you write it

**The English sentence first:**

> *"Write ADR-002 explaining 'Why we use Resend, not Postmark or SendGrid.' Cover context (we need transactional email + occasional batch), the decision, two alternatives rejected, and consequences."*

<details>
<summary>Worked answer (one valid version)</summary>

```markdown
# ADR-002: Resend for transactional email

- **Status:** Accepted
- **Date:** 2026-05-03
- **Deciders:** @billy

## Context
Streak needs transactional email (verification, password reset, weekly summary,
daily reminders). At 100k users this is ~100k emails/day with bursts. We need:
typed SDK (TS-strict alignment), DKIM/SPF/DMARC verification flow that's
straightforward, audit trail of sends, deliverability comparable to incumbents.

## Decision
Use Resend as the email provider.

## Alternatives considered
1. **Postmark.** Excellent deliverability and mature; their TS SDK is older and
   less ergonomic; pricing higher per-email at our volume. Rejected mainly on
   developer experience parity with the rest of the stack.
2. **SendGrid (Twilio).** Industry standard; SDK is clunky; Twilio's billing
   surprises (deliverability problems → enforced segmented sending) caused
   real outages at past employers. Rejected on operational concerns.
3. **Self-host Postfix.** Two innovation tokens for an undifferentiated
   capability. Rejected immediately.

## Consequences
- **Enables:** clean TS types, DKIM/SPF/DMARC instructions in the dashboard,
  webhooks for bounces and complaints we can wire into our audit trail.
- **Constrains:** vendor lock-in for the send API surface (mitigated: our
  `sendOnce` wrapper isolates Resend behind a 4-line interface; swapping
  takes a day).
- **Revisit:** if deliverability drops below 98% to inbox, evaluate Postmark.
```
</details>

---

## Lesson 65.10 — Recurring concepts from earlier chapters

- **Every Bible rule** — the review is the moment you cite them.
- **`audit_log`** (Ch 47) — partitioning is the scaling endgame.
- **`Result`, `unknown`, `noUncheckedIndexedAccess`** — the type-system reasons your reviews are short.
- **Boring-tech doctrine** — the meta-rule that explains *why* the Bible exists at all.

---

## Lesson 65.11 — What you can now read in the wild

After Chapter 65 you can:

- Walk a 600-line PR through eight review lenses and produce a structured comment list.
- Read an ADR and tell good from cargo-cult.
- Do back-of-envelope capacity math on any system size.
- Read a Mermaid architecture diagram and spot what's missing.
- Apply the **innovation-token** lens to *any* technology decision.

---

## Glossary added in Chapter 65

| Term | Definition |
|---|---|
| structured review | Eight-lens code review with prefix-tagged comments. |
| BLOCKING / nit / consider / praise | Comment-prefix conventions. |
| Mermaid | Text-based diagram format embedded in Markdown. |
| capacity planning | Back-of-envelope sizing for projected scale. |
| ADR | Architecture Decision Record. |
| boring-tech doctrine | McKinley: spend innovation tokens deliberately. |
| innovation token | The cost of unfamiliarity in a tech choice. |

---

## End-of-chapter checkpoint

- [ ] You wrote the full review of the planted-PR fixture (12 BLOCKING + 4 nits).
- [ ] `docs/adr/0001-postgres-over-mongo.md` exists.
- [ ] `docs/adr/0002-resend.md` exists (or your equivalent).
- [ ] `docs/architecture.md` has a Mermaid diagram of Streak's full topology.
- [ ] `docs/scaling.md` has the back-of-envelope plan for 100k users.

---

# Chapter 66 — Graduation Part I: the brief

> *Today's job:* build a real new feature in Streak from a one-paragraph brief. Alone. *Visible win:* the feature ships to production behind a feature flag; the runbook is written; the test suite covers it; the audit log records it; the metrics are in place.

This is the chapter where you prove the book worked. Every primitive in the brief below is something you've seen earlier — atomic UPDATEs (Ch 42), `withAudit` (Ch 47), `<ConfirmDialog>` (Ch 53), Prometheus with bounded labels (Ch 63), Vercel cron (Ch 64), feature flags (Ch 57's env-var matrix). Nothing is *new*. The skill the brief tests is **composition**.

---

## Lesson 66.1 — How a senior approaches a brief

Before you write any code, do these in order. Senior engineers do them in their head; you do them on paper.

1. **Read the brief three times.** First for shape, second for constraints, third for traps.
2. **List the risks** (the parts most likely to break or take longer than expected). Order by *blast radius if it goes wrong*.
3. **Sketch the architecture diagram** in Mermaid. Show every new component, every new external call, every new table. If you can't draw it in 20 lines of Mermaid, the design is too complex; cut.
4. **Write the ADR for the riskiest decision** *first*. The schema choice for Streak Freezes is risky (single column on `users` vs separate consumption-log table); decide and justify before coding.
5. **Plan the slices** — how you'd ship this in 3 PRs instead of 1. Smaller PRs are reviewable; one giant PR is unreviewable.
6. **Write the test list** — the unit, integration, e2e, and contract tests you'll write *before* implementation. Test-first thinking, even if you write code-first sometimes.
7. **Identify the rollback** — *"if this is broken in production at 3am, what do I undo?"*. Feature flag is your friend.

If any of these is hand-wavy, the implementation will be hand-wavy too. The senior pattern: **the work happens before the keyboard**.

---

## Lesson 66.2 — The brief

> **Streak Freezes — Pro feature.**
>
> Pro users get one streak freeze per calendar month: a one-day skip that doesn't break their streak. The home screen shows the count of remaining freezes for the current month. Consuming a freeze is a deliberate user action — confirm dialog and audit row. The reset to 1 happens at the start of each calendar month, in **the user's local timezone** (not server UTC). Safe under concurrent consumption (a user clicking twice cannot consume two freezes). Roll out behind a feature flag. Fully observable.

Estimated effort: 6–10 hours of focused work.

> **The timezone trap, named.** Read the brief twice — the word "calendar month" is doing real work. A user in Tokyo sees the calendar flip to February at 00:00 JST, which is **15:00 UTC the previous day**. If your reset cron fires at 00:00 UTC, the Tokyo user gets a "new month" freeze 9 hours after their February-1 morning has already started. This is a UX bug *and* an unfairness bug (Pacific users effectively get a longer month). A senior brief surfaces this; a senior implementation handles it.
>
> Two valid approaches:
>
> 1. **Store `users.timezone`** (e.g. `"Asia/Tokyo"`, IANA name) — a Ch 39-style migration adds the column. The reset cron runs hourly and resets per-user when their local clock crosses the month boundary; "remaining freezes" reads pass through this stored value for any date math.
> 2. **Compute on read** from stored UTC `freeze_consumed_at` timestamps — no reset cron needed. The "remaining freezes" query bucketises consumption timestamps into the user's local month at read time. Cleaner; needs the timezone column too, but only as a presentation concern.
>
> Both are buildable from what's already in the book (migrations, atomic UPDATEs, audit log). The ADR (deliverable #2) is where you defend the choice — concision counts; the senior judgment shows in *which* one you pick and *why*, not in the implementation.

---

## Lesson 66.3 — Required deliverables

These are level-7 deliverables. Every one. No shortcuts.

1. **Migration** for the schema change (column on `users`, or separate `freeze_consumption` table — your call).
2. **ADR** explaining the schema decision (`docs/adr/0003-freeze-storage.md`).
3. **`consumeFreezeForUser(userId)`** as a pure-async function returning `Result`, callable from both the form action and the integration test.
4. **`consumeFreeze` form action** wrapping it.
5. **Atomic conditional UPDATE** in `consumeFreezeForUser` — Bible rule #11.
6. **`withAudit` wrapper** for the action — Bible rule (Ch 47).
7. **Pro-status check** at both the action *and* the UI (defence in depth — Ch 6, 47).
8. **`<ConfirmDialog>`** for the user-facing confirm step (Ch 53 primitive).
9. **`/api/cron/reset-freezes/+server.ts`** — auth-gated, idempotent monthly reset, with `withAudit`.
10. **Unit tests** for any pure helper.
11. **Required**: at least one property-based test on a pure helper. Find an invariant — `splitCents(t, n).reduce((a, b) => a + b) === t`, the freeze-counter never goes negative across N random consume-and-reset interleavings, the on-read computation returns the same value for the same input regardless of when called. The shrinker (`fast-check`) will find counterexamples your example-based tests would miss.
12. **Integration test:** 10 simultaneous `consumeFreezeForUser` calls; assert ≤ 1 succeeds.
13. **Playwright e2e tests:** the happy path (consume freeze; banner shows "0 left"); the empty path (no freezes left; button disabled); the non-Pro path (free user doesn't see the feature).
14. **Feature flag** `PUBLIC_ENABLE_STREAK_FREEZES` so you can roll out / roll back instantly.
15. **Prometheus metric** `streak_freezes_consumed_total{plan}` with `plan` label bounded to `{free, pro}` via `safePlanLabel`.
16. **Structured log line** per consume + cron run, with request ID and user ID.
17. **Runbook** `docs/runbooks/streak-freezes.md` explaining what to do if (a) consume rates spike, (b) the monthly cron fails to fire, (c) the database column drifts.
18. **PR description** citing every Bible rule the change depends on.
19. **EXPLAIN ANALYZE** on the most-used query in the freeze flow (e.g. the user-status check that reads `freezes_remaining_month` or the on-read consumption count), confirming the index is hit and the plan is bounded. The concurrent-insert integration test (deliverable #12) is the runtime evidence for the atomic UPDATE; the metric screenshot (deliverable #15) is the runtime evidence for the bounded label set; the Stripe-CLI replay is the runtime evidence for the Pro gate. Bible #21: compilation and tests are necessary, not sufficient — you write down the EXPLAIN plan for review, not just the test result.

---

## Lesson 66.4 — Rollout sequence

The senior way to ship this:

1. **PR 1 (~2 hours):** migration + schema decision + ADR. *Empty implementation* — column added, default value backfilled, no UI, no actions. Merge, deploy. Verify the migration applied cleanly.
2. **PR 2 (~3 hours):** `consumeFreezeForUser` pure function + integration tests + atomic-concurrent test. *No UI yet*. Feature flag scaffold (`PUBLIC_ENABLE_STREAK_FREEZES=false` everywhere). Merge, deploy.
3. **PR 3 (~3 hours):** form action + UI + e2e tests + metric + runbook + cron. Flag still off in production. Merge, deploy.
4. **Manual flag flip** in Vercel: turn `PUBLIC_ENABLE_STREAK_FREEZES=true` for *your own user* via a per-user override (or a beta-list table). Smoke-test in production with no other users at risk.
5. **Roll out to all Pro users** by flipping the flag to true. Watch the metric and the log volume for 30 minutes. Roll back (flip flag off) at the first sign of trouble.

This is the **gradual rollout** pattern. Senior habit: never go from `0% → 100%` directly.

---

## Lesson 66.5 — When you get stuck

You will. Three escape hatches:

1. **Reread the relevant chapter.** If atomic UPDATE feels uncertain, reread Ch 42 — *not* the brief.
2. **Sketch the wire flow on paper.** What requests get sent, what responses come back, what writes hit the DB.
3. **Write the test first.** The act of writing the test name (`it('rejects a second consume in the same month')`) clarifies what the implementation must do.

Don't ask the internet for someone else's `consumeFreeze` code. The point of this chapter is to graduate *your judgment*; copying defeats it.

---

## Lesson 66.6 — Now you do it

Open a feature branch. Set a 6-hour timer. Build it.

When you commit and merge, return for Chapter 67.

---

## Glossary added in Chapter 66

| Term | Definition |
|---|---|
| brief | A one-paragraph feature description from product to engineering. |
| feature flag | An env var or DB toggle that gates a feature on/off without redeploying. |
| gradual rollout | Shipping in stages (1% → 10% → 100%) with a rollback at each step. |
| risk-ordered planning | Working through the riskiest unknowns first. |

---

## End-of-chapter checkpoint

- [ ] You read the brief three times before writing code.
- [ ] You wrote ADR-003 first.
- [ ] You shipped in 3 PRs, not 1.
- [ ] All 19 deliverables exist.
- [ ] The feature flipped on for your own user before going to all Pros.
- [ ] You can quote which Bible rules each deliverable enforces.

---

# Chapter 67 — Graduation Part II: the post-mortem

> *Today's job:* write a structured, blameless post-mortem of the feature you just built. *Visible win:* `docs/post-mortems/streak-freezes.md` lives in the repo — an artifact you could hand to a hiring manager as evidence of principal-engineer thinking.

The post-mortem is the *learning artifact*. Code teaches you one lesson at a time; the act of *writing about your own work* teaches you ten at a time. Senior engineers do this for every non-trivial feature, even when nothing went wrong.

---

## Lesson 67.1 — Why blameless

Two reasons:

1. **Blame stops learning.** When the doc reads *"I made a mistake here"*, the next time you face the same shape of problem, your brain replays the *shame*, not the *lesson*. Replace *"I"* with *"the system"*: *"The implementation didn't account for X."* Same fact, different teaching.
2. **Other readers.** A post-mortem is for future-you and for any engineer who joins the project. Naming yourself as "the bad one" doesn't help them solve the next thing.

The blameless post-mortem describes *decisions made under constraints* and *what those constraints made invisible*. Not heroes and villains. Conditions and outcomes.

> **blameless post-mortem** *(noun)* — a write-up of a decision or incident that focuses on system properties, not individual culpability. Pioneered at Etsy / Google SRE.

---

## Lesson 67.2 — The format

```markdown
# <Feature> — Post-mortem (YYYY-MM-DD)

## Context
Why we shipped this. What the user need was. What constraints shaped the design.

## What I built
3–5 line summary. Schema choice, action shape, UI shape, observability surface.

## Decisions and alternatives
For each significant decision (≥3 of them):
- What I chose.
- What I rejected.
- Why, in one sentence.

## What surprised me
Both helpful surprises (something was easier than expected) and harmful (something
was harder, or broke in a way I didn't predict). 3–8 bullets.

## What I'd do differently
Honest reflection. Specific. *"I'd write the integration test before the action."*

## What's still scary
The tail risks I haven't covered. Things I'd watch in production for 2 weeks.

## Bible rules cited
- ✅ Rules followed (with line refs).
- ⚠️ Rules bent deliberately, with rationale.

## What I'd plant for the next learner
If I were teaching this feature, where would I put a deliberate bug for them
to find? (This is a fluency exercise: it forces you to identify the most
educational *failure modes* of your own work.)
```

---

## Lesson 67.3 — A worked example

Here's a partial post-mortem for Streak Freezes, written as if you finished it. Use this shape; substitute your own decisions and surprises.

```markdown
# Streak Freezes — Post-mortem (2026-05-12)

## Context
Pro users requested a "skip a day without breaking the streak" feature.
Constraints: Pro-only, monthly reset, single-machine concurrent-safe,
observable, rollback-able. The existing audit-log infra and atomic-UPDATE
pattern from Ch 42 / 47 were prerequisites.

## What I built
- New column `users.freezes_remaining_month INTEGER NOT NULL DEFAULT 1`.
- Pure async `consumeFreezeForUser(userId): Promise<Result<void, 'no-freeze' | 'not-pro'>>` using atomic conditional UPDATE.
- Form action `consumeFreeze` wrapping it via `withAudit`.
- `<ConfirmDialog>` integration in dashboard.
- Monthly cron at `0 0 1 * *` resetting via `UPDATE users SET freezes_remaining_month = 1`.
- Counter `streak_freezes_consumed_total{plan}` with bounded label.
- Feature flag `PUBLIC_ENABLE_STREAK_FREEZES`.

## Decisions and alternatives
1. **Schema: column on `users` vs. `freeze_consumption` table.**
   Chose: column. Rejected: table.
   Why: simpler queries, no JOIN for the common "how many left this month"
   read. The table would have been right if we wanted to *retain history*
   of consumption, which we don't (yet). Documented as ADR-003.

2. **Reset mechanism: cron job vs. on-read computation.**
   Chose: cron. Rejected: compute from `consumed_at` on every read.
   Why: simpler `consumeFreezeForUser` (no awareness of period boundaries);
   slight risk if cron fails (mitigation: alert on missed cron + the field
   is bounded so worst case is "user can't consume" not "user double-consumes").

3. **Concurrency: atomic UPDATE vs SELECT-then-UPDATE.**
   Chose: atomic. Rejected: SELECT-then-UPDATE (Bible rule #11; not even close).

## What surprised me
- The `<ConfirmDialog>` integration was 4 lines — simpler than expected.
- Property-based testing for the cron's idempotency (running it twice in a
  month should be a no-op for already-reset users) was harder to express
  than I thought. Wrote it conventional-test instead and added a TODO.
- The Pro-status check needed to live in *three* places — UI button gate,
  action gate, e2e seed data. Forgetting the e2e seed initially gave me
  a flaky test that I almost wrote off as a Playwright bug.

## What I'd do differently
- Write the integration test for the atomic UPDATE *first*, before the
  action wrapper. The shape of the test would have clarified the function
  signature.
- Ship PR 1 (migration only) earlier in the day. Migrations on Vercel
  serverless need the unpooled URL and a few minutes to settle.
- Add a non-Pro e2e test from the start, not as an afterthought.

## What's still scary
- The cron is auth-gated by a shared secret. If that secret leaks, anyone
  can hit the endpoint and trigger a global reset. Mitigation in place
  (rate-limit per IP), but rotating the secret remains a manual step.
- The migration column has `DEFAULT 1`, so existing users get one freeze
  retroactively for the current month. Whether that's correct depends on
  product intent; I assumed yes.
- I haven't tested the rollback (flag off after on); should rehearse this
  before next month's reset.
- **Timezone gap.** The monthly cron at `0 0 1 * *` resets via UTC. Users
  west of UTC see their freezes reset *before* their local month starts;
  users east see it *after*. The brief said "calendar month in user's local
  timezone"; the cron does not honour that. The fix is either (a) split the
  cron into per-timezone-bucket runs (one per IANA zone we have users in),
  or (b) compute reset boundaries on read using `users.timezone`. We did
  neither in v1; v2 needs it. Filed as ADR-003-followup.

## Bible rules cited
- ✅ #11 (atomic UPDATE) — `consumeFreezeForUser` line 12.
- ✅ #12 (idempotency on external) — cron handler dedupe via reset key.
- ✅ #14 (bounded labels) — `safePlanLabel`.
- ✅ #15 (no PII logged) — log lines have user IDs only, no emails.
- ✅ #18 (forward-only migrations) — column added, never dropped.
- ✅ #20 (visible win) — *re-read on review:* PR 1 (migration only) does
  have a visible win — the deploy succeeds and `EXPLAIN` shows the new
  column. The visible win for a migration PR is the schema change; the
  user-facing feature lands in PR 2. The earlier "bent" framing was wrong:
  Bible #20 says *"every chapter delivers a visible win"* — that applies to
  *chapters*, not *PRs*. PRs are scoped by deployability, which migration
  PRs satisfy. Lesson learned: cite the rule's exact wording, not the
  paraphrase that lives in your head.

## What I'd plant for the next learner
Three deliberate bugs:
1. **Replace atomic UPDATE with SELECT-then-UPDATE.** Reader writes the
   integration test in Ch 42 style, watches concurrent calls succeed both,
   and learns Bible #11 viscerally.
2. **Use `email` as the metric label.** Reader watches Prometheus memory
   grow during e2e seeding and learns Bible #14 viscerally.
3. **Forget the Pro-status check in the action body** (only in the UI).
   Reader uses dev tools to inspect-then-submit and bypasses the gate;
   learns defence-in-depth from Ch 6 viscerally.
```

This is the artifact. Write yours after you ship.

---

## Lesson 67.4 — Reread the Bible

Open the front of the book — the 21 non-negotiables. Read each one *with the freeze feature fresh in mind*. For each, ask:

- **Did this rule earn its keep in the freeze feature?** Mark a tick.
- **Did I bend this rule? Was the bend justified?** Mark a question mark.
- **Was there a rule that *should* exist but doesn't?** Note it.

The third question is the most valuable. The Bible is the May 5 2026 version; the next edition incorporates what you (and other readers) found missing.

---

## Lesson 67.5 — Inventory what's still scary

After every major project, write a short list — three to ten items — of *"things I shipped but don't fully understand."* For most engineers most of the time, this list includes things like:

- Postgres MVCC behaviour under heavy concurrent writes.
- TLS certificate renewal at 3am if Vercel's automation fails.
- Worker-thread crash recovery if Argon2 segfaults.
- Stripe webhook signing-secret rotation choreography.
- The exact failure mode of an exhausted DB connection pool.

This is **your next learning agenda.** Knock items off it deliberately. Senior engineers do this for life — the list never reaches zero, because the field keeps moving.

---

## Lesson 67.6 — The closing rule

You came in not knowing what a terminal was. You leave with a deployed, paid, observable, audited product, an integration test suite that proves Bible rule #11 at runtime, an ADR portfolio, a security-review document, an incident-response runbook, a post-mortem of a feature you shipped alone.

You can do this work anywhere on Earth where someone needs Svelte 5 + SvelteKit + TypeScript + Postgres + Stripe in production. You're not finishing a course — you're finishing a *training cycle*. The cycle continues for life.

---

**You are now a Svelte 5 + SvelteKit + TypeScript + Postgres + Stripe principal engineer as of May 5, 2026.**

Welcome to the trade.

---

## Glossary added in Chapter 67

| Term | Definition |
|---|---|
| post-mortem | Structured retrospective of a feature, decision, or incident. |
| blameless | Framed around system properties, not individual culpability. |
| learning agenda | The running list of "what I shipped but don't fully understand." |
| training cycle | The continuous practice loop a senior engineer never finishes. |

---

## End-of-chapter checkpoint

- [ ] `docs/post-mortems/streak-freezes.md` exists.
- [ ] You reread all 21 Bible rules with the freeze feature in mind.
- [ ] Your "still scary" list has at least three items.
- [ ] You closed the book.

---

# Appendix A — Remote Functions, the experimental future

> *Optional chapter. Still experimental as of June 4, 2026 — gated behind `kit.experimental.remoteFunctions`, and the shape has already moved several times (it changed again across the June 2026 releases, up to SvelteKit 2.61.0). Don't bet a production app on it yet, but know it exists and recognise the **current** shape, not last month's.*

The book's spine — `+page.server.ts` + `load` + `actions` + `+server.ts` — is the stable, idiomatic story every senior engineer in May 2026 already knows. **Remote Functions** are where SvelteKit is heading: a unified, type-safe primitive for client→server calls that collapses load + action + REST into one shape.

This appendix gives you enough to *recognise* remote functions in the wild and rewrite *one* feature using them, so you understand the trade-off if you adopt them in your own app a year from now.

> **What changed between the May 2026 cut of this book and June 4, 2026.** Two *breaking* changes landed in the June releases and instantly date every older tutorial:
> - **`.run()` is gone (2.61.0).** You no longer call `query(...).run()`. You `await` a query *directly* — in markup, and equally in event handlers, async callbacks, and module scope. Every `.run()` you see in an older snippet is now a deletion.
> - **`enhance` callbacks receive the form instance (2.61.0).** The callback argument used to be a `{ form, data, submit }` object; it is now a copy of the form remote-function instance itself (everything except `.enhance`), and that instance exposes a programmatic `submit()` that resolves to a boolean.
>
> Smaller June additions worth knowing: `form.submit()` returns a validity boolean; remote `submit`/`hidden` fields accept booleans and numbers directly; `query.live` is async-iterable; `query.batch` is stable for the N+1 case; and dev mode now *warns* when a form's validation issues go unread (an incomplete-UX smell). The throughline: this is still the fastest-moving corner of SvelteKit — pin a version and re-verify on every minor bump.

---

## A.1 — Enabling remote functions

In `svelte.config.ts`:

```ts
const config: Config = {
  kit: {
    experimental: {
      remoteFunctions: true,
    },
  },
  compilerOptions: {
    experimental: {
      async: true,
    },
  },
};
```

The flag pair is required because remote functions use top-level `await` in markup, which depends on the experimental async-Svelte compiler mode.

---

## A.2 — The primitives (June 2026)

> **Pinned to SvelteKit 2.61.0 with `kit.experimental.remoteFunctions: true`** (June 4, 2026). Verify each signature against the live docs before adoption — this is the single most volatile corner of the framework.

`$app/server` exports six building blocks:

- **`query(schema?, fn)`** — read-only server function. You **`await` it directly** wherever you need the data: `{#each await getHabits() as h}` in markup, or `const list = await getHabits()` in a handler (the old `.run()` is gone). Cached per argument and deduplicated within a render. Methods on the returned object: `.refresh()` (re-fetch from the server), `.set(data)` (overwrite the client cache with a value you already have — e.g. from a mutation's return), and `.withOverride(fn)` (an **optimistic** local override that auto-rolls-back if the mutation fails).
- **`query.batch(schema, fn)`** — the N+1 killer. The server receives the *array* of args collected within a macrotask and returns a lookup function `(arg) => result`. Each call site still `await`s a single value; the round-trips collapse to one.
- **`query.live(fn)`** — pass an **async generator**; every `yield` pushes a new value to all subscribers. `await` it like a query, plus `.connected` (is the stream live?) and `.reconnect()`. Auto-reconnects on drop. This is the real-time primitive.
- **`form(schema, fn)`** — replaces a form action. Spread it onto `<form {...addHabit}>`; build inputs from typed accessors `addHabit.fields.name.as('text')` (and `.as('submit', 'value')` for multi-button forms). Read/write values with `.value()` / `.set({...})`, validate with `.validate()`, and customise submission with `.enhance(async (form) => { if (await form.submit()) … })` — note the callback now receives the **form instance**.
- **`command(schema, fn)`** — imperative mutation called straight from an event handler (`await toggleHabit(id)`). No automatic revalidation; you drive the re-read with the **single-flight** `.updates(...)` pattern so the mutation and the refresh share one round-trip.
- **`prerender(schema?, fn, { inputs })`** — like `query`, but resolved at *build* time for the listed `inputs`. Static data, zero per-request cost; the right tool for content that doesn't change between deploys.

> **single-flight update** *(noun)* — telling a mutation, in the same request, which queries to re-read, so the browser doesn't fire the write and *then* a second request to refresh. `await toggleHabit(id).updates(getHabits())` mutates and re-reads in one flight. Pair it with `.withOverride(...)` for an instant optimistic paint that the server reconciles or rolls back.

---

## A.3 — A worked rewrite

Take Streak's home-page habit list (currently `+page.server.ts` `load` + form action) and rewrite via remote functions:

```ts
// src/routes/(app)/dashboard/habits.remote.ts
import { query, form, command, getRequestEvent } from '$app/server';
import { error } from '@sveltejs/kit';
import * as v from 'valibot';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
import { and, eq, sql } from 'drizzle-orm';
import { addHabitForUser } from '$lib/habits-server';

// One place to read auth. `error()` short-circuits with a real HTTP status
// instead of a thrown string the client can't classify (Rule 17 — no silent
// failures; the caller gets a 401, not a generic 500).
function requireUser() {
  const { locals } = getRequestEvent();
  if (!locals.user) error(401, 'not authenticated');
  return locals.user;
}

export const getHabits = query(async () => {
  const user = requireUser();
  return db.select().from(habits).where(eq(habits.userId, user.id));
});

export const addHabit = form(
  v.object({ name: v.pipe(v.string(), v.trim(), v.minLength(1), v.maxLength(100)) }),
  async ({ name }) => {
    const user = requireUser();
    await addHabitForUser(user.id, name);
    // Re-read in the *same* flight. Awaited, not fire-and-forget: the form
    // resolves only once the fresh list is back, so the UI never paints stale.
    await getHabits().refresh();
  },
);

export const toggleHabit = command(v.string(), async (id) => {
  const user = requireUser();
  // Atomic UPDATE … (Rule 11): flip the flag in the database, never
  // SELECT-then-write. The `and(...)` scopes it to the owner — a user can't
  // toggle someone else's habit even if they forge the id.
  await db
    .update(habits)
    .set({ doneToday: sql`NOT ${habits.doneToday}` })
    .where(and(eq(habits.id, id), eq(habits.userId, user.id)));
});
```

In the page:

```svelte
<script lang="ts">
  import { getHabits, addHabit, toggleHabit } from './habits.remote';
</script>

<h1>Today</h1>

<form
  {...addHabit.enhance(async (form) => {
    if (await form.submit()) form.element.reset();
  })}
>
  <input {...addHabit.fields.name.as('text')} placeholder="Add a habit…" />
  <button type="submit">Add</button>
</form>

<ul>
  {#each await getHabits() as habit (habit.id)}
    <li>
      <button
        type="button"
        onclick={() =>
          toggleHabit(habit.id).updates(
            getHabits().withOverride((list) =>
              list.map((h) =>
                h.id === habit.id ? { ...h, doneToday: !h.doneToday } : h,
              ),
            ),
          )}
      >
        {habit.doneToday ? '✓' : '○'} {habit.name}
      </button>
    </li>
  {/each}
</ul>
```

The toggle is the part worth studying. `toggleHabit(habit.id)` is the command; `.updates(getHabits()...)` tells SvelteKit to re-read the list in the same round-trip; `.withOverride(...)` paints the flip *immediately* on the client and auto-rolls-back if the command throws. That `withOverride` is **read-your-writes on the client** — the in-browser twin of the read-after-write problem you solve on the server in Lesson 65.5a. Same shape, two layers: show the user their own write instantly, reconcile with the truth a beat later.

Compared to the stable story, you save:
- The separate `+page.server.ts`.
- The `actions` boilerplate.
- The `use:enhance` import.
- The `data` prop pattern.

You gain:
- A clearer *unit of work* per server function.
- Built-in deduplication of repeated `getHabits()` calls.
- Type-safe `.refresh()` / `.set()` / `.withOverride()` cache primitives — optimistic UI for free.
- Single-flight mutations (`.updates(...)`) that re-read in the same round-trip.

You lose:
- The progressive-enhancement *guarantee* (forms still work, but the implementation is younger).
- A year of stability — and this is not hypothetical: `.run()` was removed and the `enhance` callback shape changed in the June 2026 releases alone. Adopting means budgeting for a refactor on minor bumps.

---

## A.4 — When to adopt

**Adopt now** for: side projects; greenfield apps you control end-to-end; teams comfortable with experimental APIs and willing to refactor on minor releases.

**Wait** for: production apps with paying users; teams without dedicated time to track API changes; codebases where the boring story is well-understood and shipping is the priority.

The senior judgment call: spend an *innovation token* (Ch 65) on remote functions only if it solves a problem the stable story can't. For most teams in 2026, the stable story is enough.

---

## A.5 — Further reading

- Official SvelteKit Remote Functions docs (live, evolving): `svelte.dev/docs/kit/remote-functions`.
- The monthly **"What's new in Svelte"** posts (`svelte.dev/blog`) — the June 2026 edition is where the `.run()` removal and the `enhance`-callback change were announced. Read the one for the month you adopt.
- Rich Harris's RFC threads on GitHub.
- The migration guide once it stabilises.

---

# Appendix B — The reading list

> *Curated. The minimum set to keep going beyond the book. One sentence per entry on why it earns the spot.*

## Svelte 5 / SvelteKit

- **[svelte.dev/docs](https://svelte.dev/docs)** — primary source for runes, snippets, transitions, motion. Reread when the spec changes.
- **[kit.svelte.dev/docs](https://kit.svelte.dev/docs)** — primary source for routing, load, form actions, hooks, adapters.
- **Rich Harris on YouTube** — the framework's lead designer; his talks on reactivity and the SvelteKit philosophy are the best mental model you can install.

## TypeScript

- **[Matt Pocock — Total TypeScript](https://www.totaltypescript.com)** — beyond-the-basics typing patterns. Especially the `Type-Level Programming` chapter.
- **TypeScript handbook** — `https://typescriptlang.org/docs/handbook/2/everyday-types.html`. Surprisingly readable cover-to-cover.

## Postgres and SQL

- **Markus Winand — *Use The Index, Luke!*** — free online. Read once; reread the EXPLAIN chapters yearly. The single best resource on database performance.
- **[Crunchy Data blog](https://www.crunchydata.com/blog)** — high signal Postgres internals.
- **The Postgres docs themselves** — short, precise, and cite the source.

## Web performance and accessibility

- **Addy Osmani — *Image Optimization*** (free PDF). The CLS chapter is what the book's Bible rule #16 distills.
- **[web.dev](https://web.dev)** — Google's curated guides on Core Web Vitals.
- **[deque.com/axe](https://www.deque.com/axe/)** — the test framework your test suite uses; their docs explain *why* each WCAG rule matters.

## Security

- **OWASP Cheat Sheets** — `https://cheatsheetseries.owasp.org`. Authentication, Session Management, Input Validation, Logging, REST Security. The threat model in Chapter 44 is the OWASP Auth cheatsheet condensed.
- **[OWASP Top 10](https://owasp.org/www-project-top-ten/)** — the Bible's Bible for web security.

## SRE and operations

- **Google — *The Site Reliability Workbook*** (free online). Chapters 5 (Alerting) and 17 (Post-mortems) most relevant.
- **Will Larson — *An Elegant Puzzle*** — engineering management seen by a principal-shaped lens.

## Stripe / payments

- **[Stripe Docs — Idempotency](https://stripe.com/docs/api/idempotent_requests)** and **Webhook Signing** are required reading before going live.
- **Stripe's blog** — surprisingly good engineering writing on payments edge cases.

## Tooling

- **Conventional Commits** — `https://www.conventionalcommits.org`. Five minutes; pays back forever.
- **Vercel docs** — Adapters, ISR, Edge Functions, Cron.
- **Cloudflare R2 docs** — presigned URLs, multipart uploads, lifecycle policies.
- **[Drizzle docs](https://orm.drizzle.team)** — your ORM; the migration story is especially good.
- **[Valibot docs](https://valibot.dev)** — the boundary parser; lightweight, tree-shakeable.

## Books worth owning

- **John Ousterhout — *A Philosophy of Software Design*** — short, dense, the book on naming and complexity.
- **Brian Kernighan & Rob Pike — *The Practice of Programming*** — the canonical short book on the *craft* of software (debugging, style, performance, portability). Twenty years old; still right about almost everything that matters in 2026.
- **Donald Norman — *The Design of Everyday Things*** — UX you'll feel in every form you ever wire up.

## What not to read

- Books promising "10 patterns to architect anything." There aren't ten patterns; there are taste and trade-offs.
- LinkedIn-influencer takes on engineering. Almost all signal-free.
- Anything claiming the One True Way. The boring-tech doctrine (Ch 65) is closer to the truth.

---

> The list is finite. Read the boring stuff first; it ages slower.

---

# Appendix C — The Bible card

> *Print this on a single sheet. Stick it above your monitor. Reread it whenever a code-review feels harder than it should.*

```
THE STREAK BIBLE — May 5, 2026

TOOLING
 1. pnpm only.
 2. .svelte and .ts only — never .js.
 3. TypeScript strict from line one. No any. No !. No @ts-ignore.
 4. Svelte 5 runes only — no export let, no $:, no on:click, no <slot>.

IDIOMS
 5a. += not = x + 1.
 5b. === not ==.
 5c. const by default; let only when reassignment is real.
 5d. ?? not || for nullish defaults.
 5e. ?. over manual null checks.
 5f. Early returns over nested if.
 5g. type="button" on every <button> not inside a submit-form.
 6.  Engineer-English read-aloud on every snippet.
 7.  Every example is load-bearing.
 8.  English sentence first, code second.
 9.  No comment lies.

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
21. Compilation and tests are necessary, not sufficient.
    Demand runtime evidence.

----
Rule 5 has 7 sub-rules (5a–5g, the senior idioms).
Total: 21 numbered + 6 sub-rules = 27 named rules.
The card lists 21 for printability; sub-rules are inline in the book.
```

---

## How to use the card

- **In code review.** When you reject a PR, cite the rule by number. *"BLOCKING: violates rule #11 (line 42 is SELECT-then-UPDATE)."* The number is more authoritative than your opinion.
- **In an interview.** When asked *"how would you ship X safely?"*, the rules are the answer's spine.
- **In your own debugging.** When something's broken and you can't see why, walk the rules and ask *"did I follow each one?"*. About 60% of the time, the bug is in a violation.
- **In your runbook.** When the production incident is *"the metrics dashboard is empty"*, rule #14 is your first hypothesis.

The card is not law. It's *prior art on a single page*. The next edition will have new rules; some current ones will retire. But these 21, on May 5 2026, are the floor for the stack.

---

**End of book.**

The reader has built Streak from `pnpm create` to a deployed, audited, observable, paid product, and shipped a feature on their own. They are a principal-engineer-level-7 specialist in this stack as of May 5, 2026.

Welcome to the trade.
