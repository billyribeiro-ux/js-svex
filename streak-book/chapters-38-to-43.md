# Part VI — Data that survives a refresh

> *"Persistence felt before it's networked. localStorage first; then Postgres via Drizzle, server load, form actions, atomic conditional UPDATEs against TOCTOU, integration tests against a real DB, optimistic UI without CLS."*

---

# Chapter 38 — `localStorage` — persistence you can feel

> *Today's job:* refresh the browser; habits are still there. Visible win: hit reload — list intact.

---

## Lesson 38.1 — `localStorage` 101

`localStorage` is a per-origin, ~5 MB, string-only key-value store inside the browser.

```ts
localStorage.setItem('greeting', 'hello');
const value: string | null = localStorage.getItem('greeting');
localStorage.removeItem('greeting');
localStorage.clear();
```

It's *synchronous* (slightly slow) and *string-only* (you JSON-encode anything else).

---

## Lesson 38.2 — JSON round-trip

```ts
const habits: Habit[] = [/* ... */];
localStorage.setItem('streak.habits', JSON.stringify(habits));

const raw = localStorage.getItem('streak.habits');
const parsed: unknown = raw === null ? [] : JSON.parse(raw);
```

But `JSON.parse` returns `unknown` to us morally — we don't trust the shape. Run it through `parseHabit` (Chapter 26).

---

## Lesson 38.3 — A typed storage helper

```ts
// src/lib/storage.svelte.ts
import { browser } from '$app/environment';
import { parseHabits } from '$lib/parseHabit';
import type { Habit } from '$lib/types';

const KEY = 'streak.habits.v1';

export function loadHabits(): Habit[] {
  if (!browser) return []; // SSR safety
  const raw = localStorage.getItem(KEY);
  if (raw === null) return [];
  try {
    return parseHabits(JSON.parse(raw));
  } catch {
    return [];
  }
}

export function saveHabits(habits: Habit[]): void {
  if (!browser) return;
  localStorage.setItem(KEY, JSON.stringify(habits));
}
```

The `import { browser } from '$app/environment'` is **SvelteKit's canonical SSR safety primitive** — `browser` is `true` only in the browser, `false` during SSR or server-side rendering. Use it instead of `typeof localStorage === 'undefined'`; it's typed, intentional, and reads better.

> **`$app/environment`** — exports `browser`, `dev`, `building`, `version`. The senior way to gate browser-only or build-only code.

---

## Lesson 38.4 — Wiring with `$effect`

Two separate edits.

**1. In `src/lib/habits.svelte.ts`** — initialise from storage:

```ts
import { loadHabits, saveHabits } from '$lib/storage.svelte';

export class HabitStore {
  habits = $state<Habit[]>(loadHabits());

  // ... existing methods ...
}
```

**2. In `+page.svelte`** (or wherever the store is consumed) — save on every change:

```svelte
<script lang="ts">
  import { getHabitStore } from '$lib/contexts';
  const store = getHabitStore();

  $effect(() => {
    // saveHabits is read; touching `store.habits` makes this effect track it.
    saveHabits(store.habits);
  });
</script>
```

`$effect` only runs inside a component or inside `$effect.root` — *not* at module scope, *not* inside class fields or methods. That's why it lives in the consuming `+page.svelte`, not in the store class.

Read aloud: *"whenever the habits change, save them."* This is one of the *legitimate* uses of `$effect` — synchronising state to an external store (here: `localStorage`).

Save (`Cmd+S` / `Ctrl+S`). Add habits. Refresh the browser. They're still there.

---

## Lesson 38.5 — Build, break, fix

Open dev tools → Application → Local Storage → your origin. Edit the JSON to be malformed. Refresh. Your `parseHabits` rejects the bad data; the list resets to empty. The boundary parser catches the bad input. Senior win.

---

## Lesson 38.6 — Recurring concepts from earlier chapters

