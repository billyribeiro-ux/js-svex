# Part VIII — Real money, real polish, the full Svelte catalogue

> *"By Chapter 55, Streak is paid, animated, indexable, offline-capable. Every Svelte 5 + SvelteKit primitive has had real airtime."*

---

# Chapter 49 — Stripe Checkout, idempotent client calls, the `Money` module reused

> *Today's job:* "Upgrade to Pro" creates a Stripe Checkout Session. Visible win: pay with a Stripe test card; come back to a Pro-only feature.

---

## Lesson 49.1 — Stripe setup

1. Create a Stripe account (test mode is free, indefinitely).
2. Get a *restricted API key* with only the scopes Streak needs.
3. Store as `STRIPE_SECRET_KEY` in `.env`.

```bash
pnpm add stripe
```

```ts
// src/lib/stripe.ts
import Stripe from 'stripe';
import { STRIPE_SECRET_KEY } from '$env/static/private';

export const stripe = new Stripe(STRIPE_SECRET_KEY, {
  apiVersion: '2026-04-30.acacia', // pin the dated + named form; bump after testing
  maxNetworkRetries: 2,
  timeout: 10_000, // 10s — Bible rule
});
```

> **`apiVersion`** pins your code to a specific Stripe API release. As of the May 2026 SDK, versions are dated *and* named (e.g. `'2026-04-30.acacia'`); the name suffix is the release codename. Stripe ties API versions to TS SDK versions; check the SDK changelog and pin both. Bump deliberately after reading the changelog and running tests; otherwise Stripe will use your dashboard's "default" version, which can shift under you.

---

## Lesson 49.2 — Subscriptions schema

```ts
export const subscriptions = pgTable('subscriptions', {
  id: uuid('id').primaryKey().defaultRandom(),
  userId: uuid('user_id').notNull().references(() => users.id, { onDelete: 'cascade' }),
  stripeCustomerId: text('stripe_customer_id').notNull().unique(),
  stripeSubscriptionId: text('stripe_subscription_id').unique(),
  plan: text('plan', { enum: ['free', 'pro'] }).notNull().default('free'),
  status: text('status'), // active, past_due, cancelled, ...
  currentPeriodEnd: timestamp('current_period_end', { withTimezone: true }),
});
```

---

## Lesson 49.3 — The `/billing` action

```ts
// src/routes/(app)/billing/+page.server.ts
import type { Actions } from './$types';
import { redirect } from '@sveltejs/kit';
import { stripe } from '$lib/stripe';
import { db } from '$lib/db/client';
import { subscriptions } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { requireUser } from '$lib/auth';
import { PUBLIC_SITE_URL } from '$env/static/public';

const PRO_PRICE_ID = 'price_xxx_replace_me';

export const actions: Actions = {
  upgrade: async (event) => {
    const user = requireUser(event);

    const [sub] = await db.select().from(subscriptions).where(eq(subscriptions.userId, user.id)).limit(1);

    let stripeCustomerId = sub?.stripeCustomerId;
    if (stripeCustomerId === null || stripeCustomerId === undefined) {
      const customer = await stripe.customers.create(
        { email: user.email, metadata: { userId: user.id } },
        { idempotencyKey: `customer:${user.id}` },
      );
      stripeCustomerId = customer.id;
      await db.insert(subscriptions).values({ userId: user.id, stripeCustomerId, plan: 'free' });
    }

    const session = await stripe.checkout.sessions.create(
      {
        mode: 'subscription',
        customer: stripeCustomerId,
        line_items: [{ price: PRO_PRICE_ID, quantity: 1 }],
        success_url: `${PUBLIC_SITE_URL}/billing/success?session={CHECKOUT_SESSION_ID}`,
        cancel_url: `${PUBLIC_SITE_URL}/billing`,
      },
      // Idempotency key: pin to a fixed window so a fast double-click of *Upgrade*
      // re-uses the same Checkout Session instead of creating two. We bucket by
      // user × hour; longer windows would risk re-using a stale session.
      { idempotencyKey: `checkout:${user.id}:${Math.floor(Date.now() / 3_600_000)}` },
    );

    if (session.url === null) {
      throw new Error('Stripe Checkout did not return a redirect URL');
    }
    throw redirect(303, session.url);
  },
};
```

`idempotencyKey` on every mutating Stripe call. Bible rule. We also defensively check `session.url !== null` — Stripe's type allows `null`, even though every Checkout Session in practice has a URL.

The hourly bucket (`Math.floor(Date.now() / 3_600_000)`) is intentional for double-click protection — if the user clicks *Upgrade* twice in quick succession, both calls hit the same key and Stripe returns the cached session. The trade-off: if a user starts checkout, abandons, and retries 30 minutes later within the same hour, they get the same session URL (which Stripe will accept). For explicit retry semantics, vary the key with a sequence number stored in the session row.

---

## Lesson 49.4 — Reading: the `Money` module in production

The `formatCents`, `applyBps`, `splitCents` helpers from Chapter 30 are now imported by:
- the pricing page (display the Pro tier price),
- the billing settings page (show "next charge: $9.99 on YYYY-MM-DD"),
- the webhook handler (Ch 50, where amounts arrive).

The compounding-fluency spine the plan promised.

---

## Lesson 49.5 — Recurring concepts from earlier chapters

- **`Cents` + `formatCents`** (Ch 30) — display every Stripe price.
- **DB pool timeouts** (Ch 40) — the Stripe client gets the same `timeout` discipline.
- **Idempotency** — the per-user-per-hour bucket replaces unbounded `Date.now()`.
- **`requireUser`** (Ch 45) — every billing action requires a logged-in user.

---

## Lesson 49.6 — What you can now read in the wild

After Chapter 49 you can:

- Read **`new Stripe(secret, { apiVersion, maxNetworkRetries, timeout })`** and explain each option.
- Read **`stripe.customers.create(...)`**, **`stripe.checkout.sessions.create(...)`** with idempotency keys.
- Spot a **stale-idempotency-key bug** (using `Date.now()` per-call defeats the purpose).
- Pick the right **idempotency window** (per-user-per-hour for Checkout; per-event for webhooks).

---

## End-of-chapter checkpoint

- [ ] Click *Upgrade* → land on Stripe → pay with `4242 4242 4242 4242` → return to `/billing/success`.
- [ ] `subscriptions` row exists.

---

# Chapter 50 — Webhooks, idempotency keys, "exactly once"

> *Today's job:* Stripe pings `/api/stripe/webhook`. We mark Pro exactly once even on retries. Visible win: Stripe-CLI replay; user is Pro; `webhook_events` has one row.

---

## Lesson 50.1 — `webhook_events` table

```ts
export const webhookEvents = pgTable('webhook_events', {
  id: text('id').primaryKey(), // Stripe event ID
  receivedAt: timestamp('received_at', { withTimezone: true }).notNull().defaultNow(),
  processedAt: timestamp('processed_at', { withTimezone: true }),
});
```

