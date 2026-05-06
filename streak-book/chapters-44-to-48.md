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
  tokenHash: text('token_hash').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  fresh: boolean('fresh').notNull().default(true),
  createdAt: timestamp('created_at', { withTimezone: true }).notNull().defaultNow(),
});

export const emailVerifications = pgTable('email_verifications', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  tokenHash: text('token_hash').notNull().unique(),
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
import { createHash, randomBytes, timingSafeEqual } from 'node:crypto';

const SESSION_LIFETIME_MS = 30 * 24 * 60 * 60 * 1000;

export function newSessionToken(): string {
  return randomBytes(32).toString('base64url');
}

function sha256Hex(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export async function createSession(userId: string): Promise<string> {
  const token = newSessionToken();
  const tokenHash = sha256Hex(token);
  const expiresAt = new Date(Date.now() + SESSION_LIFETIME_MS);
  await db.insert(sessions).values({ userId, tokenHash, expiresAt });
  return token;
}

export async function findValidSession(token: string): Promise<{ userId: string } | null> {
  const tokenHash = sha256Hex(token);
  const rows = await db.select().from(sessions)
    .where(and(eq(sessions.tokenHash, tokenHash), gt(sessions.expiresAt, new Date())))
    .limit(1);
  const row = rows[0];
  if (row === undefined) {
    return null;
  }
  // Defence in depth: verify the row's stored hash against the cookie token's
  // hash in constant time. Postgres equality on a unique-indexed column is
  // already constant-time for this scenario (the index probe is content-
  // independent, and the stored value is a fixed-width digest), so the extra
  // check is belt-and-braces — but cheap, and explicit.
  if (!safeEqualHex(row.tokenHash, tokenHash)) {
    return null;
  }
  return { userId: row.userId };
}

export async function revokeSession(token: string): Promise<void> {
  const tokenHash = sha256Hex(token);
  await db.delete(sessions).where(eq(sessions.tokenHash, tokenHash));
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

> **Sliding sessions, deferred.** `findValidSession` doesn't refresh `expiresAt`. A senior pattern is sliding sessions: if the session is past 50% of its TTL, extend `expiresAt` to now + 30 days. We don't ship the slide here but flag it as a known limitation; users will be logged out 30 days after first login regardless of activity. Add when the support load justifies it.

---

## Lesson 44.4 — Login action

```ts
// src/routes/(auth)/login/+page.server.ts
import type { Actions } from './$types';
import { fail, redirect } from '@sveltejs/kit';
import { dev } from '$app/environment';
import { db } from '$lib/db/client';
import { users } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { verifyPassword } from '$lib/passwords';
import { createSession } from '$lib/sessions';

export const actions: Actions = {
  default: async ({ request, cookies }) => {
    const data = await request.formData();
    const rawEmail = data.get('email');
    const rawPassword = data.get('password');
    const email = (typeof rawEmail === 'string' ? rawEmail : '').trim().toLowerCase();
    const password = typeof rawPassword === 'string' ? rawPassword : '';

    // generic error to avoid account enumeration
    const rows = await db.select().from(users).where(eq(users.email, email)).limit(1);
    const user = rows[0];

    // always run verification to avoid timing leaks; against a dummy hash if user missing
    const dummyHash = '$argon2id$v=19$m=65536,t=3,p=4$dummy';
    const ok = await verifyPassword(password, user?.passwordHash ?? dummyHash);

    if (user === undefined || !ok) {
      return fail(400, { error: 'Invalid email or password.' });
    }

    const token = await createSession(user.id);
    cookies.set('session', token, {
      httpOnly: true,
      secure: !dev,
      sameSite: 'lax',
      path: '/',
      maxAge: 60 * 60 * 24 * 30,
    });

    throw redirect(303, '/dashboard');
  },
};
```

The `dummyHash` line: by always running `verifyPassword`, we avoid a timing leak that would reveal whether the email exists. Senior habit; cited in the OWASP authentication cheatsheet.

> **`secure: !dev` matters.** `secure: true` requires HTTPS; the browser refuses to set a secure cookie on plain `http://localhost`. SvelteKit's `cookies.set` does NOT auto-downgrade — you'll silently lose your dev session. Gate with `dev` from `$app/environment` so production gets `secure: true` and local gets `secure: false`.

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
| CSRF | SvelteKit's `csrf.checkOrigin` (default on); `sameSite: 'lax'` cookies. |
| XSS → token theft | `httpOnly` cookies. Tokens never readable by JS. |
| Replay | Short session lifetimes; "fresh" flag for sensitive actions. |

> **Why `sameSite: 'lax'` instead of `'strict'`?** Strict mode strips the cookie on *any* cross-site navigation — including a user clicking a verification link from their email or a password-reset link. The session would not travel; the user would land logged out. Lax sends the cookie on top-level GET navigations from external origins (links, address bar) but blocks it on cross-site POSTs, which is the actual CSRF threat. Lax is the senior default for session cookies; reach for strict only on cookies guarding *additional* high-value actions (e.g. a separate "reauth" cookie that gates `/admin/*`).
>
> Modern Chromium "lax-but-allowed" defaults change the cross-site behaviour for top-level navigations — a user clicking a verification link from email may keep the cookie under some conditions. The exact behaviour drifts between Chromium releases; pin to OWASP's current guidance or empirically test the verification flow during launch.

---

## Lesson 44.7 — Forgot password / password reset

The single largest auth flow we haven't built yet. Three steps:

1. User submits their email at `/forgot-password`.
2. Server *unconditionally* responds with "if that email is registered, we sent a link" (account-enumeration mitigation), then *conditionally* mints a single-use token and emails the user.
3. User clicks the link `/reset-password/[token]`, sets a new password, all their existing sessions are revoked.

Schema additions:

```ts
// schema.ts additions
export const passwordResets = pgTable('password_resets', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  hashedToken: text('hashed_token').notNull().unique(),
  expiresAt: timestamp('expires_at', { withTimezone: true }).notNull(),
  consumedAt: timestamp('consumed_at', { withTimezone: true }),
});
```

Note: we store the **hash** of the token, not the token itself. If the DB leaks, the leaked rows can't be used to reset accounts. The token is shown to the user once, embedded in the email link.

> **One encoding, applied everywhere.** 256 bits of entropy in either base64url or hex; we pick base64url for shorter URLs. Every random token in this system — `passwordResets.hashedToken`'s pre-image, `emailVerifications`'s pre-image, `newSessionToken`, the impersonation token in Ch 47.4 — is `randomBytes(32).toString('base64url')`. Mixing encodings is the kind of inconsistency that hides bugs (a comparator written for hex silently fails on base64url, etc.). Pick one; apply it.

Reset-request action:

```ts
// src/routes/(auth)/forgot-password/+page.server.ts
import type { Actions } from './$types';
import { db } from '$lib/db/client';
import { users, passwordResets } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { randomBytes, createHash } from 'node:crypto';
import { logger } from '$lib/logger';

const RESET_LIFETIME_MS = 15 * 60 * 1000; // 15 minutes — short, reset is high-value

function sha256(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export const actions: Actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const email = String(data.get('email') ?? '').trim().toLowerCase();

    // Always respond identically — prevents account enumeration.
    // The work happens or doesn't, but the user can't tell.
    const [user] = await db.select({ id: users.id, email: users.email })
      .from(users)
      .where(eq(users.email, email))
      .limit(1);

    if (user !== undefined) {
      const rawToken = randomBytes(32).toString('base64url');
      const hashedToken = sha256(rawToken);
      const expiresAt = new Date(Date.now() + RESET_LIFETIME_MS);

      await db.insert(passwordResets).values({
        userId: user.id,
        hashedToken,
        expiresAt,
      });

      // In dev we log to console; Ch 64 wires real email via Resend.
      logger.info({ email: user.email }, 'password.reset.requested');
      console.log(`Reset link: http://localhost:5173/reset-password/${rawToken}`);
    }

    return { success: true };
  },
};
```

Reset-consume action — atomic, single-use, revokes all sessions:

```ts
// src/routes/(auth)/reset-password/[token]/+page.server.ts
import type { Actions, PageServerLoad } from './$types';
import { error, fail, redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { passwordResets, sessions, users } from '$lib/db/schema';
import { and, eq, gt, isNull } from 'drizzle-orm';
import { hashPassword } from '$lib/passwords';
import { createHash } from 'node:crypto';

function sha256(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export const load: PageServerLoad = async ({ params }) => {
  const hashedToken = sha256(params.token);
  const [reset] = await db.select()
    .from(passwordResets)
    .where(and(
      eq(passwordResets.hashedToken, hashedToken),
      gt(passwordResets.expiresAt, new Date()),
      isNull(passwordResets.consumedAt),
    ))
    .limit(1);

  if (reset === undefined) {
    throw error(400, 'Reset link is invalid or has expired.');
  }
  return { tokenValid: true };
};

export const actions: Actions = {
  default: async ({ request, params, cookies }) => {
    const data = await request.formData();
    const rawPassword = data.get('password');
    const password = typeof rawPassword === 'string' ? rawPassword : '';
    if (password.length < 12) {
      // Short passwords from a user form are EXPECTED failures; use `fail`,
      // not `error`. `error` is reserved for unexpected/bug-shaped failures.
      return fail(400, { message: 'Password must be at least 12 characters.' });
    }

    const hashedToken = sha256(params.token);
    const passwordHash = await hashPassword(password);

    // Atomic consume — Bible rule #11.
    // UPDATE returns 0 rows if the token was already consumed by a parallel request.
    const consumed = await db.update(passwordResets)
      .set({ consumedAt: new Date() })
      .where(and(
        eq(passwordResets.hashedToken, hashedToken),
        gt(passwordResets.expiresAt, new Date()),
        isNull(passwordResets.consumedAt),
      ))
      .returning({ userId: passwordResets.userId });

    const [first] = consumed;
    if (first === undefined) {
      throw error(400, 'Reset link is invalid or has expired.');
    }
    const userId = first.userId;

    // Update the password and revoke all the user's existing sessions atomically.
    await db.transaction(async (tx) => {
      await tx.update(users).set({ passwordHash }).where(eq(users.id, userId));
      await tx.delete(sessions).where(eq(sessions.userId, userId));
    });

    cookies.delete('session', { path: '/' });
    throw redirect(303, '/login?reset=ok');
  },
};
```

Three senior touches:

1. **The reset action uses an atomic UPDATE-RETURNING** to consume the token. Two parallel clicks of the same link → only one succeeds. Bible rule #11.
2. **All the user's sessions are revoked on reset.** If the password was reset because it leaked, every session minted with the old password is now untrusted; nuke them.
3. **Account-enumeration discipline holds.** The `forgot-password` action returns `{ success: true }` whether the email exists or not. Same response code, same response body. The attacker can't distinguish.

> **token hashing at rest** *(noun)* — store `sha256(token)`, never the token itself. The token is high-entropy, so SHA-256 is enough (no Argon2 needed — that's for human-typed passwords).

---

## Lesson 44.8 — Recurring concepts from earlier chapters

- **`+page.server.ts` actions** (Ch 41) — login + forgot-password + reset are all named actions.
- **Atomic INSERT and atomic UPDATE-RETURNING** (Ch 42) — sessions, reset consume.
- **`drizzle-orm`** (Ch 39) — every query goes through it.
- **`fail(400, ...)` / generic responses** (Ch 41) — never enumerate.

---

## Lesson 44.9 — What you can now read in the wild

After Chapter 44 you can:

- Read the **session schema** (token, expiresAt, fresh).
- Read **secure cookie flags** (`httpOnly`, `secure`, `sameSite: 'lax'`, `path`, `maxAge`) and explain each.
- Read **`crypto.timingSafeEqual`** as the constant-time-comparison primitive.
- Read a **password-reset flow** with hashed-at-rest tokens, atomic consume, session revocation.
- Spot **account-enumeration** as a vulnerability and identify the `dummyHash` + identical-response mitigation.
- Recite the threat model out loud.

---

## End-of-chapter checkpoint

- [ ] Login form sets a session cookie.
- [ ] Forgot-password generates a logged reset link.
- [ ] Reset-password consumes the token atomically and revokes existing sessions.
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
import { createHash } from 'node:crypto';

function sha256Hex(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

const authHandle: Handle = async ({ event, resolve }) => {
  const token = event.cookies.get('session');
  if (token !== undefined) {
    const tokenHash = sha256Hex(token);
    const rows = await db.select({ user: users })
      .from(sessions)
      .innerJoin(users, eq(sessions.userId, users.id))
      .where(and(eq(sessions.tokenHash, tokenHash), gt(sessions.expiresAt, new Date())))
      .limit(1);
    const row = rows[0];
    if (row !== undefined) {
      event.locals.user = row.user;
    }
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

// `Role` is the role-union pulled from `App.Locals['user']`. When we widen the
// role set in Ch 47 (e.g. add 'support', 'auditor'), this stays in sync
// without touching every call-site.
export type Role = NonNullable<App.Locals['user']>['role'];

export function requireUser(event: RequestEvent): NonNullable<App.Locals['user']> {
  const user = event.locals.user;
  if (user === undefined) {
    throw redirect(303, `/login?next=${encodeURIComponent(event.url.pathname)}`);
  }
  return user;
}

export function requireRole(event: RequestEvent, role: Role): NonNullable<App.Locals['user']> {
  const user = requireUser(event);
  if (user.role !== role) {
    throw redirect(303, '/');
  }
  return user;
}
```

Both helpers `throw redirect(...)`. The `throw` is load-bearing: SvelteKit's `redirect()` returns `never` only when thrown, which lets TypeScript narrow `event.locals.user` from `undefined | User` to `User` after the guard. A bare `redirect(...)` call would leave the type un-narrowed *and* fail to redirect — the worst of both worlds.

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
  throw redirect(303, '/');
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
import * as v from 'valibot';

if (parentPort === null) {
  throw new Error('worker.ts must be loaded in a worker thread');
}
const port = parentPort; // narrowed to MessagePort — no `!` past this point

const params = { memoryCost: 19456, timeCost: 2, parallelism: 1 };

// Boundary parser: the message arrives over an IPC channel and is therefore
// untrusted input. Validate with Valibot rather than trust a discriminated
// union and a hand-rolled type guard. Bible rule #5 (parse, don't cast).
const HashRequest = v.object({
  id: v.string(),
  op: v.literal('hash'),
  data: v.string(),
});
const VerifyRequest = v.object({
  id: v.string(),
  op: v.literal('verify'),
  data: v.object({ plain: v.string(), hash: v.string() }),
});
const IncomingMsg = v.union([HashRequest, VerifyRequest]);

port.on('message', async (raw: unknown) => {
  const parsed = v.safeParse(IncomingMsg, raw);
  if (!parsed.success) {
    // Don't crash the worker on malformed messages; drop silently.
    return;
  }
  const msg = parsed.output;
  try {
    let result: string | boolean;
    if (msg.op === 'hash') {
      result = await hash(msg.data, params);
    } else {
      result = await verify(msg.data.hash, msg.data.plain, params);
    }
    port.postMessage({ id: msg.id, ok: true, result });
  } catch (err) {
    port.postMessage({ id: msg.id, ok: false, error: String(err) });
  }
});
```

Three senior touches over the naive form:

1. **`if (parentPort === null) throw` then `const port = parentPort`** narrows for the rest of the file. No `!` needed (Bible rule #3) — every subsequent reference uses `port` directly.
2. **Valibot at the worker boundary** (`v.parse` / `v.safeParse`). No hand-rolled `is`-predicates, no `as Record<string, unknown>` casts. The schema is the source of truth for what the worker accepts.
3. **`raw: unknown`** is the *real* type of a worker message — Node's typings used to default it to `any`, which silently allows bugs. Treat the boundary as untrusted.

```ts
// src/lib/passwords/index.ts
import { Worker } from 'node:worker_threads';
import os from 'node:os';
// Vite's `?worker` import returns a constructor that produces a Worker
// running the bundled module. Works in dev (`pnpm dev`) and production builds.
// Without `?worker`, you'd be loading a `.ts` file at runtime — broken on Node.
//
// IMPORTANT: verify against your production adapter. Vite's `?worker` query
// bundles the worker into the build, and SvelteKit's adapter-vercel ships the
// bundled output — but you should confirm the worker chunk lands at the right
// path for your deploy target. If it doesn't, fall back to the plain
//   new Worker(new URL('./worker.ts', import.meta.url), { type: 'module' })
// form. Test the build (`pnpm build && pnpm preview`) to confirm before
// shipping.
import HashWorker from './worker.ts?worker';

type WorkerMessage =
  | { id: string; ok: true; result: string | boolean }
  | { id: string; ok: false; error: string };
type DispatchOp = 'hash' | 'verify';
type DispatchData = string | { plain: string; hash: string };

interface PendingCall {
  id: string;
  resolve: (v: string | boolean) => void;
  reject: (e: Error) => void;
  timer: NodeJS.Timeout;
}

const POOL_SIZE = Math.max(1, os.cpus().length - 1);
const QUEUE_TIMEOUT_MS = 5_000; // Bible rule #13 — queued requests time out

const pool: Worker[] = [];
const queue = new Map<string, PendingCall>();

function ensurePool(): void {
  if (pool.length > 0) return;
  for (let i = 0; i < POOL_SIZE; i += 1) {
    const w = new HashWorker();
    w.on('message', (msg: WorkerMessage) => {
      const pending = queue.get(msg.id);
      if (pending === undefined) return;
      clearTimeout(pending.timer);
      queue.delete(msg.id);
      if (msg.ok) pending.resolve(msg.result);
      else pending.reject(new Error(msg.error));
    });
    w.on('error', (err) => {
      // A worker crashed. Reject every in-flight call routed to it.
      // (We don't track per-worker routing here for brevity; in production,
      // you'd map jobs → workers and only reject affected ones.)
      // Snapshot the keys first — mutating a Map while iterating it is
      // allowed in V8 but skips entries depending on insertion order. The
      // snapshot makes the iteration order-independent.
      const ids = [...queue.keys()];
      for (const id of ids) {
        const pending = queue.get(id);
        if (pending !== undefined) {
          clearTimeout(pending.timer);
          pending.reject(err);
          queue.delete(id);
        }
      }
    });
    pool.push(w);
  }
}

let nextWorker = 0;

function dispatch<T extends string | boolean>(op: DispatchOp, data: DispatchData): Promise<T> {
  ensurePool();
  return new Promise<T>((resolve, reject) => {
    const id = crypto.randomUUID();
    const timer = setTimeout(() => {
      queue.delete(id);
      reject(new Error('hash worker timed out'));
    }, QUEUE_TIMEOUT_MS);

    queue.set(id, {
      id,
      resolve: (v) => resolve(v as T),
      reject,
      timer,
    });

    const worker = pool[nextWorker];
    if (worker === undefined) {
      clearTimeout(timer);
      queue.delete(id);
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

Three production-grade fixes from the first version:

1. **`import HashWorker from './worker.ts?worker'`** — Vite's `?worker` query produces a Worker constructor that loads the *bundled* module. Without `?worker`, you'd be telling Node "load this `.ts` file at runtime," which fails outside dev. The previous form used `import.meta.resolve` and would have shipped broken in production.
2. **`QUEUE_TIMEOUT_MS = 5_000`** with a per-call `setTimeout` — Bible rule #13 applied. A wedged worker can't hold a request thread forever.
3. **`worker.on('error', ...)`** rejects in-flight calls on worker crashes. Without this, a segfaulting Argon2 binding would leak promises forever.

Things still left for a real production version: per-worker job routing (so one crash only fails its own jobs), a max queue depth to refuse new work under stampede, graceful shutdown on `SIGTERM`. These all matter at scale; we'd add them when the security review (Ch 48) demands.

---

## Lesson 46.3 — Sign-up action

```ts
// src/routes/(auth)/signup/+page.server.ts
import type { Actions } from './$types';
import { error, fail } from '@sveltejs/kit';
import * as v from 'valibot';
import { db } from '$lib/db/client';
import { users, emailVerifications } from '$lib/db/schema';
import { hashPassword } from '$lib/passwords';
import { createHash, randomBytes } from 'node:crypto';

const TOKEN_LIFETIME_MS = 24 * 60 * 60 * 1000;

// Lightweight syntactic check. Real validation is the verification email
// roundtrip — anything else (regex, mx-record probe) is theatre.
const EmailSchema = v.pipe(v.string(), v.email());

function sha256Hex(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export const actions: Actions = {
  default: async ({ request }) => {
    const data = await request.formData();
    const rawEmail = data.get('email');
    const rawPassword = data.get('password');
    const emailInput = (typeof rawEmail === 'string' ? rawEmail : '').trim().toLowerCase();
    const password = typeof rawPassword === 'string' ? rawPassword : '';

    const parsed = v.safeParse(EmailSchema, emailInput);
    if (!parsed.success) {
      return fail(400, { error: 'Invalid email.' });
    }
    const email = parsed.output;
    if (password.length < 12) {
      return fail(400, { error: 'Password must be at least 12 characters.' });
    }

    const passwordHash = await hashPassword(password);

    // Atomic insert-or-do-nothing replaces the SELECT-then-INSERT pattern
    // (Bible rule #11). The unique constraint on `email` does the dedupe
    // atomically — zero rows back means the email was taken. We respond the
    // same way either path, preserving account-enumeration discipline.
    const inserted = await db.insert(users)
      .values({ email, passwordHash })
      .onConflictDoNothing({ target: users.email })
      .returning();
    const [created] = inserted;
    if (created === undefined) {
      // Email already taken; respond with the same shape as the success path.
      return { success: true };
    }

    const token = randomBytes(32).toString('base64url');
    const tokenHash = sha256Hex(token);
    await db.insert(emailVerifications).values({
      userId: created.id,
      tokenHash,
      expiresAt: new Date(Date.now() + TOKEN_LIFETIME_MS),
    });

    // log the link to dev console; real email in Ch 64
    console.log(`Verification link: http://localhost:5173/verify-email/${token}`);

    return { success: true };
  },
};
```

Two senior touches:

1. **Valibot's `v.email()`** replaces the hand-rolled `/^[^@]+@[^@]+\.[^@]+$/` regex. The regex was loose (`a@b.c` passes; `user+tag@deep.sub.tld` passes) and tight in the wrong places. Real validation is the roundtrip — the user clicks a link in their inbox or they don't have the inbox.
2. **Atomic insert with `onConflictDoNothing`** — no SELECT-then-INSERT race. Two simultaneous signups for the same email used to result in one row (DB-unique constraint catches it) but a 500 error on the second request because the SELECT had said "free." Now the INSERT itself decides; the loser gets `inserted = []` and we respond with the same enumeration-safe `{ success: true }`. The `if (created === undefined)` guard is mandatory under TS-strict — `[].at(0)` is `User | undefined`, never `User`. Bible rule #3.

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
// In May 2026 Drizzle, `PgTransaction` is the documented public type for the
// transaction handle. Prefer it over the `Parameters<Parameters<...>>` trick
// — easier to read, survives Drizzle's internal refactors. If your Drizzle
// version pre-dates the export, fall back to:
//   type Tx = Parameters<Parameters<typeof db.transaction>[0]>[0];
// (which extracts the first parameter of the callback `db.transaction`
// expects — i.e. the transaction handle.)
import type { PgTransaction } from 'drizzle-orm/pg-core';

type Tx = PgTransaction<never, never, never>;

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

There's a foot-gun a beginner implementation always hits: if you set the impersonation token as the regular `session` cookie, you've **overwritten the admin's own session cookie**. When the admin clicks "End impersonation," there's no admin cookie to return to — they're logged out. The fix is to use a **second cookie** (`session_impersonation`) and teach `handle` to prefer it when present.

Schema additions:

```ts
// in schema.ts: extend sessions
export const sessions = pgTable('sessions', {
  // ... existing fields ...
  impersonatorId: uuid('impersonator_id').references(() => users.id, { onDelete: 'set null' }),
});
```

Action — the impersonation session goes into a *separate* cookie:

```ts
// src/routes/(app)/admin/users/[id]/impersonate/+page.server.ts
import type { Actions } from './$types';
import { redirect, error } from '@sveltejs/kit';
import { dev } from '$app/environment';
import { db } from '$lib/db/client';
import { sessions, users } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { requireRole, withAudit } from '$lib/auth';
import { newSessionToken } from '$lib/sessions';
import { userId } from '$lib/brands';
import { createHash, randomBytes } from 'node:crypto';

const IMPERSONATION_LIFETIME_MS = 15 * 60 * 1000; // 15 minutes — short-lived

function sha256Hex(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export const actions: Actions = {
  default: async (event) => {
    const admin = requireRole(event, 'admin');
    const targetIdParam = event.params.id;
    const [target] = await db.select().from(users).where(eq(users.id, targetIdParam)).limit(1);
    if (target === undefined) {
      throw error(404, 'User not found');
    }

    // Match the encoding picked in Ch 44.7: base64url, 256 bits.
    const token = randomBytes(32).toString('base64url');
    const tokenHash = sha256Hex(token);
    await withAudit(event, 'user.impersonate.start', 'user', target.id, async (tx) => {
      await tx.insert(sessions).values({
        // Re-brand at the boundary: `target.id` is `string` from Drizzle
        // (Postgres uuid → JS string). We brand it through the `userId`
        // constructor so the rest of the system sees `UserId`, not `string`.
        // Same pattern applies to admin.id; `requireRole` already returns a
        // re-branded user, so admin.id is already `UserId`.
        userId: userId(target.id),
        impersonatorId: admin.id,
        tokenHash,
        expiresAt: new Date(Date.now() + IMPERSONATION_LIFETIME_MS),
        fresh: false,
      });
    });

    // Set as a SECOND cookie — the admin's `session` cookie is preserved.
    event.cookies.set('session_impersonation', token, {
      httpOnly: true, secure: !dev, sameSite: 'lax', path: '/',
      maxAge: Math.floor(IMPERSONATION_LIFETIME_MS / 1000),
    });
    throw redirect(303, '/dashboard');
  },
};
```

> **Brand at the boundary, once.** `target.id` arrives as `string` because Drizzle reads Postgres uuids as JS strings. Pass it through `userId(target.id)` (or whichever brand constructor your `$lib/brands` module exposes) at the boundary; downstream code now sees `UserId`, not `string`, so a future bug that swaps `userId` and `impersonatorId` becomes a type error rather than a silent privilege escalation. Re-brand on every read from the DB or HTTP body — anywhere data crosses the trust boundary into your code.

`handle` (in `+hooks.server.ts`) prefers the impersonation cookie when present:

```ts
const authHandle: Handle = async ({ event, resolve }) => {
  const impersonationToken = event.cookies.get('session_impersonation');
  const sessionToken = event.cookies.get('session');

  // Prefer the impersonation token if present; the admin remains identified
  // via the `impersonatorId` field on the impersonation session row.
  const token = impersonationToken ?? sessionToken;

  if (token !== undefined) {
    const tokenHash = sha256Hex(token);
    const rows = await db.select({
      user: users,
      impersonatorId: sessions.impersonatorId,
    })
      .from(sessions)
      .innerJoin(users, eq(sessions.userId, users.id))
      .where(and(eq(sessions.tokenHash, tokenHash), gt(sessions.expiresAt, new Date())))
      .limit(1);

    const row = rows[0];
    if (row !== undefined) {
      event.locals.user = row.user;
      // `row.impersonatorId` is already `string | null` (Drizzle reads the
      // nullable uuid column straight). No `?? null` needed.
      event.locals.impersonatorId = row.impersonatorId;
    }
  }

  return resolve(event);
};
```

End-impersonation action — deletes only the impersonation session, clears only the impersonation cookie:

```ts
// src/routes/(app)/admin/end-impersonation/+server.ts
import type { RequestHandler } from './$types';
import { redirect } from '@sveltejs/kit';
import { db } from '$lib/db/client';
import { sessions } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { createHash } from 'node:crypto';

function sha256Hex(s: string): string {
  return createHash('sha256').update(s).digest('hex');
}

export const POST: RequestHandler = async ({ cookies }) => {
  const token = cookies.get('session_impersonation');
  if (token !== undefined) {
    await db.delete(sessions).where(eq(sessions.tokenHash, sha256Hex(token)));
  }
  cookies.delete('session_impersonation', { path: '/' });
  // The admin's `session` cookie is untouched — they're back in admin context.
  throw redirect(303, '/admin/users');
};
```

In the layout, show a banner when `event.locals.impersonatorId` is set:

```svelte
{#if data.impersonatorId !== null}
  <div class="impersonation-banner">
    Impersonating {data.user.email}.
    <form method="POST" action="/admin/end-impersonation">
      <button type="submit">End impersonation</button>
    </form>
  </div>
{/if}
```

The principle: *separate cookie + handle prefers it + visible banner + audit at start and end + atomic delete on exit*. Without all five, impersonation becomes a backdoor.

> **Session-row hygiene, forward-referenced.** Expired session rows accumulate in `sessions` until pruned. A leaked-cookie attacker can't use them (the row is past `expiresAt`, so `findValidSession` rejects), but the table grows unboundedly and complicates audits. Chapter 64 wires a daily cron at `/api/cron/prune-sessions` that deletes rows where `expiresAt < now()`. We don't need it functioning correctly until Streak has live users; we mention it here so it's on the launch-checklist and not invented from scratch later.

Update `App.Locals` accordingly:

```ts
declare global {
  namespace App {
    interface Locals {
      user?: { id: string; email: string; role: 'user' | 'admin' };
      impersonatorId?: string | null;
      requestId: string;
      log: import('pino').Logger;
    }
  }
}
```

---

## Lesson 47.5 — Recurring concepts from earlier chapters

- **`db.transaction`** (Ch 42) — `withAudit` enforces atomicity.
- **`Result` typing** (Ch 27) — every method that can fail uses it (still applies inside `withAudit`'s `fn`).
- **`+layout.server.ts` central gate** (Ch 45) — `requireRole` lives at the layout level. The original layout-data plumbing was introduced in Ch 32, but the central RBAC gate itself is the Ch 45 hook that populates `event.locals.user` plus the `requireRole` call inside `+layout.server.ts`.

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
const RATE = 5; // tokens
const WINDOW_MS = 60_000;
const MAX_KEYS = 10_000;

// `Map` preserves insertion order, so deleting `keys().next().value` evicts
// the oldest entry — a poor-man's LRU. For a real LRU we'd `delete` then
// re-`set` on every access; this version evicts only on insert when the cap
// is hit, which is fine for rate-limit purposes (the eviction target is
// idle keys, not hot ones).
const buckets = new Map<string, Bucket>();

function setWithCap(key: string, bucket: Bucket): void {
  if (!buckets.has(key) && buckets.size >= MAX_KEYS) {
    const oldest = buckets.keys().next();
    if (!oldest.done) {
      buckets.delete(oldest.value);
    }
  }
  buckets.set(key, bucket);
}

export function consume(key: string, tokensNeeded: number = 1): boolean {
  const now = Date.now();
  const b = buckets.get(key) ?? { tokens: RATE, lastRefill: now };
  const elapsed = now - b.lastRefill;
  if (elapsed > WINDOW_MS) {
    b.tokens = RATE;
    b.lastRefill = now;
  }
  if (b.tokens < tokensNeeded) {
    setWithCap(key, b);
    return false;
  }
  b.tokens -= tokensNeeded;
  setWithCap(key, b);
  return true;
}
```

In login action:

```ts
import { consume } from '$lib/rate-limit';

default: async ({ request, getClientAddress }) => {
  const ip = getClientAddress();
  const data = await request.formData();
  const rawEmail = data.get('email');
  const email = (typeof rawEmail === 'string' ? rawEmail : '').toLowerCase();
  if (!consume(`login:ip:${ip}`) || !consume(`login:email:${email}`)) {
    return fail(429, { error: 'Too many attempts. Try again in a minute.' });
  }
  // ... rest ...
},
```

> **IP and PII boundary.** Raw IPs are fine inside an in-memory rate-limit Map (the keys self-expire and never leave the process). They MUST NOT feed Prometheus labels (Bible #14, cardinality explosion) or audit logs (Bible #15, PII at rest). The rate-limiter never re-emits these — `consume` returns a boolean, not the key. If you ever need to log a rate-limit decision, log the *bucket category* (`'login:ip'`) and not the IP itself.

In production, replace the in-memory map with Redis. Named in the Bible. The 10k cap above is defence-in-depth: the in-memory map would otherwise grow without bound under attack, and a single-instance OOM is a worse outage than a brief rate-limit miss for the evicted key.

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

> **`'unsafe-inline'` for styles is a known relaxation.** We retain `'unsafe-inline'` in `style-src` because Svelte 5 inlines critical SSR styles for first-paint performance — turning it off breaks rendering. The known-relaxations list in `docs/security-review-2026-XX-XX.md` documents this with a planned migration to nonce-based CSP (every SSR'd `<style>` tag gets a per-request nonce; `style-src` becomes `'self' 'nonce-<value>'`). Verify the review doc actually names this relaxation; an undocumented `'unsafe-inline'` is a finding in itself, even if the relaxation is justified.

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

> **The script assumes a server is up.** `pnpm security:headers` runs `fetch` against `:4173`, which is `pnpm preview`'s default port. Either start `pnpm preview` first (in another terminal) or wire a Playwright-style `webServer` config that starts the preview automatically. The bare `vitest run` here will fail with `ECONNREFUSED` if nothing is listening; that's the correct failure mode (no false-positive green) but it's a footgun for new contributors.

---

## Lesson 48.6 — The review document

`docs/security-review-2026-05-05.md`:

```markdown
# Streak — Security Review (2026-05-05)

Author: Billy Ribeiro
Reviewers: <name 1>, <name 2>

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

## Lesson 48.9 — What we deliberately didn't build (and what to add when you need it)

The auth system shipped here is *honest*: passwords, sessions, RBAC, audit, rate limits, headers. It's the floor. Senior teams add these on top, in roughly this order:

- **Multi-factor authentication (TOTP)** — second factor at login, especially for admins. Store an encrypted shared secret per user; verify a 6-digit code via `otpauth` lib. Enforce on `requireRole('admin')`.
- **Passkeys / WebAuthn** — the May 2026 default for any new auth system. Better security and better UX than passwords. Libraries: `@simplewebauthn/server` + `@simplewebauthn/browser`. Lets the user log in with Touch ID / Face ID / a hardware key and skip passwords entirely.
- **SSO / OAuth (Google, GitHub, Microsoft)** — for B2B audiences who'd refuse to create another password. Use `arctic` (TS-first OAuth helper); pair with the existing session table.
- **Account deletion + GDPR data export** — a `/settings/danger-zone` route that exports all user data as a downloadable JSON, then 30-day soft-delete window before hard delete (with cascading deletes on habits/sessions/audit-log-with-anonymized-actor).
- **Account-lockout after N failed attempts** — separate from rate-limiting. After 10 failed logins for `user@x.com` (over any IP), the account locks for 15 minutes. A `users.failed_login_attempts INTEGER` + `users.locked_until TIMESTAMP` columns + atomic UPDATE.
- **Session forensics** — `sessions.user_agent`, `sessions.ip_address`, `sessions.last_seen_at` columns + a `/settings/sessions` page where the user can see and revoke every active session ("log out of all other devices").
- **Push-style audit alerts** — when a sensitive action happens (password change, MFA disabled, new device login), email the user immediately so a real account-takeover is visible to them within seconds.

None of these are exotic; all of them are normal for a real fintech-shaped app. Each is roughly 2–4 hours of work + tests. Ship them when the threat model justifies them, not before.

---

## End-of-chapter checkpoint

- [ ] Rate-limiting is live on `/login` and `/signup`.
- [ ] Security headers are set in `handle`.
- [ ] `pnpm security:headers` passes.
- [ ] You wrote the review document.
- [ ] You can name three things in 48.9 that should land before public beta.

End of Part VII. Next: money, polish, every Svelte primitive.
