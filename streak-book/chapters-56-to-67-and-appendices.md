# Part IX — Shipping it, operating it, the principal-engineer mindset

> *"By Chapter 67, Streak is live, observable, tested at every level, and the reader has shipped a feature alone from a one-paragraph brief and written its post-mortem."*

---

# Chapter 56 — Error boundaries, `handleError`, the failure budget

## Lesson 56.1 — `handleError` server and client

```ts
// src/hooks.server.ts
import type { HandleServerError } from '@sveltejs/kit';
import { logger } from '$lib/logger';

export const handleError: HandleServerError = ({ error, event, status, message }) => {
  const errorId = crypto.randomUUID();
  logger.error('unhandled', { errorId, status, message, path: event.url.pathname, error: redact(error) });
  return { message: 'An unexpected error occurred.', code: errorId };
};

function redact(err: unknown): unknown {
  if (err instanceof Error) return { name: err.name, message: err.message };
  return err;
}
```

Same shape in `hooks.client.ts`.

`handleError` must NEVER throw. The `redact` helper strips PII before logging.

---

## Lesson 56.2 — Error IDs surfaced to user

```svelte
<!-- src/routes/+error.svelte -->
<script lang="ts">
  import { page } from '$app/state';
</script>

<h2>Something went wrong</h2>
<p>{page.error?.message}</p>
{#if page.error?.code}
  <small>Error ID: <code>{page.error.code}</code></small>
{/if}
<a href="/">Go home</a>
```

Users can quote the ID to support; we can grep logs.

---

## Lesson 56.3 — `<svelte:boundary>`

```svelte
<svelte:boundary>
  <RiskyChild />
  {#snippet failed(error, reset)}
    <p>This part broke. <button onclick={reset}>Retry</button></p>
  {/snippet}
</svelte:boundary>
```

Granular error catching inside a working page.

---

## End-of-chapter checkpoint

- [ ] Throw a deliberate error; see the polite page.
- [ ] Error ID logged.

---

# Chapter 57 — Environment variables — the four-quadrant matrix

## Lesson 57.1 — The matrix

| | Static (build-time) | Dynamic (runtime) |
|---|---|---|
| **Private** | `$env/static/private` | `$env/dynamic/private` |
| **Public** | `$env/static/public` (`PUBLIC_*`) | `$env/dynamic/public` (`PUBLIC_*`) |

- Build-time + private — DB URL, API secrets baked at build.
- Build-time + public — public site URL, feature flags.
- Runtime + private — values that change per deploy without rebuild.
- Runtime + public — same, public.

---

## Lesson 57.2 — Boot validator

```ts
// src/lib/env.ts
import * as v from 'valibot';
import { DATABASE_URL, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET } from '$env/static/private';

const Schema = v.object({
  DATABASE_URL: v.pipe(v.string(), v.url()),
  STRIPE_SECRET_KEY: v.pipe(v.string(), v.regex(/^sk_/)),
  STRIPE_WEBHOOK_SECRET: v.pipe(v.string(), v.regex(/^whsec_/)),
});

export const env = v.parse(Schema, { DATABASE_URL, STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET });
```

Importing this module from `+hooks.server.ts` runs the validator at boot. If anything is wrong, the server crashes immediately with a clear message — not on the first user request.

---

## End-of-chapter checkpoint

- [ ] All env vars are typed and validated at boot.
- [ ] `.env.example` documents every required var.

---

# Chapter 58 — Vitest unit tests, property-based tests, the test pyramid

## Lesson 58.1 — Test pyramid

- Many unit (fast, cheap).
- Fewer integration (real DB, slower).
- Fewest e2e (full browser, slowest).

---

## Lesson 58.2 — Vitest unit

```ts
// tests/unit/parseHabit.test.ts
import { describe, it, expect } from 'vitest';
import { parseHabit } from '$lib/parseHabit';

describe('parseHabit', () => {
  it('rejects null', () => expect(parseHabit(null).ok).toBe(false));
  it('rejects empty name', () => expect(parseHabit({ id: 'x', name: '', createdAt: 1 }).ok).toBe(false));
  it('accepts valid', () => {
    const r = parseHabit({ id: 'x', name: 'Read', createdAt: 1 });
    expect(r.ok && r.value.name).toBe('Read');
  });
});
```

---

## Lesson 58.3 — Property-based on `Money`

```bash
pnpm add -D fast-check
```

