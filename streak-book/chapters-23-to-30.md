# Part IV — TypeScript as a tool, not a tax

> *"By Chapter 30 the IDE catches bugs before the browser ever sees them. The reader writes `unknown` instead of `any`, `Result` instead of throw, branded IDs instead of bare strings. They have built `parseHabit`, `Result`, `HabitStore`, and the `Money` module — every later part imports them unmodified."*

---

# Chapter 23 — Discriminated unions, formally

> *Today's job:* `RequestState<T>` becomes a real type with exhaustiveness checking. Visible win: delete a case in a switch; red squiggle, useful message.

---

## Lesson 23.1 — The discriminator

A **discriminated union** is a union type where each variant has a *common field* (the *discriminator*) whose value identifies which variant you're looking at.

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };
```

The discriminator is `status`. TypeScript narrows the type when you check it:

```ts
function describe(state: RequestState<Habit[]>): string {
  if (state.status === 'success') {
    return `${state.data.length} habits.`; // .data is available here
  }
  if (state.status === 'error') {
    return `Error: ${state.message}`; // .message is available here
  }
  return state.status; // narrowed to 'idle' | 'loading' — both literal strings, returned as-is
}
```

Read aloud: *"if the status is success, return the habit count. If it's error, return the error message. Otherwise return the status string itself ('idle' or 'loading')."*

> **discriminated union** — a union of object types sharing a common literal field that lets the compiler tell them apart.
>
> **narrowing** — what TypeScript does when a check (`status === 'success'`) lets it deduce the variant.

---

## Lesson 23.2 — Exhaustiveness with `never`

We want to catch the case where someone adds a fifth state (say, `'cancelled'`) and forgets to handle it. Add a `default` branch that asserts:

```ts
function describe(state: RequestState<Habit[]>): string {
  switch (state.status) {
    case 'idle': return 'Not started.';
    case 'loading': return 'Loading...';
    case 'success': return `${state.data.length} habits.`;
    case 'error': return `Error: ${state.message}`;
    default: return assertNever(state);
  }
}

function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}
```

If every case is covered, `state` narrows to `never` in the default branch; `assertNever` accepts. If you add a fifth variant and don't add a `case`, the new variant *isn't* `never`; `assertNever`'s parameter type fails to match; TypeScript flags it. **Compile-time exhaustiveness.**

> **`never`** — the type with no values. Used to express "this code is unreachable."

---

## Lesson 23.3 — Wiring it into Streak

Move the type to `src/lib/types.ts`:

```ts
// src/lib/types.ts
export type Habit = {
  id: string;
  name: string;
  description?: string;
  createdAt: number;
};

export type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };

export function assertNever(value: never): never {
  throw new Error(`Unhandled variant: ${JSON.stringify(value)}`);
}
```

Use it in `+page.svelte`:

```ts
import type { Habit, RequestState } from '$lib/types';
import { assertNever } from '$lib/types';

