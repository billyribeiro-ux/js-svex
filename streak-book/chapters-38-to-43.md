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
import { parseHabits } from '$lib/parseHabit';
import type { Habit } from '$lib/types';

const KEY = 'streak.habits.v1';

export function loadHabits(): Habit[] {
  if (typeof localStorage === 'undefined') return []; // SSR safety
  const raw = localStorage.getItem(KEY);
  if (raw === null) return [];
  try {
    return parseHabits(JSON.parse(raw));
  } catch {
    return [];
  }
}

export function saveHabits(habits: Habit[]): void {
  if (typeof localStorage === 'undefined') return;
  localStorage.setItem(KEY, JSON.stringify(habits));
}
```

The `typeof localStorage === 'undefined'` check is for **SSR safety** — `localStorage` doesn't exist on the server. SvelteKit runs `+page.svelte` on the server during SSR; without this guard, the page would crash there.

> **SSR safety** — code that may run on both server and client must check for browser-only globals.

---

## Lesson 38.4 — Wiring with `$effect`

Inside the `HabitStore` (or the page):

```ts
import { loadHabits, saveHabits } from '$lib/storage.svelte';

export class HabitStore {
  habits = $state<Habit[]>(loadHabits());

  // ... existing methods ...
}
```

And separately, an effect that saves:

```ts
$effect(() => {
  saveHabits(store.habits);
});
```

Read aloud: *"whenever the habits change, save them."* This is one of the *legitimate* uses of `$effect` — synchronising state to an external store (here: `localStorage`).

Save. Add habits. Refresh the browser. They're still there.

---

## Lesson 38.5 — Build, break, fix

Open dev tools → Application → Local Storage → your origin. Edit the JSON to be malformed. Refresh. Your `parseHabits` rejects the bad data; the list resets to empty. The boundary parser catches the bad input. Senior win.

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

```bash
pnpm exec drizzle-kit generate
```

This creates `drizzle/0000_*.sql`. Inspect it. Apply it:

```bash
pnpm exec drizzle-kit migrate
```

Confirm:

```bash
psql streak_dev -c "\d habits"
```

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

`+page.server.ts`:

```ts
// src/routes/(app)/+page.server.ts
import type { PageServerLoad } from './$types';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
import { eq, desc } from 'drizzle-orm';

export const load: PageServerLoad = async ({ locals }) => {
  // For now we fake locals.user. Real auth in Part VII.
  const userId = 'demo-user';

  const rows = await db.select().from(habits).where(eq(habits.userId, userId)).orderBy(desc(habits.createdAt));
  return { habits: rows };
};
```

For now you'll need to insert a demo user manually:

```sql
INSERT INTO users (id, email, password_hash) VALUES ('demo-user'::uuid, 'demo@example.com', 'fake');
```

(Adjust the cast — UUIDs are not arbitrary strings. Use a real UUID. Or seed via a `pnpm db:seed` script.)

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
// src/routes/(app)/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { habits } from '$lib/db/schema';
import { eq, and } from 'drizzle-orm';

export const actions: Actions = {
  addHabit: async ({ request, locals }) => {
    const userId = 'demo-user';
    const data = await request.formData();
    const name = String(data.get('name') ?? '').trim();
    if (name === '') return fail(400, { fieldErrors: { name: 'Name required' } });
    await db.insert(habits).values({ userId, name });
    return { success: true };
  },
  deleteHabit: async ({ request, locals }) => {
    const userId = 'demo-user';
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
  const parsed = safeParse(HabitInputSchema, {
    name: data.get('name'),
    description: data.get('description') || undefined,
  });
  if (!parsed.success) {
    return fail(400, { fieldErrors: parsed.issues.reduce((acc, i) => ({ ...acc, [i.path?.[0]?.key ?? 'form']: i.message }), {}) });
  }
  // parsed.output is HabitInput
  await db.insert(habits).values({ userId, ...parsed.output });
  return { success: true };
},
```

Cleaner. Faster to write. Same boundary safety.

---

## Lesson 41.4 — The JS-disabled test (runtime evidence)

In dev tools, *Settings* → *Disable JavaScript*. Reload. Add a habit. Watch the page reload after submit. **It still works.** That's progressive enhancement.

Re-enable JS. Same flow now feels instant.

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
  const userId = 'demo-user';
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

Set up a separate test DB. In `vitest.config.ts`:

```ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    setupFiles: ['./tests/setup.ts'],
  },
});
```

```ts
// tests/setup.ts
import { execSync } from 'node:child_process';
beforeAll(() => execSync('pnpm db:test:reset'));
```

The test:

```ts
// tests/integration/addHabit.atomic.test.ts
import { describe, it, expect } from 'vitest';
import { db } from '$lib/db/client';

describe('addHabit atomic', () => {
  it('rejects concurrent inserts at the limit', async () => {
    // seed user with 49/50 habits
    // ...
    const promises = Array.from({ length: 10 }, () => addHabit('demo-user', 'concurrent'));
    const results = await Promise.allSettled(promises);
    const successes = results.filter((r) => r.status === 'fulfilled');
    expect(successes.length).toBe(1); // only one slot left
  });
});
```

Bible rule #5 in action: never mock the DB.

---

## End-of-chapter checkpoint

- [ ] You added `habits_count` and the atomic UPDATE.
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

  let pendingDeletes: Set<string> = $state(new Set());
</script>

<form method="POST" action="?/deleteHabit" use:enhance={({ formData }) => {
  const id = String(formData.get('id'));
  pendingDeletes = new Set([...pendingDeletes, id]);

  return async ({ result }) => {
    pendingDeletes = new Set([...pendingDeletes].filter((p) => p !== id));
    if (result.type === 'failure' || result.type === 'error') {
      // surface error via toast (Ch 43 introduces toast)
      await applyAction(result);
    } else {
      await invalidate('streak:habits');
    }
  };
}}>
  <input type="hidden" name="id" value={habit.id} />
  <button type="submit" aria-label="Delete">×</button>
</form>
```

The visible list filters out `pendingDeletes`:

```ts
const visibleHabits: Habit[] = $derived(
  data.habits.filter((h) => !pendingDeletes.has(h.id))
);
```

Click ×; the row vanishes immediately. The fetch happens in the background. On failure, the row comes back.

---

## Lesson 43.2 — The CLS landmine, named again

**Don't flip a `loading: true` flag during this kind of mutation.** If you did, the row would vanish optimistically, *then* the loading skeleton would briefly replace it, then the refetch would replace the skeleton. Three layouts in 1 second. CLS spike. Users feel it.

Bible rule #16. Senior eyes spot this in code review immediately.

---

## Lesson 43.3 — Toast helper

```ts
// src/lib/toast.svelte.ts
type Toast = { id: string; message: string; kind: 'info' | 'success' | 'error' };

class ToastStore {
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

export const toast = new ToastStore(); // OK as a module-scoped client-side singleton (we'll context-ise on a per-user basis once we have auth)
```

Render in layout:

```svelte
<div class="toast-stack">
  {#each toast.toasts as t (t.id)}
    <div class="toast toast-{t.kind}">{t.message}</div>
  {/each}
</div>
```

---

## End-of-chapter checkpoint

- [ ] Optimistic delete works.
- [ ] On simulated failure, the row returns.
- [ ] No CLS-flash during the operation.
- [ ] Toast surfaces errors.

End of Part VI. Next: real users.