---

## Lesson 50.2 — The webhook handler

```ts
// src/routes/api/stripe/webhook/+server.ts
import type { RequestHandler } from './$types';
import { error, text } from '@sveltejs/kit';
import type Stripe from 'stripe';
import { stripe } from '$lib/stripe';
import { db } from '$lib/db/client';
import { subscriptions, webhookEvents } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { STRIPE_WEBHOOK_SECRET } from '$env/static/private';
import { logger } from '$lib/logger';

/**
 * Extract a Stripe customer ID from an event payload field.
 *
 * Throws when `value` is null. Callers MUST only pass values from events
 * where `customer` is guaranteed non-null:
 *   - `checkout.session.completed`
 *   - `customer.subscription.created` / `.updated` / `.deleted`
 *   - `invoice.payment_failed` / `.payment_succeeded`
 *
 * For events where `customer` can legitimately be null (e.g. some
 * `customer.deleted` edge cases), this throw is wrong — handle that path
 * separately rather than routing through this helper.
 */
function customerIdFrom(value: string | Stripe.Customer | Stripe.DeletedCustomer | null): string {
  if (typeof value === 'string') return value;
  if (value === null) throw new Error('Stripe event missing customer');
  return value.id;
}

function periodEndOf(sub: Stripe.Subscription): Date {
  // Stripe moved current_period_end to the item level in 2024+.
  // Take the latest period_end across items as the subscription's effective period end.
  const ends = sub.items.data.map((item) => item.current_period_end);
  if (ends.length === 0) {
    throw new Error('subscription has no items with current_period_end');
  }
  const max = Math.max(...ends);
  return new Date(max * 1000);
}

export const POST: RequestHandler = async ({ request }) => {
  const sig = request.headers.get('stripe-signature');
  if (sig === null) throw error(400, 'No signature');

  const raw = await request.text();
  let event: Stripe.Event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig, STRIPE_WEBHOOK_SECRET);
  } catch {
    throw error(400, 'Invalid signature');
  }

  // dedupe
  const inserted = await db.insert(webhookEvents)
    .values({ id: event.id })
    .onConflictDoNothing()
    .returning();
  if (inserted.length === 0) return text('already processed');

  // process inside transaction
  await db.transaction(async (tx) => {
    if (event.type === 'checkout.session.completed') {
      const session = event.data.object;
      const customerId = customerIdFrom(session.customer);
      const subscriptionId = typeof session.subscription === 'string'
        ? session.subscription
        : session.subscription?.id;
      if (subscriptionId === undefined) return;

      // re-fetch canonical state (defence in depth)
      const sub = await stripe.subscriptions.retrieve(subscriptionId);

      await tx.update(subscriptions)
        .set({
          stripeSubscriptionId: sub.id,
          plan: 'pro',
          status: sub.status,
          currentPeriodEnd: periodEndOf(sub),
        })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'customer.subscription.updated') {
      // Plan change, past_due transitions, cancellation scheduled — keep DB in sync.
      const sub = event.data.object;
      const customerId = customerIdFrom(sub.customer);
      await tx.update(subscriptions)
        .set({
          status: sub.status,
          currentPeriodEnd: periodEndOf(sub),
          plan: sub.status === 'active' || sub.status === 'trialing' ? 'pro' : 'free',
        })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'customer.subscription.deleted') {
      const sub = event.data.object;
      const customerId = customerIdFrom(sub.customer);
      await tx.update(subscriptions)
        .set({ plan: 'free', status: 'cancelled' })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'invoice.payment_failed') {
      // First failed renewal → status moves to 'past_due'. Stripe will retry per
      // your dunning settings; meanwhile we keep the user as Pro (grace period)
      // but flag the status so the UI can show a banner.
      const invoice = event.data.object;
      const customerId = customerIdFrom(invoice.customer);
      await tx.update(subscriptions)
        .set({ status: 'past_due' })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'invoice.payment_succeeded') {
      // Successful renewal — clear past_due if set.
      const invoice = event.data.object;
      const customerId = customerIdFrom(invoice.customer);
      await tx.update(subscriptions)
        .set({ status: 'active' })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'customer.subscription.trial_will_end') {
      // 3 days before trial ends — Stripe nudges us. We log; UI can prompt.
      // (No DB write needed; the subscription's status will update on its own.)
      logger.info({ customerId: customerIdFrom(event.data.object.customer) }, 'stripe.trial.ending');
    }

    await tx.update(webhookEvents)
      .set({ processedAt: new Date() })
      .where(eq(webhookEvents.id, event.id));
  });

  return text('ok');
};
```

The events above cover the canonical Stripe **dunning state machine**: subscriptions move `active → past_due → unpaid → canceled` over multiple weeks if a card fails, and we want our DB to reflect that without manual reconciliation. The dunning states are `past_due`, `unpaid`, `canceled`; `trial_will_end` is a courtesy ping, not dunning. We log it for observability and let the subsequent `customer.subscription.updated` event do the real work.

> **dunning** *(noun)* — the polite-collection process for failed payments. Stripe retries cards up to 4 times over 3 weeks (per your dunning settings). The webhook stream tells you which state the user is currently in.

Configure Stripe dashboard → Settings → Subscriptions and emails → Smart Retries: 4 attempts over ~3 weeks; on final failure, Stripe fires `customer.subscription.deleted` and we drop the user to free.

**Important sequence — dedupe vs transaction.** The dedupe insert (`onConflictDoNothing`) MUST happen *outside* the transaction that processes the body, OR happen *inside* the transaction with the body. We've placed dedupe BEFORE the transaction, which means a thrown error during the body rolls back the body but keeps the dedupe row — correct exactly-once semantics, since Stripe's retry will see the dedupe row and skip. The trade-off: if our handler crashes hard (process killed mid-flight), the dedupe row may be there but the body partial. Webhook retries don't help because the dedupe says "seen." The runbook is to manually re-process by deleting the dedupe row and re-triggering via Stripe CLI. Most production systems accept this trade-off.

---

## Lesson 50.3 — Why we re-fetch

We don't trust the webhook payload's `current_period_end` — it could be stale or the event could be replayed in a weird order. We refetch the canonical state from Stripe. Defence in depth.

---

## Lesson 50.4 — `pnpm test:webhook:duplicate`

Stripe ships a helper, `stripe.webhooks.generateTestHeaderString`, that signs a raw payload exactly like Stripe would. We use it to build a test that hits the real handler against a real Postgres. (Pinned as of the May 2026 SDK; verify on upgrade — Stripe occasionally renames internal helpers.)