```ts
import * as fc from 'fast-check';
import { splitCents, cents } from '$lib/money';

describe('splitCents', () => {
  it('preserves total', () => {
    fc.assert(fc.property(
      fc.integer({ min: 0, max: 1_000_000 }),
      fc.integer({ min: 1, max: 100 }),
      (total, n) => {
        const parts = splitCents(cents(total), n);
        const sum = parts.reduce((a, b) => a + (b as number), 0);
        return sum === total;
      },
    ));
  });
});
```

Random inputs across the parameter space. Catches off-by-cents instantly.

---

## End-of-chapter checkpoint

- [ ] `pnpm test:unit` runs in <2s.
- [ ] Property-based test on `splitCents` is green.

---

# Chapter 59 — Component, contract, integration tests

## Lesson 59.1 — `@testing-library/svelte`

```bash
pnpm add -D @testing-library/svelte @testing-library/user-event jsdom
```

```ts
// tests/component/HabitRow.svelte.test.ts
import { describe, it, expect } from 'vitest';
import { render, screen } from '@testing-library/svelte';
import userEvent from '@testing-library/user-event';
import HabitRow from '$lib/components/HabitRow.svelte';

describe('HabitRow', () => {
  it('renders name and calls onDelete', async () => {
    let deleted = '';
    render(HabitRow, {
      habit: { id: 'h1' as any, name: 'Read', createdAt: Date.now() },
      onDelete: (id) => { deleted = id; },
    });
    expect(screen.getByText('Read')).toBeInTheDocument();
    await userEvent.click(screen.getByLabelText('Remove Read'));
    expect(deleted).toBe('h1');
  });
});
```

---

## Lesson 59.2 — Integration: real Postgres, truncate-before-each

```ts
import { beforeEach } from 'vitest';
import { db } from '$lib/db/client';
import { sql } from 'drizzle-orm';

beforeEach(async () => {
  await db.execute(sql`TRUNCATE users, habits, sessions, audit_log RESTART IDENTITY CASCADE`);
});
```

---

## Lesson 59.3 — Contract tests

`docs/openapi.yaml` is hand-written; tests assert the live API matches.

---

## End-of-chapter checkpoint

- [ ] Component, contract, integration suites all green.

---

# Chapter 60 — Playwright e2e, a11y, visual regression

## Lesson 60.1 — Playwright config

```ts
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  webServer: { command: 'pnpm build && pnpm preview', port: 4173 },
  use: { baseURL: 'http://localhost:4173' },
  projects: [
    { name: 'chromium', use: devices['Desktop Chrome'] },
    { name: 'firefox', use: devices['Desktop Firefox'] },
  ],
});
```

---

## Lesson 60.2 — Full-flow test

```ts
// tests/e2e/full-flow.spec.ts
import { test, expect } from '@playwright/test';

test('signup → log → upgrade → logout → login → see habits', async ({ page }) => {
  await page.goto('/signup');
  await page.fill('input[name=email]', `user+${Date.now()}@example.com`);
  await page.fill('input[name=password]', 'correct horse battery staple');
  await page.click('button[type=submit]');

  // ... rest of flow ...
});
```

---

## Lesson 60.3 — A11y with axe

```bash
pnpm add -D @axe-core/playwright
```

```ts
import AxeBuilder from '@axe-core/playwright';
test('home is accessible', async ({ page }) => {
  await page.goto('/');
  const results = await new AxeBuilder({ page }).analyze();
  expect(results.violations).toEqual([]);
});
```

---

## Lesson 60.4 — Visual regression

```ts
test('home matches snapshot', async ({ page }) => {
  await page.goto('/');
  await expect(page).toHaveScreenshot();
});
```

First run creates the baseline; subsequent runs compare.

---

## End-of-chapter checkpoint

- [ ] e2e green for the full flow.
- [ ] a11y has zero violations.

---

# Chapter 61 — CI/CD with GitHub Actions

## Lesson 61.1 — The workflow

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push:
    branches: [main]
  pull_request:

jobs:
  test:
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
          --health-cmd pg_isready --health-interval 10s --health-timeout 5s --health-retries 5
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v3
        with: { version: 9 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm check
      - run: pnpm test:unit
      - run: pnpm db:migrate
        env: { DATABASE_URL: postgres://postgres:dev@localhost:5432/streak_test }
      - run: pnpm test:integration
        env: { DATABASE_URL: postgres://postgres:dev@localhost:5432/streak_test }
      - run: pnpm exec playwright install --with-deps
      - run: pnpm test:e2e
      - run: pnpm build
```

---

## Lesson 61.2 — Branch protection

In GitHub: Settings → Branches → main: require all status checks; require linear history; no force push.

---

## End-of-chapter checkpoint

- [ ] PRs run CI.
- [ ] `main` is protected.

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
