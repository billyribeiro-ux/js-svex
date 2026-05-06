# Part VII — Real users

> *"By Chapter 48 the reader has built a real, audited auth system from scratch — sessions, secure cookies, Argon2id on a worker thread, sign-up + log-in + log-out + email verification, RBAC + audit + impersonation, rate-limiting + CSP/HSTS, and a written security review."*

---

# Chapter 44 — Cookies, sessions, and the threat model

> *Today's job:* a "log in" form that sets a session cookie. Visible win: log in once; refresh anywhere — still logged in.

---

## Lesson 44.1 — Sessions vs JWTs

- **Sessions** — a row in `sessions` table; the cookie holds a random token. Pros: revocable instantly. Cons: DB hit per request.
- **JWTs** — self-contained signed tokens. Pros: no DB hit. Cons: hard to revoke before expiry.

We pick **sessions**. Streak is a fintech-shaped app; revocability matters more than latency. (The DB hit is cheap with proper indexing.)

---

## Lesson 44.2 — The schema additions

```ts
// schema.ts additions
export const sessions = pgTable('sessions', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  token: text('token').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  fresh: boolean('fresh').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const emailVerifications = pgTable('email_verifications', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  token: text('token').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
});
```

Generate and apply.

---

## Lesson 44.3 — The session lifecycle helper

```ts
// src/lib/sessions.ts
import { db } from '$lib/db/client';
import { sessions } from '$lib/db/schema';
import { and, eq, gt } from 'drizzle-orm';
import { randomBytes, timingSafeEqual } from 'node:crypto';

const SESSION_LIFETIME_MS = 30 * 24 * 60 * 60 * 1000;

export function newSessionToken(): string {
  return randomBytes(32).toString('hex');
}

export async function createSession(userId: string): Promise<string> {
  const token = newSessionToken();
  const expiresAt = new Date(Date.now() + SESSION_LIFETIME_MS);
  await db.insert(sessions).values({ userId, token, expiresAt });
  return token;
}

export async function findValidSession(token: string): Promise<{ userId: string } | null> {
  const rows = await db.select().from(sessions)
    .where(and(eq(sessions.token, token), gt(sessions.expiresAt, new Date())))
    .limit(1);
  return rows[0] ? { userId: rows[0].userId } : null;
}

export async function revokeSession(token: string): Promise<void> {
  await db.delete(sessions).where(eq(sessions.token, token));
}

export function safeEqualHex(a: string, b: string): boolean {
  try {
    return timingSafeEqual(Buffer.from(a, 'hex'), Buffer.from(b, 'hex'));
  } catch {
    return false;
  }
}
```

`timingSafeEqual` prevents timing attacks on token comparison. Senior habit.

---

## Lesson 44.4 — Login action

```ts
// src/routes/(auth)/login/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { users } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { verifyPassword } from '$lib/passwords';
import { createSession } from '$lib/sessions';

export const actions: Actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData();
    const email = String(data.get('email') ?? '').trim().toLowerCase();
    const password = String(data.get('password') ?? '');

    // generic error to avoid account enumeration
    const rows = await db.select().from(users).where(eq(users.email, email)).limit(1);
    const user = rows[0];

    // always run verification to avoid timing leaks; against a dummy hash if user missing
    const dummyHash = '$argon2id$v=19$m=65536,t=3,p=4$dummy';
    const ok = await verifyPassword(password, user?.passwordHash ?? dummyHash);

    if (!user || !ok) {
      return fail(400, { error: 'Invalid email or password.' });
    }

    const token = await createSession(user.id);
    cookies.set('session', token, {
      httpOnly: true,
      secure: true,
      sameSite: 'strict',
      path: '/',
      maxAge: 60 * 60 * 24 * 30,
    });

    redirect(303, '/dashboard');
  },
};
```

The `dummyHash` line: by always running `verifyPassword`, we avoid a timing leak that would reveal whether the email exists. Senior habit; cited in the OWASP authentication cheatsheet.

---

## Lesson 44.5 — The login form

