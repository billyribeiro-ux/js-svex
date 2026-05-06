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
  apiVersion: '2026-04-30',
  maxNetworkRetries: 2,
  timeout: 10_000, // 10s — Bible rule
});
```

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

    let [sub] = await db.select().from(subscriptions).where(eq(subscriptions.userId, user.id)).limit(1);

    let stripeCustomerId = sub?.stripeCustomerId;
    if (!stripeCustomerId) {
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
      { idempotencyKey: `checkout:${user.id}:${Date.now()}` },
    );

    redirect(303, session.url!);
  },
};
```

`idempotencyKey` on every mutating Stripe call. Bible rule.

---

## Lesson 49.4 — Reading: the `Money` module in production

The `formatCents`, `applyBps`, `splitCents` helpers from Chapter 30 are now imported by:
- the pricing page (display the Pro tier price),
- the billing settings page (show "next charge: $9.99 on YYYY-MM-DD"),
- the webhook handler (Ch 50, where amounts arrive).

The compounding-fluency spine the plan promised.

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
import { stripe } from '$lib/stripe';
import { db } from '$lib/db/client';
import { subscriptions, webhookEvents } from '$lib/db/schema';
import { eq } from 'drizzle-orm';
import { STRIPE_WEBHOOK_SECRET } from '$env/static/private';

export const POST: RequestHandler = async ({ request }) => {
  const sig = request.headers.get('stripe-signature');
  if (!sig) error(400, 'No signature');

  const raw = await request.text();
  let event;
  try {
    event = stripe.webhooks.constructEvent(raw, sig, STRIPE_WEBHOOK_SECRET);
  } catch {
    error(400, 'Invalid signature');
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
      const customerId = session.customer as string;
      const subscriptionId = session.subscription as string;

      // re-fetch canonical state (defence in depth)
      const sub = await stripe.subscriptions.retrieve(subscriptionId);

      await tx.update(subscriptions)
        .set({
          stripeSubscriptionId: sub.id,
          plan: 'pro',
          status: sub.status,
          currentPeriodEnd: new Date(sub.current_period_end * 1000),
        })
        .where(eq(subscriptions.stripeCustomerId, customerId));
    }

    if (event.type === 'customer.subscription.deleted') {
      const sub = event.data.object;
      await tx.update(subscriptions)
        .set({ plan: 'free', status: 'cancelled' })
        .where(eq(subscriptions.stripeCustomerId, sub.customer as string));
    }

    await tx.update(webhookEvents)
      .set({ processedAt: new Date() })
      .where(eq(webhookEvents.id, event.id));
  });

  return text('ok');
};
```

---

## Lesson 50.3 — Why we re-fetch

We don't trust the webhook payload's `current_period_end` — it could be stale or the event could be replayed in a weird order. We refetch the canonical state from Stripe. Defence in depth.

---

## Lesson 50.4 — `pnpm test:webhook:duplicate`

```ts
// tests/integration/stripe-webhook.dedupe.test.ts
import { describe, it, expect } from 'vitest';

describe('stripe webhook dedup', () => {
  it('processes the same event id only once', async () => {
    const eventBody = /* fixture */;
    const sig = /* compute */;
    const r1 = await fetch('/api/stripe/webhook', { method: 'POST', body: eventBody, headers: { 'stripe-signature': sig } });
    const r2 = await fetch('/api/stripe/webhook', { method: 'POST', body: eventBody, headers: { 'stripe-signature': sig } });
    expect(await r1.text()).toBe('ok');
    expect(await r2.text()).toBe('already processed');
  });
});
```

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
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { R2_ACCOUNT_ID, R2_ACCESS_KEY_ID, R2_SECRET_ACCESS_KEY, R2_BUCKET } from '$env/static/private';
import { requireUser } from '$lib/auth';

const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp', 'image/avif'];
const MAX_BYTES = 5 * 1024 * 1024;

const s3 = new S3Client({
  region: 'auto',
  endpoint: `https://${R2_ACCOUNT_ID}.r2.cloudflarestorage.com`,
  credentials: { accessKeyId: R2_ACCESS_KEY_ID, secretAccessKey: R2_SECRET_ACCESS_KEY },
  requestHandler: { connectionTimeout: 5_000, requestTimeout: 10_000 },
});