- **`parseHabits`** (Ch 26) — the same boundary parser, now wrapping `JSON.parse`.
- **`$effect` for legitimate side effects** (Ch 17) — syncing state to an external store is exactly the right use.
- **`HabitStore`** (Ch 29) — initialised from storage instead of empty.

---

## Lesson 38.7 — What you can now read in the wild

After Chapter 38 you can:

- Read **`localStorage.getItem` / `setItem` / `removeItem` / `clear`**.
- Read **`JSON.stringify` / `JSON.parse`** and know the round-trip data-loss class.
- Read **`import { browser } from '$app/environment'`** as the SvelteKit SSR-safety primitive (and recognise `typeof localStorage === 'undefined'` as the older equivalent).
- Spot **untrusted JSON entering the program** as a parser-shaped boundary.

---

## End-of-chapter checkpoint

- [ ] Habits survive a refresh.
- [ ] Manually corrupting `localStorage` doesn't crash; the list resets.

---

# Chapter 39 — The first schema, migrations, and Drizzle

> *Today's job:* a real Postgres database with a `habits` table, typed end-to-end. Visible win: `pnpm db:migrate` creates the table; `psql` confirms.

---

## Lesson 39.1 — Postgres locally

Pick one:

**Docker:**
```bash
docker compose up -d postgres
```
With a `compose.yaml`:
```yaml
services:
  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: streak_dev
    ports:
      - 5432:5432
    volumes:
      - streak-pg:/var/lib/postgresql/data
volumes:
  streak-pg:
```

**Or Homebrew (macOS):**
```bash
brew install postgresql@16
brew services start postgresql@16
createdb streak_dev
```

---

## Lesson 39.2 — Drizzle setup

```bash
pnpm add drizzle-orm postgres
pnpm add -D drizzle-kit
```