```svelte
<!-- src/routes/(auth)/login/+page.svelte -->
<script lang="ts">
  import type { PageProps } from './$types';
  let { form }: PageProps = $props();
</script>

<form method="POST">
  <label>Email <input name="email" type="email" autocomplete="email" required /></label>
  <label>Password <input name="password" type="password" autocomplete="current-password" required /></label>
  <button type="submit">Sign in</button>
  {#if form?.error}<p class="error">{form.error}</p>{/if}
</form>
```

`autocomplete` tokens — senior habit. Browsers fill correctly.

---

## Lesson 44.6 — The threat model, named

| Attack | Mitigation in Streak |
|---|---|
| Session fixation | Rotate token on login (we do — fresh `randomBytes`). On password change/elevation, rotate again. |
| Timing attack on email | Always verify against a dummy hash if user missing. |
| Account enumeration | Generic "invalid email or password." Same response for both. |
| Brute force | Rate-limit `/login` per IP and per email (Ch 48). |
| Credential stuffing | Rate-limit + (later) hCaptcha after N failures. |
| CSRF | SvelteKit's `csrf.checkOrigin` (default on); `sameSite: strict` cookies. |
| XSS → token theft | `httpOnly` cookies. Tokens never readable by JS. |
| Replay | Short session lifetimes; "fresh" flag for sensitive actions. |

---

## Lesson 44.7 — Recurring concepts from earlier chapters

- **`+page.server.ts` actions** (Ch 41) — login is just another named (or default) action.
- **Atomic INSERT** (Ch 42) — `createSession` is single-row insert; safe.
- **`drizzle-orm`** (Ch 39) — `.where(eq(users.email, email))`.
- **`fail(400, ...)`** (Ch 41) — generic error response; never enumerate.

---

## Lesson 44.8 — What you can now read in the wild

After Chapter 44 you can:

- Read the **session schema** (token, expiresAt, fresh).
- Read **secure cookie flags** (`httpOnly`, `secure`, `sameSite`, `path`, `maxAge`) and explain each.
- Read **`crypto.timingSafeEqual`** as the constant-time-comparison primitive.
- Spot **account-enumeration** as a vulnerability and identify the `dummyHash`+generic-error mitigation.
- Recite the threat model out loud.

---

## End-of-chapter checkpoint

- [ ] Login form sets a session cookie.
- [ ] You can articulate the threat model out loud.

---

# Chapter 45 — `+hooks.server.ts`, `event.locals`, auth gating

> *Today's job:* every request is checked for a valid session; `event.locals.user` is populated. Visible win: `/app/*` requires login; `/` doesn't.

---

## Lesson 45.1 — `handle`

```ts
// src/hooks.server.ts
import type { Handle } from '@sveltejs/kit';
import { sequence } from '@sveltejs/kit/hooks';
import { db } from '$lib/db/client';
import { users, sessions } from '$lib/db/schema';
import { and, eq, gt } from 'drizzle-orm';

const authHandle: Handle = async ({ event, resolve }) => {
  const token = event.cookies.get('session');
  if (token) {
    const rows = await db.select({ user: users })
      .from(sessions)
      .innerJoin(users, eq(sessions.userId, users.id))
      .where(and(eq(sessions.token, token), gt(sessions.expiresAt, new Date())))
      .limit(1);
    if (rows[0]) event.locals.user = rows[0].user;
  }
  return resolve(event);
};

export const handle = sequence(authHandle);
```

`event.locals.user` is now available everywhere — `load`, actions, `+server.ts`.

---

## Lesson 45.2 — `App.Locals`

```ts
// src/app.d.ts
declare global {
  namespace App {
    interface Locals {
      user?: { id: string; email: string; role: 'user' | 'admin' };
    }
    interface Error { message: string; code?: string; }
  }
}
export {};
```

---

## Lesson 45.3 — `requireUser`

```ts
// src/lib/auth.ts
import { redirect } from '@sveltejs/kit';
import type { RequestEvent } from '@sveltejs/kit';

export function requireUser(event: RequestEvent): NonNullable<App.Locals['user']> {
  if (!event.locals.user) {
    redirect(303, `/login?next=${encodeURIComponent(event.url.pathname)}`);
  }
  return event.locals.user;
}

export function requireRole(event: RequestEvent, role: 'admin'): NonNullable<App.Locals['user']> {
  const user = requireUser(event);
  if (user.role !== role) {
    redirect(303, '/');
  }
  return user;
}
```