export const POST: RequestHandler = async (event) => {
  const user = requireUser(event);
  const { contentType, size } = await event.request.json();

  if (!ALLOWED_MIME.includes(contentType)) error(400, 'Unsupported MIME');
  if (typeof size !== 'number' || size > MAX_BYTES) error(400, 'File too large');

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

The client then `fetch(url, { method: 'PUT', body: file })` directly to R2.

---

## Lesson 51.3 — Magic-number MIME validation

After upload, on a confirmation endpoint, fetch the `HEAD` and read the first bytes — confirm the content-type matches the declared MIME. Don't trust the client's claim.

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

## Lesson 52.3 — `svelte/motion` — `tweened`, `spring`

```svelte
<script lang="ts">
  import { spring } from 'svelte/motion';
  let count = $state(0);
  const display = spring(0, { stiffness: 0.1, damping: 0.4 });
  $effect(() => display.set(count));
</script>

<button onclick={() => count += 1}>+1</button>
<p>{$display.toFixed(0)}</p>
```

The number bounces toward its target.

---

## Lesson 52.4 — `prefers-reduced-motion`

```ts
// src/lib/motion.svelte.ts
let _reduced = $state(typeof window === 'undefined' ? false : window.matchMedia('(prefers-reduced-motion: reduce)').matches);
if (typeof window !== 'undefined') {
  window.matchMedia('(prefers-reduced-motion: reduce)').addEventListener('change', (e) => _reduced = e.matches);
}
export const prefersReducedMotion = { get value() { return _reduced; } };
```

In components:

```svelte
<script lang="ts">
  import { prefersReducedMotion } from '$lib/motion.svelte';
</script>

{#each items as item (item.id)}
  <li transition:fly={prefersReducedMotion.value ? { duration: 0 } : { y: 10, duration: 200 }}>{item.name}</li>
{/each}
```

Senior habit. WCAG-grade.

---

## Lesson 52.5 — `use:` actions

```ts
// src/lib/actions/intersect.ts
export function intersect(node: HTMLElement, callback: () => void): { destroy: () => void } {
  const obs = new IntersectionObserver((entries) => {
    if (entries[0]?.isIntersecting) callback();
  }, { threshold: 0.1 });
  obs.observe(node);
  return { destroy: () => obs.disconnect() };
}
```

```svelte
<div use:intersect={() => console.log('visible!')}>...</div>
```

---

## Lesson 52.6 — `{@attach}` directive

In Svelte 5.x as of May 2026, `{@attach}` is a successor to actions for some cases:

```svelte
<div {@attach (node) => {
  // setup
  return () => { /* cleanup */ };
}}>
  ...
</div>
```

Use it when the lifetime is bound to render rather than the DOM element being long-lived.

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
import type { RequestEvent } from '@sveltejs/kit';

export function formAction<T>(
  schema: v.GenericSchema<unknown, T>,
  handler: (input: T, event: RequestEvent) => Promise<{ success?: boolean; redirect?: string } | void>,
) {
  return async (event: RequestEvent) => {
    const data = await event.request.formData();
    const obj: Record<string, unknown> = {};
    for (const [k, v] of data.entries()) obj[k] = v;
    const parsed = v.safeParse(schema, obj);
    if (!parsed.success) {
      const fieldErrors: Record<string, string> = {};
      for (const issue of parsed.issues) {
        const key = issue.path?.[0]?.key ?? 'form';
        if (typeof key === 'string') fieldErrors[key] = issue.message;
      }
      return fail(400, { fieldErrors });
    }
    return handler(parsed.output, event);
  };
}
```

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

```svelte
<input bind:value={() => email, (v) => email = v.toLowerCase().trim()} />
```

The `email` value is always normalized as the user types.

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
<svelte:head>
  {@html `<script type="application/ld+json">${JSON.stringify({
    '@context': 'https://schema.org',
    '@type': 'Product',
    name: 'Streak Pro',
    offers: { '@type': 'Offer', price: '9.00', priceCurrency: 'USD' },
  })}</script>`}
</svelte:head>
```

(`{@html}` is needed for the script-tag content; sanitise carefully — here it's all our own text.)

---

## Lesson 54.5 — `prerender = true`

Marketing pages:

```ts
// (marketing)/+layout.ts
export const prerender = true;
```

At build time these become static HTML. Fastest, free CDN caching, perfect SEO.

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

const CACHE = `streak-${version}`;
const ASSETS = [...build, ...files];

self.addEventListener('install', (e) => {
  (e as ExtendableEvent).waitUntil(caches.open(CACHE).then((c) => c.addAll(ASSETS)));
});

self.addEventListener('activate', (e) => {
  (e as ExtendableEvent).waitUntil(
    caches.keys().then((keys) => Promise.all(keys.filter((k) => k !== CACHE).map((k) => caches.delete(k)))),
  );
});

self.addEventListener('fetch', (event) => {
  const e = event as FetchEvent;
  if (e.request.method !== 'GET') return;
  e.respondWith(
    caches.match(e.request).then((cached) => cached ?? fetch(e.request)),
  );
});
```

---

## Lesson 55.2 — Link preloading

```svelte
<!-- in +layout.svelte -->
<svelte:head>
  <body data-sveltekit-preload-data="hover" data-sveltekit-preload-code="hover">
</svelte:head>
```

Hover a link, SvelteKit preloads its code+data. Click feels instant.

---

## Lesson 55.3 — Lighthouse run

```bash
pnpm exec lighthouse https://localhost:5173/ --view
```

(Or use Chrome dev tools' Lighthouse panel.) Aim for 100/100/100/100. Fix what's red. The marketing pages should hit it easily; the app pages have a lower ceiling because of auth.

---

## End-of-chapter checkpoint

- [ ] Offline mode shows cached habits.
- [ ] Lighthouse on `/` is 100/100/100/100.

End of Part VIII. Next: shipping it.