```ts
// tests/integration/stripe-webhook.dedupe.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import Stripe from 'stripe';
import { db } from '$lib/db/client';
import { users, subscriptions, webhookEvents } from '$lib/db/schema';
import { sql } from 'drizzle-orm';
import { STRIPE_WEBHOOK_SECRET } from '$env/static/private';

const BASE = process.env.TEST_BASE_URL ?? 'http://localhost:4173';
const stripe = new Stripe('sk_test_dummy', { apiVersion: '2026-04-30.acacia' });

const TEST_USER = '00000000-0000-0000-0000-000000000099';

beforeEach(async () => {
  await db.execute(sql`TRUNCATE webhook_events, subscriptions, users RESTART IDENTITY CASCADE`);
  await db.insert(users).values({ id: TEST_USER, email: 't@t.com', passwordHash: 'fake' });
  await db.insert(subscriptions).values({
    userId: TEST_USER,
    stripeCustomerId: 'cus_test',
    plan: 'free',
  });
});

describe('stripe webhook dedup', () => {
  it('processes the same event id only once', async () => {
    const payload = JSON.stringify({
      id: 'evt_test_123',
      type: 'customer.subscription.deleted',
      data: { object: { id: 'sub_x', customer: 'cus_test', items: { data: [{ current_period_end: 1700000000 }] } } },
    });
    const header = stripe.webhooks.generateTestHeaderString({ payload, secret: STRIPE_WEBHOOK_SECRET });

    const post = (): Promise<Response> => fetch(`${BASE}/api/stripe/webhook`, {
      method: 'POST', body: payload, headers: { 'stripe-signature': header },
    });

    const r1 = await post();
    const r2 = await post();

    expect(await r1.text()).toBe('ok');
    expect(await r2.text()).toBe('already processed');

    const events = await db.select().from(webhookEvents);
    expect(events).toHaveLength(1);
  });
});
```

Run with `pnpm preview` in one terminal and `pnpm test:integration` in another (both pointed at the test database). The runtime evidence is `events.length === 1` after two identical POSTs.

We bypass `stripe.subscriptions.retrieve` in this test by using `customer.subscription.deleted`, which carries the subscription in the event payload. For `checkout.session.completed`, the handler calls `retrieve` — that path needs a separate mock or live test.

---

## Lesson 50.5 — Recurring concepts from earlier chapters