Use it in `(app)/+layout.server.ts`:

```ts
// src/routes/(app)/+layout.server.ts
import type { LayoutServerLoad } from './$types';
import { requireUser } from '$lib/auth';

export const load: LayoutServerLoad = async (event) => {
  const user = requireUser(event);
  return { user: { id: user.id, email: user.email } };
};
```

Now every `(app)/*` page is gated centrally. Forget once; gate forever.

---

## Lesson 45.4 — Logout

```ts
// src/routes/(auth)/logout/+server.ts
import type { RequestHandler } from './$types';
import { redirect } from '@sveltejs/kit';
import { revokeSession } from '$lib/sessions';

export const POST: RequestHandler = async ({ cookies }) => {
  const token = cookies.get('session');
  if (token) await revokeSession(token);
  cookies.delete('session', { path: '/' });
  redirect(303, '/');
};
```

In the layout:

```svelte
<form method="POST" action="/logout">
  <button type="submit">Log out</button>
</form>
```

---

## Lesson 45.5 — Recurring concepts from earlier chapters

- **`+hooks.server.ts`** — runs on every request before any `load`/`action`.
- **`+layout.server.ts`** (Ch 32) — central gate via `requireUser`.
- **`redirect()` returns `never`** — narrows the type after the guard.

---

## Lesson 45.6 — What you can now read in the wild

After Chapter 45 you can:

- Read **`handle({ event, resolve })`** and explain the request lifecycle.
- Read **`event.locals`** as the per-request context bag.
- Read **`sequence(handleA, handleB, ...)`** for chaining hooks.
- Read **`requireUser` / `requireRole`** as gating helpers.
- Spot pages that should be in `(app)/` but aren't, as auth gaps.

---

## End-of-chapter checkpoint

- [ ] Visiting `/app/*` while logged out redirects to login with `?next=...`.
- [ ] Logging in returns you there.
- [ ] Log out works and revokes the session row.

---

# Chapter 46 — Sign up, password hashing, the worker thread

> *Today's job:* sign up; password hashed off the request thread. Visible win: a new user can register and log in. `pnpm test:hash` confirms hash validation.

---

## Lesson 46.1 — Argon2id

Why Argon2id (over bcrypt or sha256):
- Memory-hard — resists GPU/ASIC attacks.
- OWASP recommendation as of 2026.
- Tunable parameters (`memoryCost`, `timeCost`, `parallelism`).

```bash
pnpm add @node-rs/argon2
```

---

## Lesson 46.2 — Hashing on a worker thread

A hash takes ~100–300 ms — that blocks the Node event loop on the request thread. Move it to a worker pool.

```ts
// src/lib/passwords/worker.ts
import { parentPort } from 'node:worker_threads';
import { hash, verify } from '@node-rs/argon2';

const params = { memoryCost: 19456, timeCost: 2, parallelism: 1 };

parentPort!.on('message', async (msg: { id: string; op: 'hash' | 'verify'; data: string | { plain: string; hash: string } }) => {
  try {
    let result: string | boolean;
    if (msg.op === 'hash') {
      result = await hash(msg.data as string, params);
    } else {
      const { plain, hash: existing } = msg.data as { plain: string; hash: string };
      result = await verify(existing, plain, params);
    }
    parentPort!.postMessage({ id: msg.id, ok: true, result });
  } catch (err) {
    parentPort!.postMessage({ id: msg.id, ok: false, error: String(err) });
  }
});
```