let saveState: RequestState<void> = $state({ status: 'idle' });
```

And in markup, the switch becomes:

```svelte
{#if saveState.status === 'loading'}
  <small>Saving...</small>
{:else if saveState.status === 'error'}
  <p class="error">{saveState.message}</p>
{/if}
```

---

## Lesson 23.4 — Build, break, fix

Add a fifth state to `RequestState`:

```ts
| { status: 'cancelled' }
```

Save. Watch the `describe` function (or any switch) light up red. The compiler tells you exactly where the new case is missing. Add it. Rebuild.

Then *delete* the variant. The errors disappear. This is the "compiler as tutor" loop.

---

## Lesson 23.5 — Read this code

```ts
type Action =
  | { kind: 'add'; name: string }
  | { kind: 'remove'; id: string }
  | { kind: 'rename'; id: string; newName: string };

function apply(action: Action, habits: Habit[]): Habit[] {
  switch (action.kind) {
    case 'add': return [...habits, /* ... */];
    case 'remove': return habits.filter((h) => h.id !== action.id);
    case 'rename': return habits.map((h) => h.id === action.id ? { ...h, name: action.newName } : h);
  }
}
```

What happens if you add a `'duplicate'` variant?

<details>
<summary>Answer</summary>

The function's return type stops being `Habit[]` because the switch isn't exhaustive — `'duplicate'` falls through with no return. TypeScript flags it. Add `assertNever` to the default to make the failure explicit.
</details>

---

## Lesson 23.6 — Now you write it

**The English sentence first:**

> *"Define a `BillingStatus` discriminated union for Streak's future Pro tier: 'free', 'pending', 'pro' (with a `currentPeriodEnd: number`), 'past_due', 'cancelled'. We'll use this in Chapter 49."*

<details>
<summary>Worked answer</summary>

```ts
export type BillingStatus =
  | { plan: 'free' }
  | { plan: 'pending' }
  | { plan: 'pro'; currentPeriodEnd: number }
  | { plan: 'past_due'; currentPeriodEnd: number }
  | { plan: 'cancelled'; cancelledAt: number };
```

This will appear, unchanged, in Chapter 49.
</details>

---

## Lesson 23.7 — Recurring concepts from earlier chapters

- **Union types** (Ch 4, 14) — `RequestState` is a *discriminated* union.
- **The four-state UI rule** (Ch 22) — formalised here as a real type.
- **`switch` over a discriminator** — natural extension of `if/else if/else`.
- **Type narrowing** (Ch 19) — what the compiler does when you check the discriminator.

---

## Lesson 23.8 — What you can now read in the wild

After Chapter 23 you can:

- Read **`type X = { kind: 'a' } | { kind: 'b'; ... }`** as a discriminated union.
- Read **`switch (x.kind) { case 'a': ...; case 'b': ... }`** with full narrowing.
- Read **`function assertNever(x: never): never`** as the exhaustiveness guard.
- Spot non-exhaustive switches as compile-time bugs.

---

## End-of-chapter checkpoint

- [ ] `RequestState` and `assertNever` live in `$lib/types`.
- [ ] You felt the exhaustiveness check work.
- [ ] You drafted `BillingStatus` for the future.

---

# Chapter 24 — Generics, gently

> *Today's job:* a generic `<List items={...}>{(item) => ...}</List>` that works for any item type. Visible win: one component, multiple uses.

---

## Lesson 24.1 — The generic concept

A **generic** is a type variable — a placeholder for a type that gets fixed at the call site.

```ts
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const x = first([1, 2, 3]);    // T = number; x: number | undefined
const y = first(['a', 'b']);   // T = string; y: string | undefined
```

Read aloud: *"first takes an array of T and returns one T or undefined."*

> **generic** — a type parameter. Reusable across types without losing type safety.

---

## Lesson 24.2 — Generic components

In Svelte 5, components are made generic with `<script lang="ts" generics="T">`:

```svelte
<!-- src/lib/components/List.svelte -->
<script lang="ts" generics="T extends { id: string }">
  import type { Snippet } from 'svelte';

  let {
    items,
    item,
    empty,
  }: {
    items: T[];
    item: Snippet<[T]>;
    empty?: Snippet;
  } = $props();
</script>

{#if items.length === 0}
  {#if empty}{@render empty()}{:else}<p>Nothing here.</p>{/if}
{:else}
  <ul>
    {#each items as i (i.id)}
      <li>{@render item(i)}</li>
    {/each}
  </ul>
{/if}
```

The `<T extends { id: string }>` *constrains* T to types with an `id: string` field. We need this for the keyed `{#each}`.

Use it:

```svelte
<List items={visibleHabits}>
  {#snippet item(h)}
    <HabitRow habit={h} onDelete={removeHabit} />
  {/snippet}
</List>
```

The compiler infers `T = Habit` from the call site. `h` inside the snippet is typed `Habit`.

---

## Lesson 24.3 — Constraints

`<T extends Foo>` means "T must be a subtype of Foo". Common forms:

- `<T extends string>` — T is a string-ish thing.
- `<T extends { id: string }>` — T has at least an `id`.
- `<T, K extends keyof T>` — K is one of T's property names.

---

## Lesson 24.4 — Read this code

```ts
function pluck<T, K extends keyof T>(items: T[], key: K): T[K][] {
  return items.map((i) => i[key]);
}

const names = pluck(habits, 'name'); // names: string[]
const ids = pluck(habits, 'id');     // ids: string[]
```

What's the type of `pluck(habits, 'createdAt')`?

<details>
<summary>Answer</summary>

`number[]` — because `Habit['createdAt']` is `number`. Generics let one function correctly type three different return types depending on the key passed.
</details>

---

## Lesson 24.5 — Now you write it

**The English sentence first:**

> *"Build a generic `groupBy(items, getKey)` that returns a `Record<string, T[]>` — a map from key to items. Use it later for grouping habits by category."*

<details>
<summary>Worked answer</summary>

```ts
export function groupBy<T>(items: T[], getKey: (item: T) => string): Record<string, T[]> {
  const result: Record<string, T[]> = {};
  for (const item of items) {
    const key = getKey(item);
    const bucket = result[key] ?? [];
    bucket.push(item);
    result[key] = bucket;
  }
  return result;
}
```

`Record<string, T[]>` is TypeScript shorthand for "an object whose keys are strings and whose values are `T[]`". 

Why the explicit `bucket` variable? Because under `noUncheckedIndexedAccess`, `result[key]` has type `T[] | undefined` *every* time you read it — even after `??=` would have ensured it's an array. The compiler doesn't track that. Pulling the bucket into a `const` narrows once and lets us `push` without re-reading.
</details>

---

## Lesson 24.6 — Recurring concepts from earlier chapters

- **`Snippet<T>`** (Ch 20) — generic snippet types.
- **Function-parameter destructuring** (Ch 10) — generics work alongside it.
- **`{#each ... as ... (key)}`** (Ch 9) — the `extends { id: string }` constraint exists *because* of the keyed-each requirement.

---

## Lesson 24.7 — What you can now read in the wild

After Chapter 24 you can:

- Read **`function f<T>(x: T)`** and **`<T extends Foo>`**.
- Read **`<script lang="ts" generics="T">`** in a Svelte component.
- Read **`Snippet<[T]>`** and infer T from the call site.
- Read advanced forms like **`<T, K extends keyof T>`** for type-safe property access.

---

## End-of-chapter checkpoint

- [ ] `<List>` works for habits.
- [ ] You can read `<T>`, `<T extends Foo>` aloud.
- [ ] You wrote `groupBy`.

---

# Chapter 25 — `readonly`, `as const`, `satisfies`

> *Today's job:* `HABIT_CATEGORIES` is a frozen list the compiler treats as the source of truth. Visible win: typing a wrong category name is red in the editor.

---

## Lesson 25.1 — `as const`

Without `as const`:

```ts
const FRUITS = ['apple', 'banana', 'cherry'];
// FRUITS: string[]
```

With:

```ts
const FRUITS = ['apple', 'banana', 'cherry'] as const;
// FRUITS: readonly ['apple', 'banana', 'cherry']
```

`as const` narrows literals to their *exact* values and marks the array as `readonly`. Now you can derive a type:

```ts
type Fruit = (typeof FRUITS)[number]; // 'apple' | 'banana' | 'cherry'
```

> **`as const`** — TypeScript: keep literals as their exact values, freeze structures.
>
> **`(typeof X)[number]`** — type extraction: "the union of all element types of X".

---

## Lesson 25.2 — `readonly`

Mark types `readonly` for inputs you don't mutate:

```ts
function summarise(fruits: readonly string[]): string {
  return fruits.join(', ');
}
```

Inside `summarise`, calling `fruits.push(...)` is a compile error. Senior habit: function parameters should be `readonly` unless they're genuinely meant to be mutated by the function.

---

## Lesson 25.3 — `satisfies`

```ts
const ROUTES = {
  home: '/',
  pricing: '/pricing',
  app: '/app',
} satisfies Record<string, string>;
```

`satisfies` checks that the value matches the type *without widening*. Compare:

```ts
const A: Record<string, string> = { home: '/' };
A.home // string

const B = { home: '/' } satisfies Record<string, string>;
B.home // '/' (literal type preserved)
```

`satisfies` keeps the *concrete* shape; `: Type` widens.

> **`satisfies`** — TypeScript: assert that a value conforms to a type without losing the value's specific type.

---

## Lesson 25.4 — Wiring it in

In `src/lib/types.ts`:

```ts
export const HABIT_CATEGORIES = [
  'health',
  'learning',
  'fitness',
  'mindfulness',
  'creativity',
  'social',
] as const;

export type HabitCategory = (typeof HABIT_CATEGORIES)[number];

export type Habit = {
  id: string;
  name: string;
  description?: string;
  category?: HabitCategory;
  createdAt: number;
};
```

Now `category` can only be one of those six strings. Try setting `category: 'travel'` — the compiler refuses. Compile-time guarantees with zero runtime overhead.

---

## Lesson 25.5 — Read this code

```ts
const HTTP_STATUSES = {
  ok: 200,
  notFound: 404,
  serverError: 500,
} as const satisfies Record<string, number>;

type StatusName = keyof typeof HTTP_STATUSES; // 'ok' | 'notFound' | 'serverError'
type StatusCode = (typeof HTTP_STATUSES)[StatusName]; // 200 | 404 | 500
```

What's `HTTP_STATUSES.ok`'s type?

<details>
<summary>Answer</summary>

`200` — the literal, not just `number`. Thanks to `as const`. Without it, it'd be `number`.
</details>

---

## Lesson 25.6 — Now you write it

**The English sentence first:**

> *"Define `BILLING_PLANS` as `as const` with `'free' | 'pro' | 'team'` and derive a `BillingPlan` type. Add a `plan: BillingPlan` field to a future `User` type."*

---

## Lesson 25.7 — Recurring concepts from earlier chapters

- **String union types** (Ch 14) — `'primary' | 'secondary'` is the same idea, derived automatically.
- **Type-shared in `$lib/types`** (Ch 14) — `HABIT_CATEGORIES` and `HabitCategory` live alongside `Habit`.

---

## Lesson 25.8 — What you can now read in the wild

After Chapter 25 you can:

- Read **`as const`** on object/array literals and explain the literal narrowing.
- Read **`readonly T[]`** in function signatures and know it's a no-mutation guarantee.
- Read **`obj satisfies Type`** vs **`obj: Type`** and pick correctly.
- Read **`(typeof X)[number]`** and **`keyof typeof X`** as type-extraction patterns.

---

## End-of-chapter checkpoint

- [ ] `HABIT_CATEGORIES` is frozen and the type is derived.
- [ ] `as const`, `readonly`, `satisfies` each have a clear use case in your head.

---

# Chapter 26 — Parsing the boundary — `unknown`, type guards, `parseHabit`

> *Today's job:* `parseHabit(input: unknown): Habit | null` — used wherever data crosses into the program. Visible win: paste any JSON in the dev console; get a typed habit or a clean rejection.

---

## Lesson 26.1 — The boundary rule

Anything that comes from outside the program is `unknown`:

- JSON from a server.
- Form input.
- `localStorage` reads.
- URL params.
- Environment variables.

We *never* `as Habit` it. We *parse* it.

> **boundary parser** — a function that takes `unknown` and returns either a typed value or a rejection. Lives at every input edge.

---

## Lesson 26.2 — `unknown` vs `any`

`any` opts out of type checking. `unknown` keeps the safety:

```ts
function process(input: unknown) {
  // input.foo // ❌ — unknown has no .foo
  if (typeof input === 'object' && input !== null && 'foo' in input) {
    // narrowed; you can now access input.foo carefully
  }
}
```

Always `unknown`, never `any`. Bible rule #3.

---

## Lesson 26.3 — Built-in narrowing

```ts
typeof x === 'string'
typeof x === 'number'
typeof x === 'boolean'
Array.isArray(x)
x === null
x instanceof Date
'foo' in obj // narrows obj to one with a foo
```

After any of these checks, TypeScript narrows.

---

## Lesson 26.4 — `parseHabit`

Create `src/lib/parseHabit.ts`:

```ts
// src/lib/parseHabit.ts
import type { Habit, HabitCategory } from '$lib/types';
import { HABIT_CATEGORIES } from '$lib/types';

export function parseHabit(input: unknown): Habit | null {
  if (typeof input !== 'object' || input === null) return null;
  const obj = input as Record<string, unknown>;

  if (typeof obj.id !== 'string') return null;
  if (typeof obj.name !== 'string' || obj.name.trim() === '') return null;
  if (typeof obj.createdAt !== 'number' || !Number.isFinite(obj.createdAt)) return null;

  const description: string | undefined =
    typeof obj.description === 'string' ? obj.description : undefined;

  const category: HabitCategory | undefined =
    typeof obj.category === 'string' && (HABIT_CATEGORIES as readonly string[]).includes(obj.category)
      ? (obj.category as HabitCategory)
      : undefined;

  return {
    id: obj.id,
    name: obj.name,
    createdAt: obj.createdAt,
    description,
    category,
  };
}
```

Read aloud, line by line: *"if input isn't an object, reject. Cast to a record of unknowns. Check id is a non-empty string. Check name. Check createdAt. Optionally parse description. Optionally parse category against the frozen list. Return the typed result."*

---

## Lesson 26.5 — Custom type guards

You can also write a *type predicate* function:

```ts
function isHabit(value: unknown): value is Habit {
  return parseHabit(value) !== null;
}
```

The `value is Habit` return type tells TypeScript: *"if this returns true, treat `value` as `Habit` afterward."*

> **type predicate** — a return type of the form `param is T`, asserting the parameter is `T` if the function returns true.

---

## Lesson 26.6 — Read this code

```ts
function parsePort(input: unknown): number | null {
  if (typeof input === 'number' && Number.isInteger(input) && input > 0 && input < 65536) {
    return input;
  }
  if (typeof input === 'string') {
    const n = Number(input);
    if (Number.isInteger(n) && n > 0 && n < 65536) return n;
  }
  return null;
}
```

What inputs are accepted?

<details>
<summary>Answer</summary>

- A positive integer between 1 and 65535 — accepted.
- A string that parses to such an integer (e.g. `"3000"`) — accepted.
- Anything else — rejected.

Tight boundary parser. The senior way to handle environment variables and URL params.
</details>

---

## Lesson 26.7 — Now you write it

**The English sentence first:**

> *"Write `parseHabits(input: unknown): Habit[]` that accepts an array of unknowns, parses each one, and returns the array of those that parse. Bad entries are silently dropped (we'll add proper error reporting later)."*

<details>
<summary>Worked answer</summary>

```ts
export function parseHabits(input: unknown): Habit[] {
  if (!Array.isArray(input)) return [];
  const out: Habit[] = [];
  for (const item of input) {
    const parsed = parseHabit(item);
    if (parsed !== null) out.push(parsed);
  }
  return out;
}
```
</details>

---

## Lesson 26.8 — Recurring concepts from earlier chapters

- **Optional fields `?`** (Ch 9) — described in `parseHabit`'s description and category.
- **`as const` lookup** (Ch 25) — `HABIT_CATEGORIES.includes(...)` validates against the frozen list.
- **Guard clauses** (Ch 2) — every check returns null on failure; the happy path is at the bottom.

---

## Lesson 26.9 — What you can now read in the wild

After Chapter 26 you can:

- Read **`function f(x: unknown)`** and explain why we don't write `any`.
- Read **`if (typeof x === 'string')`**, **`Array.isArray(x)`**, **`x instanceof Date`** as narrowing predicates.
- Read **`function isFoo(x: unknown): x is Foo`** as a custom type guard.
- Spot a missing boundary parser as a security/correctness gap.

---

## End-of-chapter checkpoint

- [ ] `parseHabit.ts` exists and is exported.
- [ ] You can read `unknown`, `is`-predicate, `Array.isArray` aloud.

---

# Chapter 27 — Branded types and `Result<T, E>`

> *Today's job:* `HabitId` and `UserId` are different types even though both are strings. `Result` replaces `throw` for expected failures. Visible win: passing a `UserId` to a function that wants a `HabitId` is a compile error.

---

## Lesson 27.1 — The branding trick

```ts
export type HabitId = string & { readonly __brand: 'HabitId' };
export type UserId = string & { readonly __brand: 'UserId' };

export function habitId(s: string): HabitId {
  return s as HabitId;
}
export function userId(s: string): UserId {
  return s as UserId;
}
```

`string & { __brand: 'HabitId' }` is a string with a *phantom* brand field — a TypeScript-only marker that distinguishes it from a plain string. The brand has no runtime existence.

```ts
function loadHabit(id: HabitId): void { /* ... */ }

const u: UserId = userId('user_1');
loadHabit(u); // ❌ Argument of type 'UserId' is not assignable to parameter of type 'HabitId'.
```

> **branded type** — a type that's structurally a primitive but nominally distinct via a phantom field.

---

## Lesson 27.2 — Wiring it into Streak

In `src/lib/types.ts`:

```ts
export type HabitId = string & { readonly __brand: 'HabitId' };
export type UserId = string & { readonly __brand: 'UserId' };

export function habitId(s: string): HabitId {
  return s as HabitId;
}
export function userId(s: string): UserId {
  return s as UserId;
}

export type Habit = {
  id: HabitId;
  name: string;
  description?: string;
  category?: HabitCategory;
  createdAt: number;
};
```

Update `makeHabit`:

```ts
function makeHabit(name: string): Habit {
  return {
    id: habitId(crypto.randomUUID()),
    name,
    createdAt: Date.now(),
  };
}
```

And every place that takes an `id: string` becomes `id: HabitId`. The compiler walks you through the changes.

---

## Lesson 27.3 — `Result<T, E>`

```ts
export type Result<T, E> = { ok: true; value: T } | { ok: false; error: E };

export function ok<T, E>(value: T): Result<T, E> {
  return { ok: true, value };
}
export function err<T, E>(error: E): Result<T, E> {
  return { ok: false, error };
}
```

Use `Result` for *expected* failures (validation, missing record, payment declined). Reserve `throw` for *unexpected* failures (programmer error).

```ts
function addHabit(name: string): Result<Habit, 'name-empty' | 'name-taken'> {
  const trimmed = name.trim();
  if (trimmed === '') return err('name-empty');
  if (habits.some((h) => h.name === trimmed)) return err('name-taken');
  return ok(makeHabit(trimmed));
}
```

Caller:

```ts
const result = addHabit(input);
if (!result.ok) {
  // result.error is 'name-empty' | 'name-taken'
  return;
}
// result.value is Habit
```

The error type is a union of *exact strings*, not a `string`. The compiler enforces handling each.

---

## Lesson 27.4 — Update `parseHabit`

```ts
export function parseHabit(input: unknown): Result<Habit, string> {
  if (typeof input !== 'object' || input === null) return err('not an object');
  // ... existing checks now return err('reason') instead of null ...
  return ok({ /* habit */ });
}
```

Now callers know *why* parsing failed, not just *that* it did. Senior win.

---

## Lesson 27.5 — Read this code

```ts
function divide(a: number, b: number): Result<number, 'division-by-zero'> {
  if (b === 0) return err('division-by-zero');
  return ok(a / b);
}

const r = divide(10, 0);
if (r.ok) console.log(r.value); else console.log(`error: ${r.error}`);
```

Why is this better than `throw new Error('division by zero')`?

<details>
<summary>Answer</summary>

The function's *return type* tells the caller *exactly* what failures can happen. There's no exception to catch (or forget to catch); the caller is forced by the type system to handle the error case. Compile-time guarantee, no runtime surprise.

`throw` is fine for programmer errors (assertion failures, "this code path shouldn't run"). `Result` is right for things the caller is expected to handle.
</details>

---

## Lesson 27.6 — Now you write it

**The English sentence first:**

> *"Refactor `parseHabit` to return `Result<Habit, ParseError>` where `ParseError = 'not-object' | 'no-id' | 'no-name' | 'invalid-timestamp' | ...`."*

---

## Lesson 27.7 — Recurring concepts from earlier chapters

- **Discriminated unions** (Ch 23) — `Result<T, E>` is exactly one (`{ ok: true; value: T } | { ok: false; error: E }`).
- **String unions for errors** — `'name-empty' | 'name-taken'` instead of `string`.
- **Boundary parser** (Ch 26) — refactored to return `Result` for richer error info.

---

## Lesson 27.8 — What you can now read in the wild

After Chapter 27 you can:

- Read **`type X = string & { readonly __brand: 'X' }`** as a branded primitive.
- Read **`Result<T, E>`** and the `ok()` / `err()` constructors.
- Tell when to **`throw`** (programmer error) vs **return `Result.err`** (caller-handles-this).

---

## End-of-chapter checkpoint

- [ ] `HabitId` and `UserId` are branded; you can no longer mix them.
- [ ] `Result`, `ok`, `err` live in `$lib/types`.
- [ ] `parseHabit` returns `Result`.

---

# Chapter 28 — Classes with `$state` fields

> *Today's job:* a small `Counter` class with reactive fields. Visible win: a class instance acts like a reactive store.

---

## Lesson 28.1 — Class basics

A reactive class with `$state` fields *only compiles inside a `.svelte.ts` file*. The Svelte compiler treats that extension as runes-aware; in a plain `.ts` file, `$state` is undefined. Put this file at `src/lib/counter.svelte.ts`:

```ts
// src/lib/counter.svelte.ts
export class Counter {
  value = $state(0);

  increment(): void {
    this.value += 1;
  }

  reset(): void {
    this.value = 0;
  }
}
```

In any `.svelte` file:

```svelte
<script lang="ts">
  import { Counter } from '$lib/counter.svelte';
  const c = new Counter();
  c.increment(); // c.value is 1
</script>

<p>{c.value}</p>
<button type="button" onclick={() => c.increment()}>+</button>
```

`$state` fields on classes are reactive — reading them in markup tracks them. Senior pattern: *a class with `$state` fields is a reactive store you can pass around.*

> **class** — a factory for objects sharing methods.
>
> **`new Class()`** — instantiate.
>
> **`this`** — refers to the current instance inside a method.

---

## Lesson 28.2 — Constructors and parameter properties

```ts
class Counter {
  value = $state(0);

  constructor(initial: number = 0) {
    this.value = initial;
  }
}

const c = new Counter(5); // value starts at 5
```

You can also use TypeScript's parameter properties for fields that *aren't* reactive:

```ts
class Counter {
  value = $state(0);
  constructor(public readonly initial: number = 0) {
    this.value = initial;
  }
}
```

`public readonly initial: number` declares-and-assigns in one line — `initial` is a non-reactive frozen field.

> **A subtlety with `$state` in classes.** `$state(...)` must be on the *field declaration*, not assigned in the constructor body. `this.value = $state(0)` inside a constructor would assign a plain number to a non-reactive field. Always declare reactive fields at the class top: `value = $state(0)`. Then the constructor *reassigns* it (`this.value = initial`), which works because `value` is already reactive.

---

## Lesson 28.3 — Private fields

`#name` makes a field truly private:

```ts
class Counter {
  #value = $state(0);

  get value(): number { return this.#value; }
  increment(): void { this.#value += 1; }
}
```

External code can't access `c.#value`. Senior habit: when you want to enforce invariants, make state private and expose getters/methods.

---

## Lesson 28.4 — Now you write it

**The English sentence first:**

> *"Build a `Toggle` class with `value: boolean`, methods `on()`, `off()`, `flip()`."*

<details>
<summary>Worked answer</summary>

```ts
export class Toggle {
  value = $state(false);
  on(): void { this.value = true; }
  off(): void { this.value = false; }
  flip(): void { this.value = !this.value; }
}
```
</details>

---

## Lesson 28.5 — Recurring concepts from earlier chapters

- **`$state(...)`** (Ch 1) — the same rune, now on a class field.
- **Encapsulation via `#private`** — invariants enforced by visibility.
- **Methods returning `void`** (Ch 1) — same convention applies.

---

## Lesson 28.6 — What you can now read in the wild

After Chapter 28 you can:

- Read **`class X { value = $state(0); ... }`** as a reactive class.
- Read **`new X()`** and **`x.method()`** with full type inference.
- Read **`#field`** as truly private state.
- Spot the *"`$state` only on field declarations, not in constructors"* gotcha.

---

## End-of-chapter checkpoint

- [ ] You wrote a class with `$state` fields.
- [ ] You can read `class`, `new`, `this`, `#field` aloud.

---

# Chapter 29 — The `HabitStore` class

> *Today's job:* one class owns every habit operation. Visible win: the home page is shorter; the same store could power a detail page.

---

## Lesson 29.1 — `.svelte.ts` modules

To use runes outside `.svelte` files, the file extension is `.svelte.ts`:

```ts
// src/lib/habits.svelte.ts
import { habitId, ok, err, type Habit, type HabitId, type Result } from '$lib/types';

export class HabitStore {
  habits = $state<Habit[]>([]);

  add(name: string): Result<Habit, 'empty'> {
    const trimmed = name.trim();
    if (trimmed === '') return err('empty');
    // Duplicates are *allowed* — each habit has a unique id, and the user may genuinely
    // want to track "Read" twice (e.g. morning vs evening). We do not enforce uniqueness.
    const habit: Habit = {
      id: habitId(crypto.randomUUID()),
      name: trimmed,
      createdAt: Date.now(),
    };
    this.habits = [...this.habits, habit];
    return ok(habit);
  }

  remove(id: HabitId): void {
    this.habits = this.habits.filter((h) => h.id !== id);
  }

  rename(id: HabitId, newName: string): Result<void, 'empty'> {
    const trimmed = newName.trim();
    if (trimmed === '') return err('empty');
    this.habits = this.habits.map((h) => h.id === id ? { ...h, name: trimmed } : h);
    return ok(undefined);
  }

  get count(): number {
    return this.habits.length;
  }

  get addedToday(): number {
    return this.habits.filter((h) => isToday(h.createdAt)).length;
  }
}

function isToday(epochMs: number): boolean {
  const a = new Date(epochMs);
  const b = new Date();
  return a.getDate() === b.getDate() && a.getMonth() === b.getMonth() && a.getFullYear() === b.getFullYear();
}
```

---

## Lesson 29.2 — Using it in the home page

```svelte
<script lang="ts">
  import { HabitStore } from '$lib/habits.svelte';
  const store = new HabitStore();

  // seed
  store.add('Drink water');
  store.add('Read 20 minutes');
</script>

<p>{store.count} habits, {store.addedToday} added today.</p>

{#each store.habits as habit (habit.id)}
  <HabitRow {habit} onDelete={(id) => store.remove(id)} />
{/each}
```

The store owns state and operations. The page only renders.

---

## Lesson 29.3 — The SSR-singleton landmine

Tempting to write:

```ts
// ❌ DON'T
export const store = new HabitStore();
```

This creates *one shared instance* across the entire process. On a server, that's all users at once. **Sessions leak across users.** Don't do it.

Senior rule: **stores are instantiated per request / per component tree, never globally.** We'll formalise per-request stores via context in Chapter 32.

> **SSR-singleton landmine** — sharing state across users by mistake when running on a server.

---

## Lesson 29.4 — Recurring concepts from earlier chapters

- **Classes with `$state`** (Ch 28) — the foundation of `HabitStore`.
- **`Result<T, E>`** (Ch 27) — every fallible method returns one.
- **Branded types** (Ch 27) — `HabitId` keeps habit-IDs distinct from user-IDs.
- **Getters** — `get count()` is a reactive read computed each access.

---

## Lesson 29.5 — What you can now read in the wild

After Chapter 29 you can:

- Read a **class-based reactive store** in a `.svelte.ts` module.
- Pass a class instance as a prop and let the child read its reactive fields.
- Spot the **SSR-singleton landmine** in a code review.
- Pick between **per-component-tree stores** (via context, Ch 32) and **plain class instances**.

---

## End-of-chapter checkpoint

- [ ] `HabitStore.svelte.ts` exists.
- [ ] The home page uses it.
- [ ] You can articulate the SSR-singleton landmine.

---

# Chapter 30 — The `Money` helper module

> *Today's job:* `src/lib/money.ts` handles every cents operation Streak will ever do. Visible win: a small "demo prices" component shows formatted prices ($12.99, $0.00) with no float math.

---

## Lesson 30.1 — Why integer cents

```ts
0.1 + 0.2 // 0.30000000000000004
```

That's the most-shared bug in computing. Floats lose precision on common decimals. Money math with floats compounds error every operation.

The fix: **store money as integer cents.** $12.99 is `1299`. $0.00 is `0`. $1,000,000.00 is `100000000`.

Bible rule #10. Never violated.

---

## Lesson 30.2 — `Cents` branded type

```ts
// src/lib/money.ts
export type Cents = number & { readonly __brand: 'Cents' };

export function cents(n: number): Cents {
  if (!Number.isInteger(n)) throw new Error(`Cents must be integer; got ${n}`);
  return n as Cents;
}

export type Currency = 'USD' | 'EUR' | 'GBP';

export function formatCents(c: Cents, currency: Currency = 'USD'): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(c / 100);
}
```

Read aloud: *"Cents is a branded number. To make one, validate integer. To display, divide by 100 and format with Intl."*

---

## Lesson 30.3 — Basis points

For percentages we use *basis points* — 100 bps = 1%, 10000 bps = 100%. Integer-only:

```ts
export function applyBps(amount: Cents, bps: number): Cents {
  // Cents extends number — arithmetic works without an `as number` cast.
  // Math.round forces an integer; cents(...) re-validates and re-brands.
  const result = Math.round((amount * bps) / 10000);
  return cents(result);
}

// 8% tax on $12.99
const tax = applyBps(cents(1299), 800); // 104 cents = $1.04
```

Two senior choices in `applyBps`:

1. **`cents(result)` round-trips through the validator** — if a future bug causes `result` to be non-integer, we throw at the boundary, not silently propagate a corrupt `Cents`.
2. **No `as number` casts on `amount * bps`** — `Cents` *is* a `number` at runtime, so arithmetic compiles. The brand is type-only.

The intermediate `amount * bps` can overflow `Number.MAX_SAFE_INTEGER` (~9.0×10¹⁵) at extreme values. For consumer pricing (max ~$1B in cents = 10¹¹), this is safe. For enterprise tiers we'd promote to `BigInt`:

```ts
export function applyBpsBig(amount: Cents, bps: number): Cents {
  const product = BigInt(amount) * BigInt(bps);
  const rounded = (product + 5_000n) / 10_000n; // banker-style round
  return cents(Number(rounded));
}
```

---

## Lesson 30.4 — `splitCents` (for refunds)

```ts
export function splitCents(total: Cents, n: number): Cents[] {
  if (n <= 0) throw new Error('n must be positive');
  // Cents extends number — arithmetic compiles without `as number`.
  const base = Math.floor(total / n);
  const remainder = total - base * n;
  const out: Cents[] = [];
  for (let i = 0; i < n; i += 1) {
    out.push(cents(i < remainder ? base + 1 : base));
  }
  return out;
}

// $1.00 split among 3 = [34, 33, 33]
splitCents(cents(100), 3);
```

The remainder is distributed across the first `remainder` parts. Never lose a cent.

---

## Lesson 30.5 — Wiring a demo

Create `src/routes/demo-money/+page.svelte`:

```svelte
<script lang="ts">
  import { cents, formatCents, applyBps, splitCents } from '$lib/money';

  const price = cents(1299);
  const taxedPrice = applyBps(price, 800);
  const split = splitCents(price, 3);
</script>

<h1>Money demo</h1>
<p>Price: {formatCents(price)}</p>
<p>Tax (8%): {formatCents(taxedPrice)}</p>
<p>Split 3 ways: {split.map(formatCents).join(', ')}</p>
```

Visit `http://localhost:5173/demo-money`. You should see clean, correct money. We'll delete this demo once Stripe is in (Ch 49) — for now it's the runtime evidence.

---

## Lesson 30.6 — Read this code

```ts
const a = 0.1 + 0.2;
const b = (10 + 20) / 100;
console.log(a === b); // ?
```

<details>
<summary>Answer</summary>

`false`. `a` is `0.30000000000000004`. `b` is `0.3` exactly. Floats are lying to you in real time.

The senior fix: always integer cents.
</details>

---

## Lesson 30.7 — Now you write it

**The English sentence first:**

> *"Write `parseCents(input: string): Result<Cents, 'invalid'>` that accepts strings like '12.99' or '0.05' and returns the integer cents. Reject scientific notation, multiple decimals, etc."*

<details>
<summary>Worked answer (sketch)</summary>

In `src/lib/money.ts` (add at the bottom):

```ts
import { ok, err, type Result } from '$lib/types';

export function parseCents(input: string): Result<Cents, 'invalid'> {
  const trimmed = input.trim();
  if (!/^\d+(\.\d{1,2})?$/.test(trimmed)) return err('invalid');
  const [whole = '0', fraction = ''] = trimmed.split('.');
  const padded = (fraction + '00').slice(0, 2);
  const total = Number(whole) * 100 + Number(padded);
  return ok(cents(total));
}
```

The `[whole = '0', fraction = '']` defaults guard against `noUncheckedIndexedAccess` (Ch 7) — `split('.')` returns `string[]`, so each slot is `string | undefined`. We default both.

We'll write a property-based test for this in Chapter 58.
</details>

---

## Lesson 30.8 — Recurring concepts from earlier chapters

Part IV's spine, in one place:

- **Discriminated unions** (Ch 23) — `RequestState`, `Result`, future `BillingStatus`.
- **Generics** (Ch 24) — `Result<T, E>`, `<List>`, `groupBy`.
- **`as const` / `readonly` / `satisfies`** (Ch 25) — used by `HABIT_CATEGORIES`.
- **Boundary parsing** (Ch 26) — `parseHabit` from `unknown`.
- **Branded types + `Result`** (Ch 27) — `HabitId`, `UserId`, `Cents`.
- **Classes with `$state`** (Ch 28, 29) — `HabitStore`.
- **The `Money` module** (Ch 30) — `Cents`, `formatCents`, `applyBps`, `splitCents`, `parseCents`.

---

## Lesson 30.9 — What you can now read in the wild

After Part IV you can:

- Read most of a senior TypeScript codebase line-by-line: discriminated unions, generics, branded types, `Result` returns, class-based reactive stores.
- Spot a missing boundary parser and add one.
- Spot floats used for money and rewrite as integer cents.
- Pass type-checked, branded values across module boundaries with confidence.

---

## End-of-chapter checkpoint

- [ ] `src/lib/money.ts` exists.
- [ ] `formatCents`, `applyBps`, `splitCents` work.
- [ ] You felt the float bug in the dev console and now refuse to use floats for money.

End of Part IV. Next: SvelteKit routing.