- **`+server.ts`** (Ch 33's preview, formal use here) — REST endpoint.
- **`onConflictDoNothing()` + `.returning()`** — atomic dedupe in one query.
- **`db.transaction`** (Ch 42) — webhook update + dedupe row update happen atomically.
- **Defence in depth** (Ch 6) — re-fetch canonical state instead of trusting payload.

---

## Lesson 50.6 — What you can now read in the wild

After Chapter 50 you can:

- Read **`stripe.webhooks.constructEvent(rawBody, sig, secret)`** as the verify-or-fail gate.
- Read **`stripe.webhooks.generateTestHeaderString(...)`** as the test-fixture signer.
- Spot **untyped event handlers** (`session.customer as string`) as a refactor target.
- Pick the right **idempotency-shaped table** (Stripe event ID as primary key, not a generated UUID).

---

## End-of-chapter checkpoint

- [ ] Stripe-CLI test event arrives; subscriptions row updates.
- [ ] Replay; second time is "already processed".
- [ ] Test green.

---

# Chapter 51 — Image upload to R2 via presigned URLs

> *Today's job:* habit icons. Visible win: drag an image; it's there.

---

## Lesson 51.1 — Why presigned URLs

Streaming a 5 MB upload through SvelteKit ties a worker for the upload duration. Direct-to-R2 doesn't. The server signs a URL; the client uploads directly.

---

## Lesson 51.2 — The presign endpoint

```ts
// src/routes/api/uploads/presign/+server.ts
import type { RequestHandler } from './$types';
import { json, error } from '@sveltejs/kit';
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { NodeHttpHandler } from '@smithy/node-http-handler';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import * as v from 'valibot';
import { R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET } from '$env/static/private';
import { requireUser } from '$lib/auth';
import { logger } from '$lib/logger';

const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp', 'image/avif'] as const;
const MAX_BYTES = 5 * 1024 * 1024;

const PresignSchema = v.object({
  // Spread the readonly tuple if your Valibot version complains: v.picklist([...ALLOWED_MIME]).
  contentType: v.picklist(ALLOWED_MIME),
  size: v.pipe(v.number(), v.integer(), v.minValue(1), v.maxValue(MAX_BYTES)),
});

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: { accessKeyId: R2_ACCESS_KEY_ID, secretAccessKey: R2_SECRET_ACCESS_KEY },
  requestHandler: new NodeHttpHandler({
    connectionTimeout: 5_000,
    requestTimeout: 10_000,
  }),
});

export const POST: RequestHandler = async (event) => {
  const user = requireUser(event);
  const raw: unknown = await event.request.json();
  const parsed = v.safeParse(PresignSchema, raw);
  if (!parsed.success) {
    logger.warn({ issues: parsed.issues }, 'invalid presign request');
    throw error(400, 'Invalid request');
  }
  const { contentType, size } = parsed.output;

  const key = `users/${user.id}/${crypto.randomUUID()}`;
  const command = new PutObjectCommand({
    Bucket: R2_BUCKET,
    Key: key,
    ContentType: contentType,
    ContentLength: size,
  });

  const url = await getSignedUrl(s3, command, { expiresIn: 60 });
  return json({ url, key });
};
```

Two boundary-discipline pieces:

1. **`v.safeParse(PresignSchema, raw)`** — body is `unknown` until parsed. We refuse anything outside the four allowed MIMEs and the 5 MB cap.
2. **`new NodeHttpHandler({ connectionTimeout, requestTimeout })`** — the AWS SDK v3 wraps Node's HTTP agent through this handler class. Timeouts go on the handler, not the client config object directly. Bible rule #13 applied.

The client then `fetch(url, { method: 'PUT', body: file })` directly to R2.

**R2 credentials rotation.** The S3 client reads credentials at module init from `$env/static/private`. On a deployed server, key rotation requires a redeploy to pick up new credentials. The rotation runbook (Ch 57) covers this; the alternative — re-reading via `$env/dynamic/private` per request — adds latency we don't need on a fintech-shaped flow with low rotation frequency.

---

## Lesson 51.3 — Magic-number MIME validation

After upload, on a confirmation endpoint, fetch the first bytes from R2 and assert the file's binary signature matches its declared MIME. Don't trust the client's claim — a malicious client could upload an HTML file declared as `image/png`, and a permissive viewer would render it.

```ts
// src/lib/uploads/sniff.ts
const SIGNATURES: ReadonlyArray<{ mime: string; bytes: ReadonlyArray<number | null>; offset?: number }> = [
  // PNG: 89 50 4E 47 0D 0A 1A 0A
  { mime: 'image/png', bytes: [0x89, 0x50, 0x4e, 0x47, 0x0d, 0x0a, 0x1a, 0x0a] },
  // JPEG: FF D8 FF (variants share the first 3 bytes)
  { mime: 'image/jpeg', bytes: [0xff, 0xd8, 0xff] },
  // WebP: "RIFF" .... "WEBP" — bytes 0..3 = RIFF, bytes 8..11 = WEBP
  { mime: 'image/webp', bytes: [0x52, 0x49, 0x46, 0x46, null, null, null, null, 0x57, 0x45, 0x42, 0x50] },
];

export function sniffMime(prefix: Uint8Array): string | null {
  for (const sig of SIGNATURES) {
    const match = sig.bytes.every((b, i) => b === null || prefix[i] === b);
    if (match) {
      return sig.mime;
    }
  }
  return null;
}
```

Wire it into a `POST /api/uploads/confirm` endpoint that runs after the direct PUT succeeds:

```ts
// src/routes/api/uploads/confirm/+server.ts
import type { RequestHandler } from './$types';
import { json, error } from '@sveltejs/kit';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { NodeHttpHandler } from '@smithy/node-http-handler';
import * as v from 'valibot';
import { R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET } from '$env/static/private';
import { requireUser } from '$lib/auth';
import { sniffMime } from '$lib/uploads/sniff';

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: { accessKeyId: R2_ACCESS_KEY_ID, secretAccessKey: R2_SECRET_ACCESS_KEY },
  requestHandler: new NodeHttpHandler({ connectionTimeout: 5_000, requestTimeout: 10_000 }),
});

const ConfirmSchema = v.object({
  key: v.string(),
  declaredMime: v.picklist(['image/jpeg', 'image/png', 'image/webp']),
});

export const POST: RequestHandler = async (event) => {
  const user = requireUser(event);
  const raw: unknown = await event.request.json();
  const parsed = v.safeParse(ConfirmSchema, raw);
  if (!parsed.success) {
    throw error(400, 'Invalid request');
  }
  const { key, declaredMime } = parsed.output;

  if (!key.startsWith(`users/${user.id}/`)) {
    throw error(403, 'Not your upload');
  }

  // Read the first 12 bytes via Range request — cheap, no full download.
  const head = await s3.send(new GetObjectCommand({ Bucket: R2_BUCKET, Key: key, Range: 'bytes=0-11' }));
  const body = head.Body;
  if (body === undefined) {
    throw error(404, 'Object not found');
  }
  const chunks: Array<Uint8Array> = [];
  for await (const chunk of body as AsyncIterable<Uint8Array>) {
    chunks.push(chunk);
  }
  const total = chunks.reduce((acc, c) => acc + c.length, 0);
  const prefix = new Uint8Array(total);
  let offset = 0;
  for (const c of chunks) {
    prefix.set(c, offset);
    offset += c.length;
  }

  const actualMime = sniffMime(prefix);
  if (actualMime === null || actualMime !== declaredMime) {
    throw error(415, 'File contents do not match declared type');
  }

  return json({ ok: true });
};
```

Two things to note: (1) we use `Range: bytes=0-11` so we don't pay to download the whole file just to inspect 12 bytes; (2) the namespace check `key.startsWith(\`users/${user.id}/\`)` prevents a logged-in attacker from confirming someone else's upload.

---

## Lesson 51.4 — `enhanced-img` for static images

```bash
pnpm add -D @sveltejs/enhanced-img
```

```svelte
<script>
  import heroSrc from '$lib/assets/hero.png?enhanced';
</script>

<enhanced:img src={heroSrc} alt="Streak hero" />
```

Generates AVIF/WebP, multiple sizes, correct width/height. Lighthouse loves it.

---

## Lesson 51.5 — Recurring concepts from earlier chapters

- **`requireUser`** (Ch 45) — every uploader is authenticated.
- **Boundary parsing with Valibot** (Ch 41) — every JSON request body is `unknown` until parsed.
- **Connect/request timeouts** (Bible rule #13) — applied to the S3 client.

---

## Lesson 51.6 — What you can now read in the wild

After Chapter 51 you can:

- Read **`getSignedUrl(s3, new PutObjectCommand(...), { expiresIn })`** as the presigned-URL primitive.
- Read **`new NodeHttpHandler({ connectionTimeout, requestTimeout })`** as the way to apply timeouts to AWS SDK v3.
- Spot **trusting client-declared MIME** as a security gap and add magic-number sniffing.
- Read **`@sveltejs/enhanced-img` import-with-`?enhanced`** as build-time image optimization.

---

## End-of-chapter checkpoint

- [ ] Habits can have icons (uploaded direct to R2).
- [ ] Marketing hero image goes through `enhanced-img`.

---

# Chapter 52 — The full Svelte 5 motion catalogue

> *Today's job:* the app moves. Visible win: every list change feels alive; reduced-motion users get static UI.

---

## Lesson 52.1 — Built-in transitions

```svelte
<script lang="ts">
  import { fade, fly, slide, scale, blur, draw, crossfade } from 'svelte/transition';
  import { quintOut, elasticInOut } from 'svelte/easing';
</script>

{#each items as item (item.id)}
  <li transition:fly={{ y: 10, duration: 200, easing: quintOut }}>{item.name}</li>
{/each}

{#if open}
  <div transition:fade>...</div>
{/if}
```

---

## Lesson 52.2 — `animate:flip` for reorder

```svelte
{#each habits as habit (habit.id)}
  <li animate:flip={{ duration: 200 }}>{habit.name}</li>
{/each}
```

When the list reorders, items glide to their new positions. Swap a habit's index and watch.

---

## Lesson 52.3 — `svelte/motion` — `Tween`, `Spring`

As of Svelte 5.x (May 2026), `Spring` is exported as a class with `current`, `target`, and `set()`. The class form is preferred for new code; the legacy `spring()` factory still exists alongside it for backwards compatibility. Svelte 5 provides runes-style classes (`Tween`, `Spring`) and the legacy stores (`tweened`, `spring`):

```svelte
<script lang="ts">
  import { Spring } from 'svelte/motion';
  let count = $state(0);
  const display = new Spring(0, { stiffness: 0.1, damping: 0.4 });
  $effect(() => { display.target = count; });
</script>

<button type="button" onclick={() => count += 1}>+1</button>
<p>{display.current.toFixed(0)}</p>
```

The number bounces toward its target. `display.current` is the live value; `display.target` is what we're animating toward. Setting `target` is the trigger.

(The legacy `import { spring } from 'svelte/motion'` + `$display` store form still works in 5.x; you'll see it in older codebases.)

---

## Lesson 52.4 — `prefers-reduced-motion`

Some users get motion sick from animation; the OS exposes their preference and we respect it. Svelte 5 ships the reactive read for you — **`prefersReducedMotion` from `svelte/motion`**:

```svelte
<script lang="ts">
  import { fly } from 'svelte/transition';
  import { prefersReducedMotion } from 'svelte/motion';
</script>

{#each items as item (item.id)}
  <li transition:fly={prefersReducedMotion.current ? { duration: 0 } : { y: 10, duration: 200 }}>
    {item.name}
  </li>
{/each}
```

Read aloud:

| Expression | Read aloud as |
|---|---|
| `import { prefersReducedMotion } from 'svelte/motion'` | *"Bring in Svelte's reactive read of the OS reduce-motion setting."* |
| `prefersReducedMotion.current` | *"True if the user has asked their OS to reduce motion — read reactively, right now."* |
| `? { duration: 0 }` | *"If so, animate over zero milliseconds — i.e. don't."* |
| `: { y: 10, duration: 200 }` | *"Otherwise, the normal 200 ms fly."* |

`prefersReducedMotion` is a `MediaQuery` instance: `.current` is a reactive boolean that flips the moment the user changes the OS setting, with no listener wiring on your side. One import, one property, WCAG-grade. Senior habit.

### Under the hood — a reusable pattern

It's worth seeing what `.current` is *doing*, because the same shape solves a dozen other "reactive read of a browser API" problems. Hand-rolled, it's a module-scope rune backed by a `matchMedia` listener:

```ts
// src/lib/motion.svelte.ts — illustrative; prefer the built-in above in real code
import { browser } from '$app/environment';

let _reduced = $state(browser ? window.matchMedia('(prefers-reduced-motion: reduce)').matches : false);

if (browser) {
  window.matchMedia('(prefers-reduced-motion: reduce)').addEventListener('change', (event) => {
    _reduced = event.matches;
  });
}

export const prefersReducedMotion = {
  get current(): boolean {
    return _reduced;
  },
};
```

Two things this teaches, even though you'll reach for the built-in in real code:

- **`import { browser } from '$app/environment'`** — matches the senior pattern from Ch 38; preferred over hand-rolled `typeof window` checks because it's tree-shaken at build time and avoids the temptation to reach for `window` accidentally inside the SSR branch.
- **Module-scope `$state` and Ch 29's SSR-singleton rule.** Ch 29 banned per-request mutable state at module scope on the server (one bucket would be shared across every request). This `_reduced` is fine because: (a) on the server, `browser` is `false`, the listener never runs, and `_reduced` is the constant `false` for everyone; (b) in the browser, this module is loaded once per tab, set once via `addEventListener`, and the "singleton" is bounded by tab lifetime, not request lifetime. The real `svelte/motion` export *is* this exact reasoning, written by the Svelte team — which is why it exposes `.current` to match the `MediaQuery` interface, and why we mirror that name above instead of inventing our own.

---

## Lesson 52.5 — `use:` actions

```ts
// src/lib/actions/intersect.ts
// Fires `callback` every time the element scrolls into view. The callback
// will fire repeatedly if the element scrolls in/out. For "fire once on
// first intersection" use the once-only variant below.
export function intersect(node: HTMLElement, callback: () => void): { destroy: () => void } {
  const obs = new IntersectionObserver((entries) => {
    if (entries[0]?.isIntersecting) {
      callback();
    }
  }, { threshold: 0.1 });
  obs.observe(node);
  return { destroy: () => obs.disconnect() };
}

// Once-only variant: unobserve after the first match. Use this for analytics
// "saw this element" events where re-firing would inflate counts.
export function intersectOnce(node: HTMLElement, callback: () => void): { destroy: () => void } {
  const obs = new IntersectionObserver((entries) => {
    if (entries[0]?.isIntersecting) {
      callback();
      obs.unobserve(node);
    }
  }, { threshold: 0.1 });
  obs.observe(node);
  return { destroy: () => obs.disconnect() };
}
```

```svelte
<div use:intersect={() => console.log('visible!')}>...</div>
```

Pick `intersect` for "lazy-load images as they scroll in" (you want repeats if the user scrolls back up); pick `intersectOnce` for "fire a 'viewed-pricing' analytic event" (you want exactly one).

---

## Lesson 52.6 — `{@attach}` directive

In Svelte 5.x as of May 2026, `{@attach}` is a successor to actions for some cases. The current syntax is `{@attach attachmentFn}` where `attachmentFn(node)` returns either `void` or a cleanup function:

```svelte
<script lang="ts">
  function setup(node: HTMLElement): () => void {
    // setup
    return () => {
      // cleanup
    };
  }
</script>

<div {@attach setup}>
  ...
</div>
```

Use it when the lifetime is bound to render rather than the DOM element being long-lived. (Pin to current docs; Svelte 5's attachment API is one of the surfaces the team is still iterating on.)

---

## Lesson 52.7 — Recurring concepts from earlier chapters

- **`{#each ... (key)}`** (Ch 9) — keyed-each is what makes `animate:flip` work.
- **`prefers-reduced-motion`** — accessibility-first by default.
- **`$effect` for genuine side-effects** (Ch 17) — `display.target = count` is one of the legitimate uses.

---

## Lesson 52.8 — What you can now read in the wild

After Chapter 52 you can:

- Read **`transition:fly`, `animate:flip`, `use:foo`, `{@attach (node) => ...}`** and explain the lifecycle of each.
- Read the **`Spring` / `Tween` runes-style classes** in `svelte/motion`.
- Spot a **missing `prefers-reduced-motion` gate** as an a11y bug.
- Write a **custom `use:` action** with `destroy` cleanup.

---

## End-of-chapter checkpoint

- [ ] Every list animates correctly.
- [ ] Reduced-motion users see no transitions.
- [ ] You've used `transition:`, `animate:flip`, `svelte/motion`, `use:`, `{@attach}`.

---

# Chapter 53 — Forms, validation patterns, the framework's full form story

> *Today's job:* every form follows one consistent pattern — accessible, progressively enhanced, type-safe.

---

## Lesson 53.1 — `formAction(schema, handler)`

```ts
// src/lib/validation/formAction.ts
import { fail } from '@sveltejs/kit';
import * as v from 'valibot';
import type { Action, RequestEvent } from '@sveltejs/kit';

export function formAction<T>(
  schema: v.GenericSchema<unknown, T>,
  handler: (input: T, event: RequestEvent) => Promise<{ success?: boolean; redirect?: string } | void>,
): Action {
  return async (event: RequestEvent) => {
    const data = await event.request.formData();
    const obj: Record<string, unknown> = {};
    for (const [k, val] of data.entries()) {
      obj[k] = val;
    }
    const parsed = v.safeParse(schema, obj);
    if (!parsed.success) {
      const fieldErrors: Record<string, string> = {};
      for (const issue of parsed.issues) {
        const key = issue.path?.[0]?.key ?? 'form';
        if (typeof key === 'string') {
          fieldErrors[key] = issue.message;
        }
      }
      return fail(400, { fieldErrors });
    }
    return handler(parsed.output, event);
  };
}
```

Two boundary-discipline pieces in this snippet:

- **Loop variable named `val`, not `v`** — `v` is already imported as the Valibot namespace at the top of the file. Reusing the name shadows it inside the loop body and makes future edits ("oh, I'll just call `v.something()` here") fail in confusing ways. Always rename.
- **Explicit return type annotation `: Action`** — matches SvelteKit's `Action` shape and makes the inferred form actions type-checked at the page level. Without it, the returned function's type is structurally compatible but isn't the *named* type SvelteKit's tooling expects.

Used as:

```ts
export const actions = {
  addHabit: formAction(HabitInputSchema, async (input, event) => {
    const user = requireUser(event);
    await withAudit(event, 'habit.add', 'habit', null, async (tx) => {
      await tx.insert(habits).values({ userId: user.id, ...input });
    });
    return { success: true };
  }),
};
```

---

## Lesson 53.2 — Function bindings for normalization

As of Svelte 5.x (May 2026), function-form bindings use the comma-separated getter/setter syntax inside the binding expression:

```svelte
<input bind:value={() => email, (v) => email = v.toLowerCase().trim()} />
```

The `email` value is always normalized as the user types. (Earlier 5.0 betas used an array-tuple form `[() => email, (v) => ...]`; that has been replaced — pin to the comma form for new code.)

---

## Lesson 53.3 — ARIA error wiring

```svelte
<label for="email">Email</label>
<input
  id="email"
  name="email"
  type="email"
  autocomplete="email"
  aria-invalid={form?.fieldErrors?.email ? 'true' : undefined}
  aria-describedby={form?.fieldErrors?.email ? 'email-error' : undefined}
  required
/>
{#if form?.fieldErrors?.email}
  <p id="email-error" class="error">{form.fieldErrors.email}</p>
{/if}
```

Screen readers announce the error when focus enters an invalid field.

---

## Lesson 53.4 — Snapshots

For a multi-step form that survives navigation:

```svelte
<script lang="ts">
  let formState = $state({ name: '', email: '' });

  export const snapshot = {
    capture: () => formState,
    restore: (v: typeof formState) => formState = v,
  };
</script>
```

Navigate away, come back; form is restored.

---

## Lesson 53.5 — Autocomplete tokens

| Field | Autocomplete value |
|---|---|
| Email | `email` |
| New password | `new-password` |
| Current password | `current-password` |
| OTP | `one-time-code` |
| Name | `name` / `given-name` / `family-name` |
| Address | `street-address` / `postal-code` / etc. |

Always set them. Free UX win.

---

## Lesson 53.6 — Recurring concepts from earlier chapters

- **Valibot schemas** (Ch 41) — `formAction` lifts the boilerplate.
- **`bind:value` function-form** (Ch 18 preview, formal here) — normalisation at the input edge.
- **ARIA / autocomplete** (Ch 4, 8) — every form continues the accessibility discipline.
- **Snapshots** — SvelteKit's built-in tool for preserving in-page state across navigation.

---

## Lesson 53.7 — What you can now read in the wild

After Chapter 53 you can:

- Read **`formAction(schema, handler)`** as a typed wrapper that lifts validation.
- Read **`bind:value={() => x, (v) => x = transform(v)}`** as the function-binding form.
- Read **`aria-invalid` + `aria-describedby` + `id="email-error"`** as the ARIA error-wiring pattern.
- Read **`export const snapshot = { capture, restore }`** in a `+page.svelte`.

---

## Lesson 53.8 — Read this code

**Snippet 1.** Will the form below block submission with an empty email field if JavaScript is disabled in the browser?

```svelte
<form method="POST" action="?/signup">
  <label for="email">Email</label>
  <input id="email" name="email" type="email" required bind:value={user.email} />
  <button type="submit">Sign up</button>
</form>
```

*Answer.* **Yes.** Native HTML5 form validation works without JavaScript. The `required` attribute is enforced by the browser before the form posts; an empty value pops the browser's native "Please fill out this field" tooltip and the POST never leaves the page. `bind:value` is a Svelte enhancement layered on top — when JS is off, it's inert, but `name="email"` still sends the value as form data and `required` still gates submission. This is the whole point of progressive enhancement: the form works at every layer of capability. The lesson: every input that's required server-side should also have `required` client-side; you get free a11y-grade error UX without writing a line of JS.

**Snippet 2.** What renders, and what's the bug?

```svelte
<script lang="ts">
  let { form } = $props();
</script>

<input name="age" aria-invalid={form?.fieldErrors?.age ? 'true' : 'false'} />
{#if form?.fieldErrors?.age}
  <p>{form.fieldErrors.age}</p>
{/if}
```

*Answer.* The bug is **`aria-invalid="false"` instead of `undefined`** when there's no error. ARIA semantics: `aria-invalid="false"` *explicitly* declares the field as valid, which screen readers may announce ("entry is valid"). For an untouched, never-validated field, that's a lie — we don't know if it's valid yet. Use `aria-invalid={form?.fieldErrors?.age ? 'true' : undefined}` so the attribute is omitted (= "no judgement made") rather than asserting validity. Also missing: `aria-describedby` linking the error `<p id="age-error">` to the input. Compare against Lesson 53.3 — the snippet above is what people *almost* write; the ARIA-correct version is what you ship.

---

## Lesson 53.9 — Now you write it

**Task.** Build a `<NumberInput>` component with native HTML5 validation (`min`, `max`, `step`) plus `aria-invalid` wiring that flips on when the value is out of range. Bonus: render an error `<p>` linked via `aria-describedby` when the value is invalid.

*Worked answer.*

```svelte
<!-- src/lib/components/NumberInput.svelte -->
<script lang="ts">
  let {
    value = $bindable(0),
    min,
    max,
    step = 1,
    label,
    name,
  }: {
    value?: number;
    min: number;
    max: number;
    step?: number;
    label: string;
    name: string;
  } = $props();

  const id = $derived(`num-${name}`);
  const errorId = $derived(`${id}-error`);
  const outOfRange = $derived(value < min || value > max);
  const errorText = $derived(outOfRange ? `Must be between ${min} and ${max}.` : null);
</script>

<label for={id}>{label}</label>
<input
  {id}
  {name}
  type="number"
  {min}
  {max}
  {step}
  bind:value
  required
  aria-invalid={errorText !== null ? 'true' : undefined}
  aria-describedby={errorText !== null ? errorId : undefined}
/>
{#if errorText !== null}
  <p id={errorId} class="error">{errorText}</p>
{/if}
```

Key moves: (1) `bind:value` with `$bindable` so a parent can two-way bind; (2) `aria-invalid` only set when there's a real error (not `'false'` for valid); (3) `aria-describedby` linked to a unique error ID derived from `name`; (4) the native `min`/`max`/`step` attributes do the heavy lifting — the `outOfRange` derivation is just for our visual + screen-reader error. With JS off, the browser's native validation still catches out-of-range submits.

---

## End-of-chapter checkpoint

- [ ] `formAction` is used by every form in the app.
- [ ] All forms have ARIA error wiring.
- [ ] Multi-step form has snapshot.

---

# Chapter 54 — SEO, structured data, OpenGraph, sitemaps

> *Today's job:* paste any marketing URL into Slack — see a rich preview. Visible win: `/sitemap.xml` validates; Google Rich Results passes.

---

## Lesson 54.1 — `<svelte:head>` with the `SEO` component

```svelte
<!-- src/lib/components/SEO.svelte -->
<script lang="ts">
  import { page } from '$app/state';
  let {
    title,
    description,
    image = '/og-default.png',
    type = 'website',
  }: { title: string; description: string; image?: string; type?: 'website' | 'article' } = $props();

  const fullTitle = $derived(`${title} | Streak`);
  const url = $derived(page.url.href);
</script>

<svelte:head>
  <title>{fullTitle}</title>
  <meta name="description" content={description} />

  <meta property="og:title" content={fullTitle} />
  <meta property="og:description" content={description} />
  <meta property="og:image" content={image} />
  <meta property="og:type" content={type} />
  <meta property="og:url" content={url} />

  <meta name="twitter:card" content="summary_large_image" />
  <meta name="twitter:title" content={fullTitle} />
  <meta name="twitter:description" content={description} />
  <meta name="twitter:image" content={image} />

  <link rel="canonical" href={url} />
</svelte:head>
```

Used per page:

```svelte
<SEO title="Pricing" description="Streak Pro at $9/month — habits that stick." image="/og/pricing.png" />
```

> **Create `static/og-default.png`** (1200×630 PNG, the OpenGraph reference resolution). The default image is referenced from the `<SEO>` component; without the file, social previews show a broken image when a page doesn't pass an explicit `image` prop. Ship the default at the start, override per-page where you have something better.

---

## Lesson 54.2 — Sitemap as `+server.ts`

```ts
// src/routes/sitemap.xml/+server.ts
import type { RequestHandler } from './$types';

const URLS = ['/', '/about', '/pricing'];

export const GET: RequestHandler = ({ url }) => {
  const xml = `<?xml version="1.0" encoding="UTF-8"?>
<urlset xmlns="http://www.sitemaps.org/schemas/sitemap/0.9">
  ${URLS.map((u) => `<url><loc>${url.origin}${u}</loc></url>`).join('\n  ')}
</urlset>`;
  return new Response(xml, { headers: { 'content-type': 'application/xml' } });
};
```

---

## Lesson 54.3 — Robots

```ts
// src/routes/robots.txt/+server.ts
import type { RequestHandler } from './$types';

export const GET: RequestHandler = ({ url }) => {
  const body = `User-agent: *
Allow: /
Disallow: /app/
Disallow: /admin/
Sitemap: ${url.origin}/sitemap.xml
`;
  return new Response(body, { headers: { 'content-type': 'text/plain' } });
};
```

---

## Lesson 54.4 — JSON-LD structured data

```svelte
<script lang="ts">
  // Build the JSON-LD payload, then escape every character that could break out
  // of an inline <script> or be misread by an HTML parser. Five replacements:
  //  - `<`  → `<` : prevents `</script>` from closing the tag prematurely.
  //  - `>`  → `>` : symmetry; protects against `]]>` / lazy CDATA parsers.
  //  - `&`  → `&` : prevents HTML entity injection (e.g. `&lt;` re-decoded).
  //  - U+2028 → ` ` : JS treats line-separator as a newline; breaks JSON.
  //  - U+2029 → ` ` : same for paragraph-separator.
  const ldJson = JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: 'Streak Pro',
    offers: { '@type': 'Offer', price: '9.00', priceCurrency: 'USD' },
  })
    .replace(/</g, '\\u003c')
    .replace(/>/g, '\\u003e')
    .replace(/&/g, '\\u0026')
    .replace(/ /g, '\\u2028')
    .replace(/ /g, '\\u2029');
</script>

<svelte:head>
  {@html `<script type="application/ld+json">${ldJson}</script>`}
</svelte:head>
```

Why all five: even if the JSON content is *our own* today (no user input, no injection concern *yet*), the moment a user-supplied product name lands in this payload — a SKU description, a referral note, a partner-supplied OG title — a single `</script>` or stray `&lt;` smuggled in would break out of the tag. The five replacements are the standard hardening for inline JSON in HTML; copy them as a unit, never just `<`. Senior habit.

---

## Lesson 54.5 — `prerender = true`

Marketing pages:

```ts
// (marketing)/+layout.ts
export const prerender = true;
```

At build time these become static HTML. Fastest, free CDN caching, perfect SEO.

---

## Lesson 54.6 — Recurring concepts from earlier chapters

- **`+server.ts`** (Ch 33) — sitemap and robots are typed JSON/text endpoints.
- **`prerender = true`** — page-options export, applied to the marketing layout for the whole subtree.
- **`<svelte:head>`** — dynamic `<head>` content per page.
- **`@html` with explicit escaping** — JSON-LD safety pattern.

---

## Lesson 54.7 — What you can now read in the wild

After Chapter 54 you can:

- Read **`<SEO title description image>`** as a per-page metadata snippet.
- Read **`/sitemap.xml/+server.ts`** and **`/robots.txt/+server.ts`** as text-shaped endpoints.
- Read **JSON-LD with `</` escape** as the safe-by-construction pattern.
- Read **`export const prerender = true`** at the layout level and trace which routes inherit it.

---

## Lesson 54.8 — Read this code

**Snippet 1.** A layout sets a `<title>` and a page sets a different `<title>`. Which one wins?

```svelte
<!-- src/routes/+layout.svelte -->
<svelte:head>
  <title>Streak — habits that stick</title>
</svelte:head>

<!-- src/routes/pricing/+page.svelte -->
<svelte:head>
  <title>Pricing | Streak</title>
</svelte:head>
```

*Answer.* **The page wins** for `<title>` (and for any `<svelte:head>` element keyed by tag name, like `<title>`, plus elements with the same `name` for `<meta>`). SvelteKit's `<svelte:head>` follows a "page-overrides-layout" rule for duplicate keys — exactly what you want, because the most specific component (the page) usually has the most accurate metadata. Order of execution in the rendered HTML reflects this: layout `<head>` content is emitted first, then the page's, and browsers honour the *last* `<title>` they see. The lesson: it's safe to set a generic `<title>` in your root layout as a fallback; pages that override it just work.

**Snippet 2.** What URL does this `og:image` resolve to when crawled by Slack/Twitter from `https://streak.app/blog/post-1`?

```svelte
<svelte:head>
  <meta property="og:image" content="og/post-1.png" />
</svelte:head>
```

*Answer.* **It depends — and that's the bug.** A relative URL like `og/post-1.png` is resolved against the page's URL, so against `https://streak.app/blog/post-1` it would resolve to `https://streak.app/blog/og/post-1.png` — almost certainly a 404 unless you happen to have that path. Worse: many social-media crawlers follow stricter rules than browsers and refuse to resolve relative URLs at all, treating the field as missing. **Always use absolute URLs for `og:image`** (and for `og:url`, `twitter:image`, and `<link rel="canonical">`): either a leading `/og/post-1.png` (root-relative) for crawlers that handle it, or, safer, the fully-qualified `https://streak.app/og/post-1.png` built from `page.url.origin`. The `<SEO>` component in Lesson 54.1 takes the easy path — relative `image="/og-default.png"` — which is fine for *most* crawlers; harden by prefixing `page.url.origin` if you see broken previews.

---

## End-of-chapter checkpoint

- [ ] `<SEO>` is used on every marketing page.
- [ ] `/sitemap.xml`, `/robots.txt` work.
- [ ] Google Rich Results Test passes the pricing page.

---

# Chapter 55 — Service workers, offline, prerendering, performance

> *Today's job:* offline support and Lighthouse 100s. Visible win: turn off Wi-Fi after first load — habits still render.

---

## Lesson 55.1 — `src/service-worker.ts`

```ts
// src/service-worker.ts
/// <reference types="@sveltejs/kit" />
/// <reference no-default-lib="true"/>
/// <reference lib="esnext" />
/// <reference lib="webworker" />

import { build, files, version } from '$service-worker';

declare const self: ServiceWorkerGlobalScope;

const CACHE = `streak-${version}`;
const ASSETS = [...build, ...files];

self.addEventListener('install', (e) => {
  e.waitUntil(caches.open(CACHE).then((c) => c.addAll(ASSETS)));
});

self.addEventListener('activate', (e) => {
  e.waitUntil(
    caches.keys().then((keys) => Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)))),
  );
});

self.addEventListener('fetch', (e) => {
  if (e.request.method !== 'GET') {
    return;
  }
  e.respondWith(
    caches.match(e.request).then((cached) => cached ?? fetch(e.request)),
  );
});
```

The `declare const self: ServiceWorkerGlobalScope;` at the top, paired with the `/// <reference lib="webworker" />` directive, gives TypeScript the right global types — `e: ExtendableEvent` for `install`/`activate`, `e: FetchEvent` for `fetch` — without any `as` casts. Bible rule: `as` is a smell; type the global instead.

**Cache TTL — why we get away with cache-first.** The handler above is *cache-first with no TTL* — once an asset is in the cache, it stays there until `activate` evicts the old `CACHE` key. That sounds dangerous (stale forever) but it's safe here for one specific reason: the cache key is `streak-${version}` and `version` changes on every deploy. New deploy → new cache key → `activate` deletes the old cache → fresh fetches. The strategy is "rebuild the SW on every deploy" not "expire entries." For marketing pages this is ideal: instant offline render, no per-asset TTL bookkeeping, predictable invalidation tied to your deploy cadence. **For user data, never cache** — if you reach `/api/habits` from inside the SW, return `fetch(e.request)` unconditionally. If you ever need a finer-grained policy (e.g. "cache marketing for 1 hour even within the same deploy"), upgrade to **stale-while-revalidate**:

```ts
self.addEventListener('fetch', (e) => {
  if (e.request.method !== 'GET') {
    return;
  }
  e.respondWith(
    caches.match(e.request).then((cached) => {
      const network = fetch(e.request).then((res) => {
        const copy = res.clone();
        caches.open(CACHE).then((c) => c.put(e.request, copy));
        return res;
      });
      if (cached !== undefined) {
        e.waitUntil(network);
        return cached;
      }
      return network;
    }),
  );
});
```

That gives instant cached response *and* a background refresh for next time. The trade-off: you serve one stale response per refresh window. For a fintech-shaped flow this is a non-starter on data routes; for marketing it's fine.

---

## Lesson 55.2 — Link preloading

The `data-sveltekit-preload-*` attributes go on the **`<body>` tag in `src/app.html`** (or any element above your routed content). They *cannot* go inside `<svelte:head>` (which only contains head children).

```html
<!-- src/app.html -->
<body data-sveltekit-preload-data="hover" data-sveltekit-preload-code="hover">
  <div style="display: contents">%sveltekit.body%</div>
</body>
```

Hover a link, SvelteKit preloads its code+data. Click feels instant.

You can also override per-link with `<a href="..." data-sveltekit-preload-data="off">` for routes that shouldn't auto-preload (heavy pages, paginated lists).

---

## Lesson 55.3 — Lighthouse run

First add the dev dependency:

```bash
pnpm add -D lighthouse
```

Then run against a production build:

```bash
pnpm build && pnpm preview
pnpm exec lighthouse http://localhost:4173/ --view
```

(Or use Chrome dev tools' Lighthouse panel against `http://localhost:5173/` in dev — note dev mode reports lower scores because Vite ships unminified.) Aim for 100/100/100/100 on production builds. Fix what's red. The marketing pages should hit it easily; the app pages have a lower ceiling because of auth and JS-heavy interactivity.

> **CI note.** `lighthouse` needs Chrome installed; on CI you can either pull in the `puppeteer` runtime (which ships its own headless Chromium) or skip Lighthouse there entirely and run it locally before each release. We do the latter for Streak — Lighthouse on a slow GitHub Actions runner produces noisy scores; a fast laptop run pre-deploy is the truer signal.

---

## Lesson 55.4 — Recurring concepts from earlier chapters

Part VIII's spine, in one place:

- **Stripe + idempotency + Money** (Ch 49–50).
- **R2 presigned uploads + boundary parsing** (Ch 51).
- **Every Svelte motion primitive** (Ch 52).
- **`formAction` + ARIA + snapshots** (Ch 53).
- **SEO with `<svelte:head>` + sitemap + robots + JSON-LD** (Ch 54).
- **Service workers + `prerender` + link preloading** (Ch 55).

By the end of Part VIII, Streak is a polished, paid, animated, indexable, offline-capable product.

---

## Lesson 55.5 — What you can now read in the wild

After Part VIII you can:

- Read a **service worker** and explain install/activate/fetch lifecycles.
- Read **`$service-worker` exports** (`build`, `files`, `version`, `base`).
- Read **`data-sveltekit-preload-*`** attributes and tune per-app.
- Read **`export const prerender / ssr / csr`** page options and pick correctly.
- Run a **Lighthouse audit** against a production build and fix what's red.

---

## End-of-chapter checkpoint

- [ ] Offline mode shows cached habits.
- [ ] Lighthouse on `/` is 100/100/100/100.

End of Part VIII. Next: shipping it.