```ts
// src/lib/passwords/index.ts
import { Worker } from 'node:worker_threads';
import { fileURLToPath } from 'node:url';
import os from 'node:os';

type WorkerMessage = { id: string; ok: true; result: string | boolean } | { id: string; ok: false; error: string };
type DispatchOp = 'hash' | 'verify';
type DispatchData = string | { plain: string; hash: string };

interface PendingCall {
  id: string;
  resolve: (v: string | boolean) => void;
  reject: (e: Error) => void;
}

const POOL_SIZE = Math.max(1, os.cpus().length - 1);

const pool: Worker[] = [];
const queue: PendingCall[] = [];

function ensurePool(): void {
  if (pool.length > 0) return;
  for (let i = 0; i < POOL_SIZE; i += 1) {
    const w = new Worker(fileURLToPath(import.meta.resolve('./worker.ts')));
    w.on('message', (msg: WorkerMessage) => {
      const idx = queue.findIndex((q) => q.id === msg.id);
      if (idx === -1) return;
      const pending = queue[idx];
      if (pending === undefined) return;
      queue.splice(idx, 1);
      if (msg.ok) pending.resolve(msg.result);
      else pending.reject(new Error(msg.error));
    });
    pool.push(w);
  }
}

let nextWorker = 0;

function dispatch<T extends string | boolean>(op: DispatchOp, data: DispatchData): Promise<T> {
  ensurePool();
  return new Promise<T>((resolve, reject) => {
    const id = crypto.randomUUID();
    queue.push({
      id,
      resolve: (v) => resolve(v as T),
      reject,
    });
    const worker = pool[nextWorker];
    if (worker === undefined) {
      reject(new Error('worker pool empty'));
      return;
    }
    worker.postMessage({ id, op, data });
    nextWorker = (nextWorker + 1) % pool.length;
  });
}

export function hashPassword(plain: string): Promise<string> {
  return dispatch<string>('hash', plain);
}

export function verifyPassword(plain: string, hash: string): Promise<boolean> {
  return dispatch<boolean>('verify', { plain, hash });
}
```

The actual production version would handle worker crashes, queue timeouts (the Bible's `connect_timeout` rule applied: queued requests should time out if no worker responds in N seconds), graceful shutdown, and a max queue depth so a worker-pool stampede can't OOM the process. We're showing the core mechanism; the production-hardening version goes in `src/lib/passwords/index.ts` once the security review (Ch 48) flags it.

---

## Lesson 46.3 — Sign-up action

```ts
// src/routes/(auth)/signup/+page.server.ts
import type { Actions } from './$types';
import { fail } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { users, emailVerifications } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { hashPassword } from '$lib/passwords';
import { randomBytes } from 'node:crypto';

const TOKEN_LIFETIME_MS = 24 * 60 * 60 * 1000;

export const actions: Actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = String(data.get('email') ?? '').trim().toLowerCase();
    const password = String(data.get('password') ?? '');

    if (!/^[^@]+@[^@]+\.[^@]+$/.test(email)) return fail(400, { error: 'Invalid email.' });
    if (password.length < 12) return fail(400, { error: 'Password must be at least 12 characters.' });

    // generic response — same whether email exists or not
    const existing = await db.select().from(users).where(eq(users.email, email)).limit(1);
    if (existing.length > 0) {
      // pretend we sent the email; same UX
      return { success: true };
    }

    const passwordHash = await hashPassword(password);
    const [created] = await db.insert(users).values({ email, passwordHash }).returning();

    const token = randomBytes(32).toString('hex');
    await db.insert(emailVerifications).values({
      userId: created.id,
      token,
      expiresAt: new Date(Date.now() + TOKEN_LIFETIME_MS),
    });

    // log the link to dev console; real email in Ch 64
    console.log(`Verification link: http://localhost:5173/verify-email/${token}`);

    return { success: true };
  },
};
```

---

## Lesson 46.4 — `pnpm test:hash`

```ts
// tests/unit/hash.test.ts
import { describe, it, expect } from 'vitest';
import { hashPassword, verifyPassword } from '$lib/passwords';