`drizzle.config.ts` (project root):

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './src/lib/db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL ?? 'postgres://postgres:dev@localhost:5432/streak_dev' },
  strict: true,
});
```

`.env`:

```
DATABASE_URL=postgres://postgres:dev@localhost:5432/streak_dev
```

`.env` is gitignored. `.env.example` is committed (without secrets).

---

## Lesson 39.3 — The schema

```ts
// src/lib/db/schema.ts
import { pgTable, uuid, text, timestamp } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  email: text('email').notNull().unique(),
  passwordHash: text('password_hash').notNull(),
  role: text('role', { enum: ['user', 'admin'] }).notNull().default('user'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const habits = pgTable('habits', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  name: text('name').notNull(),
  description: text('description'),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export type DbHabit = typeof habits.$inferSelect;
export type NewDbHabit = typeof habits.$inferInsert;
```

Notes:
- **`uuid` PK** — global unique. Doesn't leak row counts.
- **`withTimezone: true`** — TZ-aware. Senior habit; never store naive timestamps.
- **`references(...).onDelete: 'cascade'`** — deleting a user deletes their habits.
- **`role: text(..., { enum: [...] })`** — Postgres enum-shaped check, typed in Drizzle.

---

## Lesson 39.4 — Generate and run the migration

Add convenience scripts to `package.json`:

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:studio": "drizzle-kit studio"
  }
}
```

Then:

```bash
pnpm db:generate
```

This creates `drizzle/0000_*.sql`. Inspect it. Apply it:

```bash
pnpm db:migrate
```

Confirm:

```bash
psql streak_dev -c "\d habits"
```

`pnpm db:studio` opens a browser-based DB explorer — handy for sanity checks.

---

## Lesson 39.5 — `EXPLAIN ANALYZE`

The runtime-evidence rule. Insert a row:

```sql
INSERT INTO users (email, password_hash) VALUES ('test@example.com', 'fake');
INSERT INTO habits (user_id, name) VALUES ((SELECT id FROM users LIMIT 1), 'Test');
EXPLAIN ANALYZE SELECT * FROM habits WHERE user_id = (SELECT id FROM users LIMIT 1);
```

Read the plan. At first there's no `user_id` index, so the planner does a sequential scan. We'll add the index when we have a query that needs it (Ch 55).

---

## Lesson 39.6 — The forward-only rule

Once a migration is committed and applied, **never edit it**. Schema changes ship as new migrations. `IF NOT EXISTS` on every `CREATE`. `CREATE INDEX CONCURRENTLY` requires `-- no-transaction`. Bible rule #18.

---

## Lesson 39.7 — Recurring concepts from earlier chapters

- **`type` declarations** (Ch 9) — Drizzle's `$inferSelect` / `$inferInsert` give you typed row shapes.
- **`as const`** (Ch 25) — `enum: ['user', 'admin']` narrows to a literal union.
- **Forward-only rule** (Bible #18) — applied to migrations from day one.

---

## Lesson 39.8 — What you can now read in the wild

After Chapter 39 you can:

- Read a Drizzle schema with `pgTable`, `uuid`, `text`, `timestamp`, `integer`, `boolean`.
- Read `references(() => other.id, { onDelete: 'cascade' })` for foreign keys.
- Read `EXPLAIN ANALYZE` output and tell sequential scan from index scan.
- Spot a missing `withTimezone: true` on a timestamp column.

---

## End-of-chapter checkpoint

- [ ] Postgres is running locally.
- [ ] `users` and `habits` tables exist.
- [ ] You ran `EXPLAIN ANALYZE` and read the plan.

---

# Chapter 40 — `+page.server.ts` and the server load

> *Today's job:* the home page loads habits from Postgres. Visible win: open Streak in two browsers; add a habit in one; refresh the other; it's there.

---

## Lesson 40.1 — `+page.server.ts`

```ts
// src/lib/db/client.ts
import postgres from 'postgres';
import { drizzle } from 'drizzle-orm/postgres-js';
import { DATABASE_URL } from '$env/static/private';

const client = postgres(DATABASE_URL, {
  max: 10,
  idle_timeout: 20,
  connect_timeout: 10,
});

export const db = drizzle(client);
```

`max: 10` is the pool size. `connect_timeout: 10` is the Bible rule applied — never let a hung handshake hang a request.

`+page.server.ts` (lives at `(app)/dashboard/+page.server.ts` per Ch 37's split):

```ts
// src/routes/(app)/dashboard/+page.server.ts
import type { PageServerLoad } from './$types';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
import { eq, desc } from 'drizzle-orm';

const DEMO_USER_ID = '00000000-0000-0000-0000-000000000001'; // a real UUID we seed below

export const load: PageServerLoad = async ({ locals }) => {
  // For now we use a hardcoded demo user. Real `locals.user` in Part VII.
  const userId = DEMO_USER_ID;

  const rows = await db.select().from(habits).where(eq(habits.userId, userId)).orderBy(desc(habits.createdAt));
  return { habits: rows };
};
```

Seed the demo user once:

```sql
INSERT INTO users (id, email, password_hash)
VALUES ('00000000-0000-0000-0000-000000000001', 'demo@example.com', 'fake')
ON CONFLICT (id) DO NOTHING;
```

(Real UUID — `'demo-user'` would have failed the UUID type check. Better in a real app: a `pnpm db:seed` script that uses `crypto.randomUUID()` and writes the chosen ID into a dev-only `.env.seed`.)

---

## Lesson 40.2 — `$env/static/private`

Server-only secrets are imported via `$env/static/private`. Try to import it from `+page.svelte`; the build fails. **The reader sees the error deliberately.** That's the runtime-evidence layer for Bible rule #19.

---

## Lesson 40.3 — Serialisation through `devalue`

`load`'s return is serialised via `devalue` — handles Date, Map, Set, BigInt, RegExp, even circular refs. Class instances with methods don't survive (just data).

---

## Lesson 40.4 — Now you write it

**The English sentence first:**

> *"Add a `/stats` route's server load that returns counts per month using SQL `date_trunc('month', created_at)`."*

---

## Lesson 40.5 — Recurring concepts from earlier chapters

- **`+page.server.ts`** (Ch 31's split, formal coverage now) — server-only `load`.
- **`$env/static/private`** (Ch 31, Bible rule #19) — secrets safe at build time.
- **DB pool timeouts** — Bible rule #13 applied.

---

## Lesson 40.6 — What you can now read in the wild

After Chapter 40 you can:

- Read **`+page.server.ts`** with `PageServerLoad` and `Actions`.
- Read **DB pool config** (`max`, `idle_timeout`, `connect_timeout`) and explain each.
- Tell which env-var quadrant a given import comes from (private/public × static/dynamic).
- Read **`devalue`-serialised** load output and know what survives the wire.

---

## End-of-chapter checkpoint

- [ ] Open the app in two tabs; data is shared (after refresh).
- [ ] You handled `$env/static/private` correctly.
- [ ] You set DB pool timeouts.

---

# Chapter 41 — Form actions and progressive enhancement

> *Today's job:* "add habit" and "delete habit" go through server actions. Visible win: works without JavaScript; instant with it.

---

## Lesson 41.1 — Form actions

```ts
// src/routes/(app)/dashboard/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
import { eq, and } from 'drizzle-orm';

// Same demo UUID we seeded in Ch 40. Real auth replaces this in Part VII.
const DEMO_USER_ID = '00000000-0000-0000-0000-000000000001';

export const actions: Actions = {
  addHabit: async ({ request, locals }) => {
    const userId = DEMO_USER_ID;
    const data = await request.formData();
    const name = String(data.get('name') ?? '').trim();
    if (name === '') return fail(400, { fieldErrors: { name: 'Name required' } });
    await db.insert(habits).values({ userId, name });
    return { success: true };
  },
  deleteHabit: async ({ request, locals }) => {
    const userId = DEMO_USER_ID;
    const data = await request.formData();
    const id = String(data.get('id') ?? '');
    await db.delete(habits).where(and(eq(habits.id, id), eq(habits.userId, userId)));
    return { success: true };
  },
};
```

In markup:

```svelte
<script lang="ts">
  import { enhance } from '$app/forms';
  import type { PageProps } from './$types';
  let { data, form }: PageProps = $props();
</script>

<form method="POST" action="?/addHabit" use:enhance>
  <input name="name" required />
  <button type="submit">Add</button>
  {#if form?.fieldErrors?.name}
    <p class="error">{form.fieldErrors.name}</p>
  {/if}
</form>

{#each data.habits as habit (habit.id)}
  <li>
    {habit.name}
    <form method="POST" action="?/deleteHabit" use:enhance>
      <input type="hidden" name="id" value={habit.id} />
      <button type="submit" aria-label="Delete">×</button>
    </form>
  </li>
{/each}
```

> **`use:enhance`** — Svelte directive that progressively enhances a form: with JS, the submit happens via `fetch` and the page revalidates without a full reload; without JS, the form submits the old-fashioned way. Same code path either way.

---

## Lesson 41.2 — `fail` vs `redirect` vs `error`

- `return fail(400, { ... })` — validation failed; render the same page with the data.
- `redirect(303, '/somewhere')` — concluded; go elsewhere.
- `error(500, '...')` — unexpected; renders `+error.svelte`.

---

## Lesson 41.3 — Switching `parseHabit` to Valibot

Now that the reader has felt the cost of hand-rolling `parseHabit`, drop in Valibot:

```bash
pnpm add valibot
```

```ts
// src/lib/validation/schemas.ts
import * as v from 'valibot';

export const HabitInputSchema = v.object({
  name: v.pipe(v.string(), v.trim(), v.minLength(1, 'Name required'), v.maxLength(100)),
  description: v.optional(v.pipe(v.string(), v.maxLength(500))),
});

export type HabitInput = v.InferOutput<typeof HabitInputSchema>;
```

In actions:

```ts
import { safeParse } from 'valibot';
import { HabitInputSchema } from '$lib/validation/schemas';

addHabit: async ({ request }) => {
  const data = await request.formData();
  // FormData.get returns string | File | null; we only accept strings.
  const rawName = data.get('name');
  const rawDesc = data.get('description');

  const parsed = safeParse(HabitInputSchema, {
    name: typeof rawName === 'string' ? rawName : '',
    description: typeof rawDesc === 'string' && rawDesc !== '' ? rawDesc : undefined,
  });
  if (!parsed.success) {
    const fieldErrors: Record<string, string> = {};
    for (const issue of parsed.issues) {
      const key = issue.path?.[0]?.key;
      if (typeof key === 'string') fieldErrors[key] = issue.message;
    }
    return fail(400, { fieldErrors });
  }
  // parsed.output is HabitInput
  await db.insert(habits).values({ userId: DEMO_USER_ID, ...parsed.output });
  return { success: true };
},
```

Cleaner. Faster to write. Same boundary safety. The explicit `typeof rawName === 'string'` checks beat `|| undefined` (Bible rule #5: never `||` for nullish defaults), and the imperative `for ... of` reduce avoids `acc` spread allocations on every issue.

---

## Lesson 41.4 — The JS-disabled test (runtime evidence)

In dev tools, *Settings* → *Disable JavaScript*. Reload. Add a habit. Watch the page reload after submit. **It still works.** That's progressive enhancement.

Re-enable JS. Same flow now feels instant.

---

## Lesson 41.5 — Recurring concepts from earlier chapters

- **`+page.server.ts`** (Ch 40) — actions live alongside the load.
- **Boundary parser** (Ch 26) — Valibot replaces hand-rolled `parseHabit` for form input.
- **`use:enhance`** — progressive-enhancement primitive.
- **Bible rule #5** — explicit `typeof === 'string'` instead of `||`.

---

## Lesson 41.6 — What you can now read in the wild

After Chapter 41 you can:

- Read **`export const actions: Actions = { ... } satisfies Actions`** with named actions.
- Read **`fail(400, { ... })`**, **`redirect(303, '/...')`**, **`error(500, '...')`** and pick the right one.
- Read **`<form method="POST" action="?/named" use:enhance>`** and run the JS-disabled test mentally.
- Read a Valibot schema and explain what passes/fails.

---

## End-of-chapter checkpoint

- [ ] Add and delete go through server actions.
- [ ] Validation errors render inline.
- [ ] You tested with JS disabled.

---

# Chapter 42 — Atomic mutations, TOCTOU, concurrency hygiene

> *Today's job:* a per-user habits-cap. Two concurrent inserts at the cap don't both succeed. Visible win: an integration test fires 10 concurrent adds at the cap; exactly the right number succeed.

This is a level-7 cornerstone. Read carefully.

---

## Lesson 42.1 — Race conditions and TOCTOU

```ts
// ❌ TOCTOU
const count = await db.select({ c: count() }).from(habits).where(eq(habits.userId, userId));
if (count[0].c >= MAX) return fail(400, { error: 'limit' });
await db.insert(habits).values({ userId, name });
```

Between the `SELECT` and the `INSERT`, another request can sneak in. Both pass the check; both insert. The cap is breached.

> **TOCTOU** — *Time of Check, Time of Use.* The window between checking and acting is exploitable.

---

## Lesson 42.2 — The atomic alternative

```sql
UPDATE users
   SET habits_count = habits_count + 1
 WHERE id = $1
   AND habits_count < $2
RETURNING habits_count;
```

If zero rows return, we lost the race. Reject. If one row returns, we won; insert the habit.

Add a `habits_count` column to `users` via a new migration:

```ts
// in schema.ts
export const users = pgTable('users', {
  // ... existing ...
  habitsCount: integer('habits_count').notNull().default(0),
});
```

In Drizzle, the atomic update:

```ts
import { sql } from 'drizzle-orm';

const MAX_HABITS = 50;

addHabit: async ({ request, locals }) => {
  const userId = DEMO_USER_ID;
  // ... parse name ...

  const updated = await db.update(users)
    .set({ habitsCount: sql`${users.habitsCount} + 1` })
    .where(and(eq(users.id, userId), lt(users.habitsCount, MAX_HABITS)))
    .returning({ count: users.habitsCount });

  if (updated.length === 0) {
    return fail(400, { error: `Habit limit (${MAX_HABITS}) reached` });
  }

  await db.insert(habits).values({ userId, name });
  return { success: true };
},
```

To handle `delete`, the symmetric decrement:

```ts
.set({ habitsCount: sql`${users.habitsCount} - 1` })
.where(and(eq(users.id, userId), gt(users.habitsCount, 0)))
```

---

## Lesson 42.3 — Transactions

When two operations need to *both* succeed or *both* fail:

```ts
await db.transaction(async (tx) => {
  await tx.update(users).set(/* ... */).where(/* ... */);
  await tx.insert(habits).values(/* ... */);
});
```

If anything throws inside, the whole transaction rolls back. Use it for the *update + insert* pair if you want atomicity.

---

## Lesson 42.4 — Audit log atomically

When the audit row is required, it goes inside the same transaction:

```ts
await db.transaction(async (tx) => {
  await tx.insert(habits).values({ userId, name });
  await tx.insert(auditLog).values({ actorId: userId, action: 'habit.add', targetId: /* habit id */ });
});
```

Bible rule: never write the audit *after* a "best-effort" log call. Either it's atomic with the action or it didn't happen.

---

## Lesson 42.5 — Integration test against real Postgres

For an integration test, the form-action callback isn't directly callable. **Extract the logic into a pure async function** that the action *and* the test can both call:

```ts
// src/lib/habits-server.ts
import { db } from '$lib/db/client';
import { habits, users } from '$lib/db/schema';
import { and, eq, lt, sql } from 'drizzle-orm';
import { ok, err, type Result } from '$lib/types';

const MAX_HABITS = 50;

export async function addHabitForUser(
  userId: string,
  name: string,
): Promise<Result<{ id: string }, 'limit-reached'>> {
  return db.transaction(async (tx) => {
    const updated = await tx.update(users)
      .set({ habitsCount: sql`${users.habitsCount} + 1` })
      .where(and(eq(users.id, userId), lt(users.habitsCount, MAX_HABITS)))
      .returning({ count: users.habitsCount });

    if (updated.length === 0) return err('limit-reached');

    const [inserted] = await tx.insert(habits).values({ userId, name }).returning({ id: habits.id });
    if (inserted === undefined) throw new Error('insert returned no rows'); // genuine programmer error
    return ok({ id: inserted.id });
  });
}
```

The action becomes a thin wrapper:

```ts
addHabit: async ({ request }) => {
  // ... parse name ...
  const result = await addHabitForUser(DEMO_USER_ID, parsed.output.name);
  if (!result.ok) return fail(400, { error: `Habit limit (${MAX_HABITS}) reached` });
  return { success: true };
},
```

Now the integration test:

```ts
// tests/integration/addHabit.atomic.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { db } from '$lib/db/client';
import { users, habits } from '$lib/db/schema';
import { sql } from 'drizzle-orm';
import { addHabitForUser } from '$lib/habits-server';

const TEST_USER = '00000000-0000-0000-0000-000000000099';

beforeEach(async () => {
  await db.execute(sql`TRUNCATE habits, users RESTART IDENTITY CASCADE`);
  await db.insert(users).values({
    id: TEST_USER,
    email: 'test@example.com',
    passwordHash: 'fake',
    habitsCount: 49, // one slot left
  });
});

describe('addHabitForUser atomic', () => {
  it('rejects concurrent inserts at the limit', async () => {
    const promises = Array.from({ length: 10 }, () => addHabitForUser(TEST_USER, 'concurrent'));
    const results = await Promise.all(promises);
    const successes = results.filter((r) => r.ok);
    expect(successes.length).toBe(1); // only one slot left
  });
});
```

Bible rule #5 in action: never mock the DB. The test runs against a real `streak_test` Postgres; concurrent calls hit the atomic UPDATE; exactly one wins.

> **A note on `auditLog`** referenced in Lesson 42.4: that table lands formally in Chapter 47 (audit log + RBAC). For Chapter 42, the *pattern* of "mutation + audit inside one transaction" is what matters; you can stub `tx.insert(auditLog)` for now or wait until Ch 47 to wire it up.

---

## Lesson 42.6 — Recurring concepts from earlier chapters

- **`Result<T, E>`** (Ch 27) — `addHabitForUser` returns one.
- **The TOCTOU rule** — Bible rule #11 applied as the cornerstone of every read-modify-write.
- **Truncate-before-each** — fast test isolation pattern (formal coverage Ch 59).

---

## Lesson 42.7 — What you can now read in the wild

After Chapter 42 you can:

- Read **`UPDATE … SET … WHERE … RETURNING …`** as the atomic conditional update.
- Read **`db.transaction(async (tx) => { ... })`** and explain the rollback semantics.
- Spot a **SELECT-then-UPDATE** in a code review and replace it.
- Write an **integration test** against a real Postgres with truncate-before-each isolation.

---

## End-of-chapter checkpoint

- [ ] You added `habits_count` and the atomic UPDATE.
- [ ] You extracted `addHabitForUser` as a testable pure-async function.
- [ ] You wrote the concurrent-insert integration test.
- [ ] It runs against a real Postgres.

---

# Chapter 43 — Optimistic UI without the skeleton flash

> *Today's job:* deleting a habit feels instant on a slow connection; failures roll back. Visible win: throttled "Slow 3G" — delete still vanishes immediately; on simulated failure, it reappears.

---

## Lesson 43.1 — The optimistic pattern

```svelte
<script lang="ts">
  import { enhance, applyAction } from '$app/forms';
  import { invalidate } from '$app/navigation';
  import { toast } from '$lib/toast.svelte';
  import type { PageProps } from './$types';

  let { data }: PageProps = $props();

  let pendingDeletes: Set<string> = $state(new Set());

  const visibleHabits = $derived(
    data.habits.filter((h) => !pendingDeletes.has(h.id))
  );
</script>

<ul>
  {#each visibleHabits as habit (habit.id)}
    <li>
      {habit.name}
      <form method="POST" action="?/deleteHabit" use:enhance={({ formData, cancel }) => {
        const raw = formData.get('id');
        if (typeof raw !== 'string' || raw === '') {
          // Defensive: if the hidden input is missing or wrong, bail out.
          // This shouldn't happen in our markup, but `String(null)` would
          // produce the literal string "null" and we'd add it to pendingDeletes.
          cancel();
          return;
        }
        const id = raw;
        pendingDeletes = new Set([...pendingDeletes, id]);

        return async ({ result }) => {
          pendingDeletes = new Set([...pendingDeletes].filter((p) => p !== id));
          if (result.type === 'failure' || result.type === 'error') {
            toast.show("Couldn't delete habit. Please try again.", 'error');
            await applyAction(result);
          } else {
            await invalidate('streak:habits');
          }
        };
      }}>
        <input type="hidden" name="id" value={habit.id} />
        <button type="submit" aria-label="Delete {habit.name}">×</button>
      </form>
    </li>
  {/each}
</ul>
```

Click ×; the row vanishes immediately because `pendingDeletes` filters it out of `visibleHabits`. The fetch happens in the background. On failure: a toast appears and the row comes back. On success: `invalidate('streak:habits')` re-runs the load, the row is gone for real.

> **`enhance`** runs the inner callback synchronously (the optimistic update) and then *returns* an async finalisation function. That's the shape `({ formData }) => async ({ result, update }) => { ... }`. Read it as: *"on submit, run the optimistic update; on response, run the reconciliation."*

---

## Lesson 43.2 — The CLS landmine, named again

**Don't flip a `loading: true` flag during this kind of mutation.** If you did, the row would vanish optimistically, *then* the loading skeleton would briefly replace it, then the refetch would replace the skeleton. Three layouts in 1 second. CLS spike. Users feel it.

Bible rule #16. Senior eyes spot this in code review immediately.

---

## Lesson 43.3 — Toast helper

```ts
// src/lib/toast.svelte.ts
type Toast = { id: string; message: string; kind: 'info' | 'success' | 'error' };

export class ToastStore {
  toasts = $state<Toast[]>([]);

  show(message: string, kind: Toast['kind'] = 'info'): void {
    const id = crypto.randomUUID();
    this.toasts = [...this.toasts, { id, message, kind }];
    setTimeout(() => this.dismiss(id), 4000);
  }

  dismiss(id: string): void {
    this.toasts = this.toasts.filter((t) => t.id !== id);
  }
}
```

We *don't* `export const toast = new ToastStore()` — that's the SSR-singleton landmine from Ch 29. A module-scope instance lives once per server process; on Vercel that means *all SSR-rendered pages share the same toasts array*. Instead, instantiate per-component-tree via context, like `HabitStore` in Ch 32:

```ts
// src/lib/contexts.ts (extend)
const TOAST_KEY = Symbol('toast');

export function setToastStore(store: ToastStore): void {
  setContext(TOAST_KEY, store);
}

export function getToastStore(): ToastStore {
  const store = getContext<ToastStore | undefined>(TOAST_KEY);
  if (store === undefined) throw new Error('ToastStore not in context');
  return store;
}
```

In the root layout:

```svelte
<!-- src/routes/+layout.svelte -->
<script lang="ts">
  import { ToastStore } from '$lib/toast.svelte';
  import { setToastStore } from '$lib/contexts';

  const toasts = new ToastStore();
  setToastStore(toasts);
</script>

<div class="toast-stack">
  {#each toasts.toasts as t (t.id)}
    <div class="toast toast-{t.kind}">{t.message}</div>
  {/each}
</div>
```

Anywhere a child needs to push a toast (the optimistic-delete handler above, for instance):

```svelte
<script lang="ts">
  import { getToastStore } from '$lib/contexts';
  const toast = getToastStore();
</script>
```

Per-tree instance. Per-request safe.

---

## Lesson 43.4 — Recurring concepts from earlier chapters

Part VI's spine, in one place:

- **`localStorage` first, DB second** (Ch 38 → 40) — persistence felt before networked.
- **Boundary parsing** at every input edge (Ch 26, 31, 38, 41).
- **Atomic UPDATE-RETURNING** (Ch 42) — the cornerstone of safe mutations.
- **Form actions + `use:enhance`** (Ch 41) — progressive enhancement.
- **The CLS landmine** (Ch 22, named again here).

---

## Lesson 43.5 — What you can now read in the wild

After Part VI you can:

- Read a `+page.server.ts` with `load` + `actions` + Drizzle queries.
- Spot a **SELECT-then-UPDATE TOCTOU** in code review.
- Spot a **CLS-flash** optimistic-update bug (loading flag flipped during a mutation that already has its own optimistic representation).
- Write an integration test that hits a real Postgres with truncate-before-each isolation.
- Tell which `+page.*` / `+server.ts` file does what, and why.

---

## End-of-chapter checkpoint

- [ ] Optimistic delete works.
- [ ] On simulated failure, the row returns.
- [ ] No CLS-flash during the operation.
- [ ] Toast surfaces errors.

End of Part VI. Next: real users.