describe('passwords', () => {
  it('hashes and verifies', async () => {
    const h = await hashPassword('correct horse battery staple');
    expect(await verifyPassword('correct horse battery staple', h)).toBe(true);
    expect(await verifyPassword('wrong', h)).toBe(false);
  });
});
```

`pnpm test:hash` would run `vitest run tests/unit/hash.test.ts`. Add it to `package.json`'s scripts.

---

## Lesson 46.5 — Recurring concepts from earlier chapters

- **Worker threads** as the JS analogue of Rust's `spawn_blocking`.
- **Boundary parser idiom** (Ch 26) — email/password validation at the action edge.
- **`returning()`** (Ch 39) — `db.insert(users).values(...).returning()` for the new ID.

---

## Lesson 46.6 — What you can now read in the wild

After Chapter 46 you can:

- Read **Argon2id config** (`memoryCost`, `timeCost`, `parallelism`) and explain trade-offs.
- Read a **worker-pool** implementation and trace a request from `hashPassword` to a worker thread and back.
- Recognise **password length over complexity** (NIST 800-63B, no max, no character classes) as the modern recommendation.
- Spot a **password hash on the request thread** as a tail-latency disaster.

---

## End-of-chapter checkpoint

- [ ] Signup creates a user, logs the verify link.
- [ ] Login works after verifying.
- [ ] `pnpm test:hash` is green.

---

# Chapter 47 — RBAC, audit log, the admin route

> *Today's job:* `/admin/users` for admins only. Every admin action lands an audit row. Visible win: non-admin can't reach it; admin sees the dashboard; audit log fills.

---

## Lesson 47.1 — `auditLog` table

```ts
export const auditLog = pgTable('audit_log', {
  id: uuid('id').primaryKey().defaultRandom(),
  actorId: uuid('actor_id').references(() => users.id, { onDelete: 'set null' }),
  action: text('action').notNull(), // 'habit.add', 'user.impersonate', etc.
  targetType: text('target_type'),
  targetId: text('target_id'),
  payloadDigest: text('payload_digest'), // JSON summary, no PII
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});
```

---

## Lesson 47.2 — `withAudit`

The transaction type is the parameter type of `db.transaction`'s callback:

```ts
// src/lib/auth.ts
import { db } from '$lib/db/client';
import { auditLog } from '$lib/db/schema';
import type { RequestEvent } from '@sveltejs/kit';

type Tx = Parameters<Parameters<typeof db.transaction>[0]>[0];

export async function withAudit<T>(
  event: RequestEvent,
  action: string,
  targetType: string | null,
  targetId: string | null,
  fn: (tx: Tx) => Promise<T>,
): Promise<T> {
  return db.transaction(async (tx) => {
    const result = await fn(tx);
    await tx.insert(auditLog).values({
      actorId: event.locals.user?.id ?? null,
      action,
      targetType,
      targetId,
      payloadDigest: null,
    });
    return result;
  });
}
```

Used as:

```ts
addHabit: async (event) => {
  // ... parse ...
  await withAudit(event, 'habit.add', 'habit', null, async (tx) => {
    await tx.insert(habits).values(/* ... */);
  });
},
```

The audit row is *inside* the transaction — atomic. Bible-compliant.

---

## Lesson 47.3 — Centralised admin gate

```ts
// src/routes/(app)/admin/+layout.server.ts
import type { LayoutServerLoad } from './$types';
import { requireRole } from '$lib/auth';

export const load: LayoutServerLoad = async (event) => {
  const user = requireRole(event, 'admin');
  return { user };
};
```

Forget once, gate forever. Every page under `(app)/admin/` is protected.

---

## Lesson 47.4 — Impersonation safely

Schema additions:

```ts
// in schema.ts: extend sessions
export const sessions = pgTable('sessions', {
  // ... existing fields ...
  impersonatorId: uuid('impersonator_id').references(() => users.id, { onDelete: 'set null' }),
});
```

Action:

```ts
// src/routes/(app)/admin/users/[id]/impersonate/+page.server.ts
import type { Actions } from './$types';
import { redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { sessions, users } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { requireRole, withAudit } from '$lib/auth';
import { newSessionToken } from '$lib/sessions';

const IMPERSONATION_LIFETIME_MS = 15 * 60 * 1000; // 15 minutes — short-lived

export const actions: Actions = {
  default: async (event) => {
    const admin = requireRole(event, 'admin');
    const targetId = event.params.id;
    const [target] = await db.select().from(users).where(eq(users.id, targetId)).limit(1);
    if (target === undefined) redirect(303, '/admin/users');

    const token = newSessionToken();
    await withAudit(event, 'user.impersonate.start', 'user', target.id, async (tx) => {
      await tx.insert(sessions).values({
        userId: target.id,
        impersonatorId: admin.id,
        token,
        expiresAt: new Date(Date.now() + IMPERSONATION_LIFETIME_MS),
        fresh: false,
      });
    });

    event.cookies.set('session', token, {
      httpOnly: true, secure: true, sameSite: 'strict', path: '/',
      maxAge: Math.floor(IMPERSONATION_LIFETIME_MS / 1000),
    });
    redirect(303, '/dashboard');
  },
};
```

In the layout, show a banner when impersonating:

```svelte
{#if data.user.impersonatorId}
  <div class="impersonation-banner">
    Impersonating {data.user.email}.
    <form method="POST" action="/admin/end-impersonation">
      <button type="submit">End impersonation</button>
    </form>
  </div>
{/if}
```

The end-impersonation action deletes only this session (not the admin's original session) and clears the cookie.

The principle is *fresh short-lived session + visible banner + audit row at start and end*. Without all three, impersonation becomes a backdoor.

---

## Lesson 47.5 — Recurring concepts from earlier chapters

- **`db.transaction`** (Ch 42) — `withAudit` enforces atomicity.
- **`Result` typing** (Ch 27) — every method that can fail uses it (still applies inside `withAudit`'s `fn`).
- **`+layout.server.ts` central gate** (Ch 32, 45) — `requireRole` lives at the layout level.

---

## Lesson 47.6 — What you can now read in the wild

After Chapter 47 you can:

- Read an **`audit_log`** schema and trace what gets written when.
- Read **`withAudit(event, action, target, fn)`** as the single-call atomicity helper.
- Read a **fresh-session impersonation** flow and identify the three pieces (session, banner, audit).
- Spot a **role-check inside a handler** (instead of in the layout) as a refactor target.

---

## End-of-chapter checkpoint

- [ ] `(app)/admin/*` is admin-only.
- [ ] `audit_log` fills on every action that uses `withAudit`.
- [ ] Impersonation flow logs at start and end.

---

# Chapter 48 — Security review — the principal-engineer checklist

> *Today's job:* a structured security review of your own auth system. At least three findings, fixed. Visible win: `docs/security-review-2026-XX-XX.md` lives in the repo.

---

## Lesson 48.1 — STRIDE for web auth

Walk every category, citing how Streak addresses it:

| STRIDE | Mitigation |
|---|---|
| **S**poofing | Sessions + `httpOnly` cookies; rotate on login |
| **T**ampering | CSRF (`csrf.checkOrigin`, `sameSite: strict`); signed-cookie option |
| **R**epudiation | `audit_log` atomic to actions |
| **I**nformation disclosure | `redactError` (Ch 56); generic auth responses; no PII logging |
| **D**oS | Rate-limit `/login`, `/signup`; pool sizing; query timeouts |
| **E**levation | RBAC central in `(admin)/+layout.server.ts`; `requireRole` |

---

## Lesson 48.2 — OWASP Top 10 walkthrough

Run grep audits:

```bash
# any console.log of sensitive things?
grep -rE "console\.log.*(password|token|secret|email|cookie)" src/

# any disabled CSRF?
grep -r "checkOrigin: false" src/

# any plain-text password storage?
grep -r "password.*=.*[^h]ash" src/

# any unconfigured timeouts?
grep -rE "(fetch|postgres|stripe|resend)\(" src/ | grep -v "timeout"
```

---

## Lesson 48.3 — Rate-limiting

```ts
// src/lib/rate-limit.ts
type Bucket = { tokens: number; lastRefill: number };
const buckets = new Map<string, Bucket>();
const RATE = 5; // tokens
const WINDOW_MS = 60_000;

export function consume(key: string, tokensNeeded: number = 1): boolean {
  const now = Date.now();
  const b = buckets.get(key) ?? { tokens: RATE, lastRefill: now };
  const elapsed = now - b.lastRefill;
  if (elapsed > WINDOW_MS) {
    b.tokens = RATE;
    b.lastRefill = now;
  }
  if (b.tokens < tokensNeeded) {
    buckets.set(key, b);
    return false;
  }
  b.tokens -= tokensNeeded;
  buckets.set(key, b);
  return true;
}
```

In login action:

```ts
import { consume } from '$lib/rate-limit';

default: async ({ request, getClientAddress }) => {
  const ip = getClientAddress();
  const data = await request.formData();
  const email = String(data.get('email') ?? '').toLowerCase();
  if (!consume(`login:ip:${ip}`) || !consume(`login:email:${email}`)) {
    return fail(429, { error: 'Too many attempts. Try again in a minute.' });
  }
  // ... rest ...
},
```

In production, replace the in-memory map with Redis. Named.

---

## Lesson 48.4 — CSP and the security headers

In `+hooks.server.ts`'s `handle`:

```ts
const response = await resolve(event);
response.headers.set('Strict-Transport-Security', 'max-age=63072000; includeSubDomains; preload');
response.headers.set('X-Content-Type-Options', 'nosniff');
response.headers.set('X-Frame-Options', 'DENY');
response.headers.set('Referrer-Policy', 'strict-origin-when-cross-origin');
response.headers.set('Permissions-Policy', 'accelerometer=(), camera=(), geolocation=(), microphone=()');
response.headers.set('Content-Security-Policy', "default-src 'self'; script-src 'self'; style-src 'self' 'unsafe-inline'; img-src 'self' data: blob:; font-src 'self' data:; connect-src 'self'; frame-ancestors 'none'");
return response;
```

Every value justified in the security-review doc. CSP starts strict; loosen only with reason.

---

## Lesson 48.5 — `pnpm security:headers`

A small Vitest test against the running preview server. Run `pnpm preview` in one terminal, then run the test in another (or use Playwright's `webServer` config — Ch 60).

```ts
// tests/security/headers.test.ts
import { describe, it, expect } from 'vitest';

const BASE = process.env.TEST_BASE_URL ?? 'http://localhost:4173';

describe('security headers', () => {
  it('sets HSTS, CSP, etc.', async () => {
    const r = await fetch(`${BASE}/`);
    expect(r.headers.get('strict-transport-security')).toContain('max-age=');
    expect(r.headers.get('content-security-policy')).toContain("default-src 'self'");
    expect(r.headers.get('x-content-type-options')).toBe('nosniff');
    expect(r.headers.get('x-frame-options')).toBe('DENY');
    expect(r.headers.get('referrer-policy')).toBeTruthy();
    expect(r.headers.get('permissions-policy')).toBeTruthy();
  });
});
```

Add to `package.json`:

```json
{
  "scripts": {
    "security:headers": "vitest run tests/security/headers.test.ts"
  }
}
```

In CI, this runs against the deployed preview URL via `TEST_BASE_URL`.

---

## Lesson 48.6 — The review document

`docs/security-review-2026-05-05.md`:

```markdown
# Streak — Security Review (2026-05-05)

## Scope
Auth system: signup, login, logout, sessions, RBAC, audit log.

## Findings

### 1. Session token comparison (FIXED)
**Severity:** Medium.
**Issue:** Session token compared with `===`; vulnerable to timing.
**Fix:** Use `crypto.timingSafeEqual`. Ref: `src/lib/sessions.ts:42`.

### 2. CSP missing on /login (FIXED)
...

### 3. Rate-limit per-IP only (FIXED)
**Issue:** Original implementation rate-limited by IP; an attacker behind a CDN could bypass.
**Fix:** Per-IP and per-email both buckets must allow.
```

---

## Lesson 48.7 — Recurring concepts from earlier chapters

Part VII's spine, in one place:

- **Sessions, secure cookies, threat model** (Ch 44).
- **`+hooks.server.ts`, `event.locals`, `requireUser`** (Ch 45).
- **Argon2id on a worker thread** (Ch 46).
- **RBAC + `withAudit` + impersonation** (Ch 47).
- **Rate-limiting + CSP/HSTS + security review document** (Ch 48).

Together, that's a real auth system the reader could defend in a code review.

---

## Lesson 48.8 — What you can now read in the wild

After Part VII you can:

- Articulate STRIDE for a web auth system out loud, with the Streak-specific mitigation for each category.
- Write an OWASP-Top-10 self-audit grep script for your own codebase.
- Read **CSP**, **HSTS preload**, **X-Frame-Options**, **Referrer-Policy**, **Permissions-Policy** and explain what each defends against.
- Recognise an in-memory rate-limiter as a single-instance limit and know to swap for Redis at scale.
- Write a **security review document** with named findings and fixes.

---

## End-of-chapter checkpoint

- [ ] Rate-limiting is live on `/login` and `/signup`.
- [ ] Security headers are set in `handle`.
- [ ] `pnpm security:headers` passes.
- [ ] You wrote the review document.

End of Part VII. Next: money, polish, every Svelte primitive.
