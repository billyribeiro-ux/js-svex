# Part II — Things, not just numbers

> *"The counter becomes a list. The list becomes typed. The typed list becomes searchable. By the end of Part II Streak knows what a Habit is."*

---

# Chapter 7 — Arrays, the list of habits

> *Today's job:* replace the single `habitsLoggedToday` counter with a list of habit *names*. Visible win: three habits render as a list, the same Log/Undo/Reset buttons still work but on the list's length.

---

## Lesson 7.1 — From a number to a list

Until now, *"how many habits I've logged"* has been a single number. That worked when habits were anonymous tally marks. They aren't, in real life — habits have *names*. So we replace the number with a list of names.

Update `src/routes/+page.svelte`. Replace the script section's first line:

```ts
let habitsLoggedToday = $state(0);
```

…with:

```ts
let habits: string[] = $state(['Drink water', 'Read 20 minutes', 'Walk 8000 steps']);
```

Read aloud: *"let habits be a reactive list of strings, starting with three real habits."*

Two observations:

1. **The variable name changed from `habitsLoggedToday` to `habits`.** What we're tracking is now *the list of habits themselves*, not just a count. The name reflects the meaning. *Names that lie are the most expensive kind of bug.*

2. **The type annotation `string[]` is explicit.** TypeScript could infer `string[]` from the literal array, but writing it out makes the function-signatures of helpers (which we'll write in a moment) easier to discuss. Mid-stack senior habit.

Now we have to fix everything that referenced `habitsLoggedToday`. The count is now `habits.length`. The "any habits?" check is `habits.length > 0`. Update every reference:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habits: string[] = $state(['Drink water', 'Read 20 minutes', 'Walk 8000 steps']);
  let userName = $state('');

  const days: string[] = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

  function getMondayBasedDayIndex(): number {
    const sundayBased: number = new Date().getDay();
    if (sundayBased === 0) return 6;
    return sundayBased - 1;
  }

  const todayIndex: number = getMondayBasedDayIndex();
</script>

<h1>Welcome back, {userName.trim() === '' ? 'friend' : userName}.</h1>

<label>
  Your name
  <input type="text" bind:value={userName} placeholder="What should we call you?" />
</label>

<div class="day-strip">
  {#each days as day, i}
    <span class="day" class:today={i === todayIndex} class:past={i < todayIndex}>{day}</span>
  {/each}
</div>

{#if habits.length === 0}
  <div class="empty-state">
    <h2>No habits yet</h2>
    <p>Add your first one!</p>
  </div>
{:else}
  <p>
    You're tracking
    <strong>{habits.length}</strong>
    {habits.length === 1 ? 'habit' : 'habits'}.
  </p>

  <ul class="habit-list">
    {#each habits as habit}
      <li>{habit}</li>
    {/each}
  </ul>
{/if}

<style>
  .day-strip { display: flex; gap: 0.5rem; margin: 1rem 0; }
  .day { padding: 0.25rem 0.5rem; border-radius: 999px; background: #eee; font-size: 0.875rem; }
  .day.today { background: #2563eb; color: white; font-weight: 600; }
  .day.past { opacity: 0.5; }

  .empty-state { text-align: center; padding: 2rem; margin: 2rem 0; border: 2px dashed #ccc; border-radius: 0.5rem; }
  .empty-state h2 { margin-top: 0; color: #666; }

  .habit-list { list-style: none; padding: 0; }
  .habit-list li { padding: 0.5rem 0.75rem; background: #f5f5f5; margin: 0.25rem 0; border-radius: 0.25rem; }
</style>
```

Save (`Cmd+S` / `Ctrl+S`). You'll see three habits listed.

I removed the *Log a habit*, *Undo*, *Reset* buttons in this chapter — they don't fit the new model, and we're going to put back something better in Chapter 8 (a real *add* and *delete*). Don't worry; we're not losing functionality, we're upgrading it.

---

## Lesson 7.2 — `noUncheckedIndexedAccess` and the small TypeScript surprise

If you try to access `habits[0]` directly:

```ts
const first: string = habits[0]; // ❌ Type error in strict mode
```

…TypeScript will complain. The exact error is something like:

> *Type 'string | undefined' is not assignable to type 'string'.*

Why? Because TypeScript, in strict mode with `noUncheckedIndexedAccess: true` (which our scaffold turned on), doesn't trust that the array is non-empty. `habits[0]` could be `undefined` if `habits` is empty. The compiler is forcing you to acknowledge that.

You have two correct ways to handle it:

```ts
// Option 1 — accept the union type and narrow before use.
const first1: string | undefined = habits[0];
if (first1 === undefined) return;
// from here on, TypeScript knows first1 is string.

// Option 2 — provide a default with `??`.
const first2: string = habits[0] ?? 'fallback';
```

Or, equivalently, destructure-and-narrow:

```ts
const [first] = habits;
if (first === undefined) return;
// from here on, TypeScript knows first is string.
```

Or with `array.at(...)` (formal coverage Ch 11):

```ts
const first = habits.at(0); // string | undefined — same shape, narrow as above
```

There's a third syntax in the wild — `habits[0]!` with a `!` non-null assertion that *lies* to TypeScript and says "trust me, this isn't undefined." **It's banned in this book — Bible rule #3.** We never write it; the two patterns above cover every real case.

For now we'll loop with `{#each}` and avoid raw indexing. We'll meet `at()` and `find()` properly in Chapter 11.

---

## Lesson 7.3 — `{#each}` keyed with an `(item)` term

When the list might *change* (items added, removed, reordered), Svelte needs to know which DOM node corresponds to which item. Otherwise it might re-render the wrong row when a habit is deleted, or animate the wrong item when one is added.

For now `habits` is an array of strings, so we don't have unique IDs yet. We'll add IDs in Chapter 9, and at that point we'll switch to **keyed `{#each}`**:

```svelte
{#each habits as habit (habit.id)}
  <li>{habit.name}</li>
{/each}
```

Read aloud: *"for each habit in habits, keyed by `habit.id`, render a list item."*

> **keyed `{#each}`** *(noun)* — passing `(uniqueKey)` after the loop variable so Svelte tracks identity. Critical for correct rendering when items can move, animate, or change.

For Chapter 7's array of plain strings, we don't have unique identity (you could have *"Read"* twice and they'd be indistinguishable). That's a sign we need a richer data shape — which is exactly what Chapter 9 brings.

---

## Lesson 7.4 — Read this code

### Snippet A

```ts
const fruits: string[] = ['apple', 'banana'];
const count: number = fruits.length;
console.log(`There are ${count} fruits.`);
```

What's printed? And what type is `count`?

<details>
<summary>Answer</summary>

`There are 2 fruits.` `count` is `number`. The `length` property of an array is always `number`.
</details>

### Snippet B

```svelte
<script lang="ts">
  const colors: string[] = [];
</script>

{#if colors.length === 0}
  <p>No colors yet.</p>
{:else}
  <p>{colors.length} colors</p>
{/if}
```

What does the page show?

<details>
<summary>Answer</summary>

*"No colors yet."* — the array is empty (`length` is `0`), the `{#if}` branch is taken.
</details>

---

## Lesson 7.5 — Now you write it

**The English sentence first:**

> *"Add a fourth and a fifth habit to the initial list — your two real habits. Watch the count and the list both update on hot reload, with no extra work."*

One-line edit in the script. The point is to *feel* hot reload re-render and confirm the markup adapts to a different list.

<details>
<summary>Worked answer</summary>

```ts
let habits: string[] = $state([
  'Drink water',
  'Read 20 minutes',
  'Walk 8000 steps',
  'Stretch',           // your two
  'Journal one line',
]);
```

Save. The header now says *"You're tracking 5 habits"* and the list has five items. Watch the `5` text update letter-by-letter when you change the count via `{habits.length}`. That's `$state` + reactivity doing its job.
</details>

---

## Lesson 7.6 — Recurring concepts from earlier chapters

- **`$state(...)`** (Ch 1) — `habits` is reactive, so the count and list both update.
- **`{#if ... }{:else}{/if}`** (Ch 6) — empty-state branch.
- **`{#each ... as ... }`** (Ch 5) — rendering the list.
- **Plural-aware text** (Ch 1) — `habits.length === 1 ? 'habit' : 'habits'`.
- **`type="button"`, scoped `<style>`, the `userName` greeting** — all still working.

---

## Lesson 7.7 — What you can now read in the wild

After Chapter 7 you can:

- Read **`let xs: T[] = $state([...])`** as a typed reactive array.
- Read **`xs.length`**, **`xs[i]`** (and know it's `T | undefined` in strict mode).
- Read **`xs.at(i)`** — safe alternative to indexing.
- Spot when a snippet is using a *keyed* `{#each}` vs an unkeyed one and explain why.

---

## Glossary added in Chapter 7

| Term | Definition |
|---|---|
| `string[]` | Array of strings. |
| `array.length` | The number of elements. |
| `array.at(i)` | Safe indexed access — returns `T \| undefined`. |
| `noUncheckedIndexedAccess` | Strict-mode flag that makes `arr[i]` typed as `T \| undefined`. |
| keyed `{#each}` | `{#each items as item (item.id)}` — tracked by identity. |

---

## End-of-chapter checkpoint

- [ ] You see your habits as a list, not a number.
- [ ] The empty-state card appears if you set the array to `[]`.
- [ ] You can explain why `noUncheckedIndexedAccess` makes `habits[0]` a `string | undefined`.

---

# Chapter 8 — Adding and removing, immutably

> *Today's job:* a text input that adds a new habit; an X button on each row that removes it. *Visible win:* the list grows and shrinks live as you type and click.

We're going to write our first form, our first delete handler, and learn the senior-engineer rule about *immutable* updates.

---

## Lesson 8.1 — `bind:value`, formally

You met `bind:value` glancingly in Chapter 4. Now formally:

```svelte
<script lang="ts">
  let newHabit: string = $state('');
</script>

<input type="text" bind:value={newHabit} />
<p>You typed: {newHabit}</p>
```

Read aloud: *"bind the input's value to `newHabit`."*

When the user types, `newHabit` updates. When code reassigns `newHabit`, the input updates. Two-way.

> **`bind:value`** — Svelte directive: two-way bind a variable to an input's value.

---

## Lesson 8.2 — Adding to an array, immutably

The naive way to add to an array is `array.push(item)`:

```ts
habits.push('New habit');
```

This works. But we **don't write it that way**. We use the spread operator:

```ts
habits = [...habits, 'New habit'];
```

Read aloud: *"the new habits list is the old list, plus the new habit at the end."*

> **spread operator (`...`)** — expands an array (or object) into its elements. `[...habits, 'New']` makes a new array containing every old element followed by `'New'`.

Why prefer this over `push`?

1. **Reactivity** — Svelte 5's `$state` proxies handle both, but immutable updates produce a *new array reference*. Reference-equality changes trigger downstream `$derived` re-computation reliably; in-place mutations don't always (depending on how the value is used).
2. **Reasoning** — when you assign `habits = newList`, every reader can be sure they're looking at the new state. With `push`, the same array object now contains different elements; readers who captured `const oldHabits = habits;` get a surprise.
3. **History / undo** — immutable updates make undo trivial. You keep references to old states.
4. **Convention** — modern codebases lean immutable. Writing `push` in a code review will earn comments.

So our rule: **for `$state` arrays, always reassign with a new array.**

---

## Lesson 8.3 — Removing with `array.filter`

Removal mirrors the same idea:

```ts
habits = habits.filter((h) => h !== habitToRemove);
```

Read aloud: *"the new habits list is the old list, with `habitToRemove` filtered out."*

> **`array.filter(predicate)`** — returns a new array containing only elements for which `predicate(element)` is true. Doesn't mutate the original.

We'll meet `filter` formally in Chapter 11. Today we use the simplest case.

---

## Lesson 8.4 — Wiring it into Streak

Update `+page.svelte`. The script gains a `newHabit` field, an `addHabit` function, and a `removeHabit` function. The markup gains a form and an X button on each row.

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habits: string[] = $state(['Drink water', 'Read 20 minutes', 'Walk 8000 steps']);
  let newHabit: string = $state('');
  let userName = $state('');

  const days: string[] = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

  function getMondayBasedDayIndex(): number {
    const sundayBased: number = new Date().getDay();
    if (sundayBased === 0) return 6;
    return sundayBased - 1;
  }

  const todayIndex: number = getMondayBasedDayIndex();

  function addHabit(): void {
    const trimmed: string = newHabit.trim();
    if (trimmed === '') {
      return;
    }

    habits = [...habits, trimmed];
    newHabit = '';
  }

  function removeHabit(habitToRemove: string): void {
    habits = habits.filter((h) => h !== habitToRemove);
  }

  function handleSubmit(event: SubmitEvent): void {
    event.preventDefault();
    addHabit();
  }
</script>

<h1>Welcome back, {userName.trim() === '' ? 'friend' : userName}.</h1>

<label>
  Your name
  <input type="text" bind:value={userName} placeholder="What should we call you?" />
</label>

<div class="day-strip">
  {#each days as day, i}
    <span class="day" class:today={i === todayIndex} class:past={i < todayIndex}>{day}</span>
  {/each}
</div>

<form onsubmit={handleSubmit}>
  <input
    type="text"
    bind:value={newHabit}
    placeholder="Add a new habit..."
  />
  <button type="submit" disabled={newHabit.trim() === ''}>Add</button>
</form>

{#if habits.length === 0}
  <div class="empty-state">
    <h2>No habits yet</h2>
    <p>Add your first one above.</p>
  </div>
{:else}
  <p>You're tracking <strong>{habits.length}</strong> {habits.length === 1 ? 'habit' : 'habits'}.</p>

  <ul class="habit-list">
    {#each habits as habit}
      <li>
        <span>{habit}</span>
        <button type="button" onclick={() => removeHabit(habit)} aria-label="Remove {habit}">×</button>
      </li>
    {/each}
  </ul>
{/if}

<style>
  /* ... earlier styles ... */
  form { margin: 1rem 0; display: flex; gap: 0.5rem; }
  form input { flex: 1; padding: 0.5rem; }
  form button { padding: 0.5rem 1rem; }

  .habit-list li {
    display: flex; justify-content: space-between; align-items: center;
    padding: 0.5rem 0.75rem; background: #f5f5f5; margin: 0.25rem 0; border-radius: 0.25rem;
  }
  .habit-list button {
    background: transparent; border: none; cursor: pointer; font-size: 1.25rem; color: #888;
  }
  .habit-list button:hover { color: #c00; }
</style>
```

Save (`Cmd+S` / `Ctrl+S`). Type a habit, hit Enter (or click *Add*); it appears. Click the × on any row; it disappears.

---

## Lesson 8.5 — The careful pieces

Several senior-engineer choices in that snippet:

**1. `newHabit.trim()` before checking.** A user typing only spaces shouldn't add an empty habit. `.trim()` removes leading/trailing whitespace.

> **`.trim()`** — string method that returns a new string with surrounding whitespace removed.

**2. The `disabled` is on the button, not just a guard in `addHabit`.** Two layers of defence.

**3. The `onsubmit={handleSubmit}` with `event.preventDefault()`.** A `<form>`, by default, reloads the page on submit. We don't want that — we want our JavaScript to handle the add. `event.preventDefault()` cancels the default behaviour. We'll meet form actions properly in Chapter 41 where the *server* handles submission; today we're handling client-side only.

> **`event.preventDefault()`** — cancel the browser's default action for an event. For form submit: don't reload the page.
>
> **`SubmitEvent`** — the TypeScript type for a form-submit event.

**4. The arrow function in `onclick={() => removeHabit(habit)}`.** When you need to *call* a handler with arguments, you can't write `onclick={removeHabit(habit)}` — that calls `removeHabit` immediately at render time. The arrow function `() => removeHabit(habit)` *creates a function* that, when called, calls `removeHabit(habit)`. This is the senior idiom for parameterised handlers.

**5. `aria-label="Remove {habit}"`.** Screen readers read the button's accessible label. *"Remove Drink water"* is helpful; *"×"* is not. Senior accessibility habit; we'll deepen Chapter 53.

**6. `+= 1` *isn't* in this chapter.** Adding a habit to a list isn't *"increase by one"* — it's *"append"*. The shape of the operation is different from the shape of incrementing a counter. Senior reading skill: notice when an idiom from one context doesn't fit another.

---

## Lesson 8.6 — The "names aren't unique" problem

Add a habit named *"Read"*. Now add another habit named *"Read"*. The list has two identical entries. Click the × on the first one. What happens?

Both vanish. Because `removeHabit` filters out *every* habit equal to the string `"Read"`.

That's a real bug. It's foreshadowed in Chapter 7 — *"strings aren't unique enough to be identities."* The fix is to give each habit a unique ID. We'll do exactly that in Chapter 9.

---

## Lesson 8.7 — Now you write it

**The English sentence first:**

> *"Add a 'Clear all' button below the list. When clicked, it empties the habits array. Use the same pattern as the existing handlers — a `function` with a `void` return, no arguments."*

<details>
<summary>Worked answer</summary>

```ts
function clearAllHabits(): void {
  habits = [];
}
```

```svelte
{#if habits.length > 0}
  <button type="button" onclick={clearAllHabits}>Clear all</button>
{/if}
```

We wrap the button in `{#if habits.length > 0}` so it doesn't render at all when there's nothing to clear (instead of using `disabled`). Either is reasonable; `{#if}` is cleaner for "the action is meaningless right now, don't even show it."
</details>

---

## Lesson 8.8 — Recurring concepts from earlier chapters

- **Guard clause** (Ch 2) — `if (trimmed === '') return;` in `addHabit`.
- **`.trim()`** (Ch 4) — same trick we used for `userName`.
- **`array.filter`** (Ch 7 preview) — formal use here.
- **`disabled={cond}`** (Ch 3) — *Add* button greys when the input is empty.
- **Defence in depth** (Ch 6) — UI guard *and* function guard, working together.
- **`type="button"`** (Ch 1) — every new button has it.

---

## Lesson 8.9 — What you can now read in the wild

After Chapter 8 you can:

- Read **`xs = [...xs, item]`** as the immutable append.
- Read **`xs = xs.filter((x) => x !== target)`** as the immutable remove.
- Read **`<form onsubmit={handler}>`** with `event.preventDefault()` and explain the page-reload prevention.
- Read **`onclick={() => fn(arg)}`** as a parameterised handler.
- Read **`aria-label="..."`** as accessible text for icon-only buttons.

---

## Glossary added in Chapter 8

| Term | Definition |
|---|---|
| `bind:value` | Two-way bind a variable to an input. |
| spread operator `...` | Expands an array/object into its elements. |
| `array.filter(pred)` | New array of elements satisfying `pred`. |
| `.trim()` | Remove surrounding whitespace from a string. |
| `event.preventDefault()` | Cancel the browser's default for an event. |
| `SubmitEvent` | TypeScript type for a form submit. |
| `aria-label` | Accessible label for screen readers. |
| arrow function | `() => ...` — anonymous function literal. |
| immutable update | Producing a new array/object instead of mutating. |

---

## End-of-chapter checkpoint

- [ ] You can add a habit and see it appear.
- [ ] You can delete a habit by clicking ×.
- [ ] You can explain *out loud* why we use `[...habits, item]` instead of `habits.push(item)`.
- [ ] You found the duplicate-name bug.

---

# Chapter 9 — Objects and your first real `Habit` type

> *Today's job:* each habit gains a stable `id`, the `name`, and a `createdAt` timestamp. Visible win: each row shows the name and "added 2 minutes ago".

This is the chapter where Streak goes from a list of strings to a real domain. We meet objects, our first `type`, and the small senior-habit detail of "every entity has an id."

---

## Lesson 9.1 — What an object is, mentally

An **object** is a collection of named values. Think *labelled drawers in a filing cabinet*. Each drawer has a *key* (a string name) and holds a *value*.

```ts
const billy = {
  firstName: 'Billy',
  age: 33,
  hasDog: false,
};
```

You read values with dot-access:

```ts
billy.firstName // 'Billy'
billy.age       // 33
```

> **object** *(noun)* — a collection of key/value pairs. Keys are strings; values can be anything.
>
> **property** *(noun)* — a single key/value pair on an object. `firstName: 'Billy'` is a property.
>
> **dot access** — reading a property with `obj.key`.
>
> **bracket access** — reading a property with `obj['key']`. Used when the key is dynamic. We use dot when we can.

You can also write values:

```ts
billy.age = 34;
```

But the same caution applies as with arrays: for `$state` objects we prefer immutable updates.

---

## Lesson 9.2 — `type Habit`

In TypeScript, you describe the *shape* of an object with a `type`:

```ts
type Habit = {
  id: string;
  name: string;
  createdAt: number;
};
```

Read aloud: *"a Habit is a thing with an id (string), a name (string), and a createdAt timestamp (number)."*

> **`type`** — TypeScript keyword that declares a named type. Used heavily.
>
> **`interface`** — a similar keyword. For most purposes, `type` and `interface` are interchangeable. *In this book we use `type`* — it composes better (unions, intersections), and consistency is its own value. Reading other people's `interface`s is no trouble; you'll see them in the wild.

You can use `Habit` anywhere a type is expected:

```ts
let habits: Habit[] = $state([]);

function makeHabit(name: string): Habit {
  return {
    id: crypto.randomUUID(),
    name: name,
    createdAt: Date.now(),
  };
}
```

`crypto.randomUUID()` and `Date.now()` are about to become regulars.

> **`crypto.randomUUID()`** — a built-in function that returns a unique 36-character string like `'a3b1...'`. Globally unique enough for practical purposes; perfect for IDs. Available in modern browsers and in Node 19+. Requires a *secure context* — works on `localhost` and on any `https://` site, but fails on plain `http://` outside localhost. We won't deploy outside HTTPS, so this is fine.
>
> **`Date.now()`** — milliseconds since the Unix epoch (Jan 1, 1970 UTC). A `number`.

We store `createdAt` as a `number` (epoch ms), not a `Date`. Why? Because numbers serialise cleanly (they survive JSON round-trips), they sort naturally, and they're trivially comparable. We convert to a `Date` when we need to *display* it. This is a common senior pattern.

---

## Lesson 9.3 — A `formatRelativeTime` helper

We want each row to show *"added 2 minutes ago"*. Create a new file, `src/lib/formatRelativeTime.ts`:

```ts
// src/lib/formatRelativeTime.ts
export function formatRelativeTime(epochMs: number, now: number = Date.now()): string {
  const diffMs: number = now - epochMs;
  const diffSeconds: number = Math.floor(diffMs / 1000);

  if (diffSeconds < 5) return 'just now';
  if (diffSeconds < 60) return `${diffSeconds} seconds ago`;

  const diffMinutes: number = Math.floor(diffSeconds / 60);
  if (diffMinutes < 60) return `${diffMinutes} ${diffMinutes === 1 ? 'minute' : 'minutes'} ago`;

  const diffHours: number = Math.floor(diffMinutes / 60);
  if (diffHours < 24) return `${diffHours} ${diffHours === 1 ? 'hour' : 'hours'} ago`;

  const diffDays: number = Math.floor(diffHours / 24);
  return `${diffDays} ${diffDays === 1 ? 'day' : 'days'} ago`;
}
```

Read aloud: *"format a relative time. Take an epoch millisecond timestamp and (optionally) a 'now' for testing. Return human-readable text like 'just now' or '2 minutes ago'."*

Several senior pieces:

1. **The function is in `src/lib/`.** Reusable code lives in `src/lib/`. The path becomes the import: `import { formatRelativeTime } from '$lib/formatRelativeTime';`.

> **`$lib`** — Svelte's import alias for `src/lib/`. Using `$lib/foo` instead of `../../../lib/foo` keeps imports stable when files move.

2. **Default parameter `now: number = Date.now()`.** Lets us pass a fake `now` in tests so the function is testable without mocking time. Senior testability habit.

3. **`Math.floor(...)`.** Truncates a decimal toward zero. We use it because `diffMs / 1000` can be `123.456` — we want the integer second count.

> **`Math.floor`** — round down to the nearest integer.

4. **The early-return guard pattern from Chapter 2** — applied to a chain of "is it under X?" checks. Senior idiom in time-formatting helpers.

5. **Plural-aware text** — *"1 minute ago"*, *"2 minutes ago"*. Same care as Chapter 1's `habit / habits`.

---

## Lesson 9.4 — Wiring it all into Streak

Update `src/routes/+page.svelte`:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';

  type Habit = {
    id: string;
    name: string;
    createdAt: number;
  };

  function makeHabit(name: string): Habit {
    return {
      id: crypto.randomUUID(),
      name: name,
      createdAt: Date.now(),
    };
  }

  let habits: Habit[] = $state([
    makeHabit('Drink water'),
    makeHabit('Read 20 minutes'),
    makeHabit('Walk 8000 steps'),
  ]);
  let newHabit: string = $state('');
  let userName = $state('');

  // ... days, getMondayBasedDayIndex, todayIndex unchanged ...

  function addHabit(): void {
    const trimmed: string = newHabit.trim();
    if (trimmed === '') return;

    habits = [...habits, makeHabit(trimmed)];
    newHabit = '';
  }

  function removeHabit(id: string): void {
    habits = habits.filter((h) => h.id !== id);
  }

  function handleSubmit(event: SubmitEvent): void {
    event.preventDefault();
    addHabit();
  }
</script>

<!-- ... markup mostly unchanged, but the list becomes: -->

<ul class="habit-list">
  {#each habits as habit (habit.id)}
    <li>
      <div>
        <strong>{habit.name}</strong>
        <small>{formatRelativeTime(habit.createdAt)}</small>
      </div>
      <button type="button" onclick={() => removeHabit(habit.id)} aria-label="Remove {habit.name}">×</button>
    </li>
  {/each}
</ul>
```

Save (`Cmd+S` / `Ctrl+S`). Add a new habit. Notice it shows *"just now"*. Add the same name twice — both rows survive deletion of the other, because they have different IDs.

The `(habit.id)` in `{#each habits as habit (habit.id)}` is the **keyed each** Chapter 7 previewed. Now it has a real key.

---

## Lesson 9.5 — Read this code

### Snippet A

```ts
type User = {
  id: string;
  email: string;
  isPro: boolean;
  joinedAt: number;
};

const u: User = {
  id: 'abc',
  email: 'billy@example.com',
  isPro: true,
  joinedAt: Date.now(),
};

console.log(u.email, u.isPro);
```

What's printed?

<details>
<summary>Answer</summary>

`billy@example.com true`. Two values, separated by a space (the default with multiple `console.log` arguments).
</details>

### Snippet B

```ts
type Habit = { name: string; description?: string };

const h1: Habit = { name: 'Read' };
const h2: Habit = { name: 'Walk', description: 'Take the dog' };
console.log(h1.description ?? 'no description');
console.log(h2.description ?? 'no description');
```

<details>
<summary>Answer</summary>

`no description` then `Take the dog`. The `?` after `description` makes the property optional — the field can be absent, in which case access returns `undefined`. `??` falls back to the default.

This is *exactly* the pattern that makes `??` and optional properties so useful.
</details>

---

## Lesson 9.6 — Now you write it

**The English sentence first:**

> *"Habits should optionally have a description. Add a `description?: string` field to the `Habit` type, surface a second input in the form for it, and render the description under the habit name when present."*

<details>
<summary>Worked answer (sketch)</summary>

```ts
type Habit = {
  id: string;
  name: string;
  description?: string;
  createdAt: number;
};

let newDescription: string = $state('');

function addHabit(): void {
  const trimmedName: string = newHabit.trim();
  if (trimmedName === '') return;

  const trimmedDesc: string = newDescription.trim();

  habits = [
    ...habits,
    {
      id: crypto.randomUUID(),
      name: trimmedName,
      description: trimmedDesc === '' ? undefined : trimmedDesc,
      createdAt: Date.now(),
    },
  ];
  newHabit = '';
  newDescription = '';
}
```

In markup:

```svelte
<form onsubmit={handleSubmit}>
  <input type="text" bind:value={newHabit} placeholder="Habit name..." />
  <input type="text" bind:value={newDescription} placeholder="(optional) description..." />
  <button type="submit" disabled={newHabit.trim() === ''}>Add</button>
</form>

<!-- in the list -->
{#each habits as habit (habit.id)}
  <li>
    <div>
      <strong>{habit.name}</strong>
      {#if habit.description}
        <p class="habit-desc">{habit.description}</p>
      {/if}
      <small>{formatRelativeTime(habit.createdAt)}</small>
    </div>
    <button type="button" onclick={() => removeHabit(habit.id)} aria-label="Remove {habit.name}">×</button>
  </li>
{/each}
```

The `description: trimmedDesc === '' ? undefined : trimmedDesc` line is a small but important detail: we don't store an empty string for "no description". `undefined` is the right value for *missing*. Same lesson as Chapter 4's `null` vs empty string.
</details>

---

## Lesson 9.7 — Recurring concepts from earlier chapters

- **Guard clauses** (Ch 2) — `formatRelativeTime` is one big chain of them.
- **Plural-aware text** (Ch 1) — *"1 minute ago"* vs *"2 minutes ago"*.
- **`{#if cond}...{/if}`** (Ch 6) — render description when present.
- **Spread + filter** (Ch 8) — still how we add and remove habits.
- **`type="button"` + `aria-label`** (Ch 1, 8) — every new button has them.
- **The "input model vs data model"** rule (Ch 4) — `description` is `string` in the input, `string | undefined` in the data.

---

## Lesson 9.8 — What you can now read in the wild

After Chapter 9 you can:

- Read **`type X = { ... }`** declarations and explain each field.
- Read **`field?: T`** as optional.
- Read **`crypto.randomUUID()`** and **`Date.now()`** as ID and timestamp generators.
- Read **`obj.field`** vs **`obj['field']`** — and know when each is used.
- Read **`import { fn } from '$lib/...'`** as a path-stable internal import.
- Read a small helper function with multiple early-return guards and explain its branches.

---

## Glossary added in Chapter 9

| Term | Definition |
|---|---|
| object | A collection of named values. |
| property | A key/value pair on an object. |
| dot access / bracket access | Two ways to read properties. |
| `type` | TypeScript keyword for a named type. |
| `interface` | Similar to `type`; we prefer `type`. |
| optional property `?` | A property that may or may not be present. |
| `crypto.randomUUID()` | Generates a unique-enough string ID (secure context). |
| `Date.now()` | Milliseconds since 1970 UTC. |
| `Math.floor` | Round down to integer. |
| `$lib` | Import alias for `src/lib/`. |
| secure context | A page served over HTTPS or `localhost`. Required for `crypto.randomUUID()`. |

---

## End-of-chapter checkpoint

- [ ] Habits now have IDs, names, and timestamps.
- [ ] Each row shows a relative-time string under the name.
- [ ] Adding the same name twice no longer breaks delete.
- [ ] (After exercise) descriptions are optional and render correctly.

---

# Chapter 10 — Destructuring, read what you need

> *Today's job:* clean up the habit-row markup. Visible win: identical UI; cleaner script.

---

## Lesson 10.1 — Object destructuring

When you want several properties from an object, you can name them all at once:

```ts
const habit: Habit = { id: 'abc', name: 'Read', createdAt: 123 };

const { id, name, createdAt } = habit;
```

Read aloud: *"pull `id`, `name`, and `createdAt` out of habit and bind them to local names."*

You can rename:

```ts
const { name: habitName } = habit;
// now habitName === 'Read'
```

You can default:

```ts
const { description = '(no description)' } = habit;
// description is the property's value, or the default if missing
```

You can combine with rest:

```ts
const { id, ...rest } = habit;
// id is 'abc'; rest is { name, createdAt, description? }
```

---

## Lesson 10.2 — Function-parameter destructuring

This is the form you'll see in 99% of senior code:

```ts
function HabitRow({ habit }: { habit: Habit }) {
  // ...
}
```

Read aloud: *"a function whose argument is an object with a `habit` property of type Habit."*

This is a preview of `$props` — Svelte 5 components destructure props this way:

```svelte
<script lang="ts">
  let { habit }: { habit: Habit } = $props();
</script>
```

We meet `$props` formally in Chapter 14.

---

## Lesson 10.3 — Wiring it into Streak

In our existing markup, the `{#each}` block can use destructuring inline:

```svelte
{#each habits as { id, name, description, createdAt } (id)}
  <li>
    <div>
      <strong>{name}</strong>
      {#if description}
        <p class="habit-desc">{description}</p>
      {/if}
      <small>{formatRelativeTime(createdAt)}</small>
    </div>
    <button type="button" onclick={() => removeHabit(id)} aria-label="Remove {name}">×</button>
  </li>
{/each}
```

Same UI; less `habit.` repetition.

You can also destructure inside helper functions:

```ts
function summariseHabit({ name, createdAt }: Habit): string {
  return `${name} (added ${formatRelativeTime(createdAt)})`;
}
```

---

## Lesson 10.4 — Read this code

### Snippet A — destructure with default

```ts
type Settings = { theme?: 'light' | 'dark'; fontSize?: number };

function applySettings({ theme = 'light', fontSize = 16 }: Settings): void {
  console.log(theme, fontSize);
}

applySettings({});
applySettings({ theme: 'dark' });
```

What's printed?

<details>
<summary>Answer</summary>

```
light 16
dark 16
```

`theme = 'light'` and `fontSize = 16` provide defaults *during destructuring*. The first call passes an empty object — both defaults apply. The second call overrides `theme` but not `fontSize`.
</details>

### Snippet B — five positional vs five destructured

Which would you rather call from a different file?

```ts
// Version 1: positional
function createInvoice(amount: number, currency: string, dueDate: number, customerId: string, isRecurring: boolean): Invoice {
  // ...
}
createInvoice(1000, 'USD', Date.now() + 86400000, 'cust_123', false);

// Version 2: destructured
function createInvoice({ amount, currency, dueDate, customerId, isRecurring }: {
  amount: number; currency: string; dueDate: number; customerId: string; isRecurring: boolean;
}): Invoice {
  // ...
}
createInvoice({
  amount: 1000,
  currency: 'USD',
  dueDate: Date.now() + 86400000,
  customerId: 'cust_123',
  isRecurring: false,
});
```

<details>
<summary>Answer</summary>

Version 2, almost always. The call site is self-documenting. Reading the positional version, you have to remember the order. Reading the destructured version, every argument tells you what it is. Senior habit: when a function has more than three parameters, take an options object.
</details>

---

## Lesson 10.5 — Now you write it

**The English sentence first:**

> *"Refactor `summariseHabit({ name, createdAt }: Habit): string` (from Lesson 10.3) to also accept the optional `description` and include it when present, like 'Read 20 minutes — Take time for a real book (added 5 minutes ago)'."*

You'll need to destructure with a default and use a ternary inside a template literal.

<details>
<summary>Worked answer</summary>

```ts
function summariseHabit({ name, description, createdAt }: Habit): string {
  const descPart: string = description !== undefined ? ` — ${description}` : '';
  return `${name}${descPart} (added ${formatRelativeTime(createdAt)})`;
}
```

Or with a destructure default (slightly less readable but valid):

```ts
function summariseHabit({ name, description = '', createdAt }: Habit): string {
  const descPart: string = description !== '' ? ` — ${description}` : '';
  return `${name}${descPart} (added ${formatRelativeTime(createdAt)})`;
}
```

Either works. The first version is clearer about *"description is optional"*; the second uses destructuring's default-value feature.

Senior habit: when destructuring optional fields, decide whether `undefined` is meaningfully different from the default. If yes (rare): keep `?`, no destructure default. If no: use a destructure default.
</details>

---

## Lesson 10.6 — Recurring concepts from earlier chapters

- **Object types** (Ch 9) — destructuring is a way to read those types.
- **Optional properties `?`** (Ch 9) — destructure defaults handle them.
- **Senior habit: "more than three params? take an options object"** — formalised here.

---

## Lesson 10.7 — What you can now read in the wild

After Chapter 10 you can:

- Read **`const { x, y, z } = obj`** as multi-property extraction.
- Read **`const { x: localName } = obj`** as renaming.
- Read **`const { x = default } = obj`** as fallback.
- Read **`const { x, ...rest } = obj`** as splitting one out.
- Read **`function f({ x, y }: { x: T; y: U })`** — the standard senior shape.

---

## Glossary added in Chapter 10

| Term | Definition |
|---|---|
| destructuring | Pulling several properties out of an object at once. |
| renaming on destructure | `const { foo: localName }` |
| default on destructure | `const { foo = 'default' }` |
| rest on destructure | `const { foo, ...rest }` |
| function-parameter destructuring | `function f({ x, y })` |

---

## End-of-chapter checkpoint

- [ ] You can read `const { x, y, z } = obj;` aloud.
- [ ] You can read `function f({ x, y }: T)` aloud.
- [ ] You refactored `removeHabit` (or kept it the same and explained why).

---

# Chapter 11 — `map`, `filter`, `find`, `some`, `every`, `includes`

> *Today's job:* a search box that narrows the list; a count of matching habits; a "any habit added today?" indicator. Visible win: type "read" — only matching habits show.

These are the array methods you'll use every single day. Each one is small.

---

## Lesson 11.1 — `array.map(fn)` — transform each element

```ts
const habits: Habit[] = [/* ... */];
const names: string[] = habits.map((h) => h.name);
```

Read aloud: *"map every habit to its name."*

> **`array.map(fn)`** — returns a new array where each element is the result of `fn(originalElement)`.

Doesn't mutate the original. Returns a *new* array of the same length.

---

## Lesson 11.2 — `array.filter(pred)` — keep matching

We've seen it; here's it formally:

```ts
const longHabits: Habit[] = habits.filter((h) => h.name.length > 10);
```

Read aloud: *"filter habits to those whose name is longer than ten characters."*

The function is called a **predicate** — it returns a boolean. Items where the predicate returns `true` are kept; the others are discarded.

> **predicate** *(noun)* — a function that returns a boolean. Used by `filter`, `find`, `some`, `every`.

---

## Lesson 11.3 — `array.find(pred)` — first match

```ts
const matched: Habit | undefined = habits.find((h) => h.name === 'Read');
```

Returns the first element where the predicate is true, or `undefined` if none match. *The `undefined` is important* — the type system makes you handle the missing case.

```ts
if (matched === undefined) {
  // no match — handle it
  return;
}

// from here on, `matched` is `Habit`
console.log(matched.name);
```

That's the pattern. The compiler narrows the type after the early return.

---

## Lesson 11.4 — `some` and `every` — boolean reductions

```ts
const anyAddedToday: boolean = habits.some((h) => isToday(h.createdAt));
const allNamed: boolean = habits.every((h) => h.name !== '');
```

- **`some(pred)`** — `true` if at least one element matches.
- **`every(pred)`** — `true` if every element matches. (Vacuously `true` for an empty array — be aware.)

---

## Lesson 11.5 — `array.includes(value)` — for primitives

```ts
const isWeekend: boolean = ['Sat', 'Sun'].includes(today);
```

Returns `true` if the array contains the exact value. Works for primitives (`string`, `number`, `boolean`); for objects, you usually want `some` with a comparison.

---

## Lesson 11.5b — `array.reduce(fn, initial)` — the heavy lifter

`map` returns an array of the same length. `filter` returns a subset. `reduce` returns *one thing* — a single value computed from every element.

```ts
const total: number = [1, 2, 3, 4].reduce((sum, n) => sum + n, 0);
// total is 10
```

Read aloud: *"start with sum equal to zero. For each number, the new sum is the old sum plus the number. After the last number, return the final sum."*

The first argument to `reduce` is the **accumulator function** `(acc, item) => nextAcc`. The second argument is the **initial value** of the accumulator — pass it explicitly always. (Without an initial, JavaScript uses the first element, which surprises you when the array is empty or when you want a different *type* than the elements.)

> **`array.reduce(fn, initial)`** — fold a list down to one value. The most powerful array method; also the one beginners fear. The mental model: *"keep an accumulator; for each item, update it; return what you have at the end."*

Three useful shapes:

```ts
// 1. Sum the lengths of every habit name.
const totalNameLength: number = habits.reduce(
  (sum, h) => sum + h.name.length,
  0,
);

// 2. Find the oldest habit (smallest createdAt). Destructure-and-narrow,
//    Bible rule #3: never habits[0]!.
const [first, ...rest] = habits;
const oldest: Habit | undefined = first === undefined
  ? undefined
  : rest.reduce(
      (oldest, h) => (h.createdAt < oldest.createdAt ? h : oldest),
      first,
    );

// 3. Group by name's first letter — accumulator is an object, very
//    different shape from elements. Immutable update inside the reducer.
const byInitial: Record<string, Habit[]> = habits.reduce<Record<string, Habit[]>>(
  (acc, habit) => ({
    ...acc,
    [habit.name[0] ?? '?']: [...(acc[habit.name[0] ?? '?'] ?? []), habit],
  }),
  {},
);
```

The `reduce<Record<string, Habit[]>>(...)` form is *generic call notation* — we pin the accumulator's type because TS can't infer it from `{}` alone. You'll see this in the wild; remember it.

(Mutating `acc` inside the reducer with `acc[key].push(...)` would also work, and is faster for huge arrays. We use the immutable form because it composes with `$derived` predictably and matches the Ch 8 spread-rule. For 50 habits the perf difference is unmeasurable.)

We meet `reduce` in real use in Ch 16 (computing totals from `$derived`) and Ch 30 (`splitCents` for refunds).

---

## Lesson 11.6 — Wiring it into Streak

We add a search box that filters the visible habits. Update the script:

```ts
let searchQuery: string = $state('');

const visibleHabits = $derived(
  habits.filter((h) => h.name.toLowerCase().includes(searchQuery.toLowerCase()))
);
```

> **`$derived(expression)`** *(preview — full coverage Ch 16)* — a Svelte 5 rune that computes a value from other reactive state and recomputes when those dependencies change. Read it as *"`visibleHabits` is whatever you'd get by running this filter every time, but Svelte tracks the work for you."* The `expression` form (one line) is shown here; the multi-line `$derived.by(() => { ... })` form lands in Ch 16.

In markup:

```svelte
<input type="search" bind:value={searchQuery} placeholder="Search habits..." />

{#each visibleHabits as habit (habit.id)}
  <!-- ...row... -->
{/each}
```

Read aloud: *"the visible habits are those whose name (lowercase) contains the query (lowercase). For each visible habit, render a row."*

Save (`Cmd+S` / `Ctrl+S`). Type *"read"* — only the *Read 20 minutes* habit appears. Clear the search — all habits return.

> **`.toLowerCase()`** — string method returning a lowercase copy. We compare both sides lowercase so search is case-insensitive — a senior UX habit.
>
> **`<input type="search">`** — an input meant for search queries. Browsers render an X-to-clear button and apply slightly different styling. Behaves the same as `type="text"` for binding.

We *could* have written the filter inline inside `{#each}` — and many tutorials do — but pulling it into a named `$derived` makes the value searchable, testable, and self-documenting. Senior habit: name your derived values.

---

## Lesson 11.7 — Read this code

### Snippet A

```ts
const numbers: number[] = [1, 2, 3, 4, 5];
const evens: number[] = numbers.filter((n) => n % 2 === 0);
const doubled: number[] = evens.map((n) => n * 2);
console.log(doubled);
```

<details>
<summary>Answer</summary>

`[4, 8]`. Filter to evens (`2`, `4`); double each (`4`, `8`).

The `%` is the *modulo* operator: `n % 2` is the remainder when `n` is divided by 2 — `0` for evens, `1` for odds.
</details>

### Snippet B

```ts
const users = [{ name: 'Alice' }, { name: 'Bob' }, { name: 'Charlie' }];
const found = users.find((u) => u.name === 'Bob');
console.log(found?.name ?? 'not found');
```

<details>
<summary>Answer</summary>

`Bob`. `find` returns the matching user; `?.name` reads `name`; `??` is unused because `name` is present.

If the search were for `'David'`, `found` would be `undefined`, `found?.name` would short-circuit to `undefined`, and `??` would supply `'not found'`.
</details>

---

## Lesson 11.8 — Now you write it

**The English sentence first:**

> *"Show a small `<small>` near the top of the list saying 'Showing X of Y' when a search is active. When the search is empty, hide it."*

<details>
<summary>Worked answer</summary>

We already have `visibleHabits` as a `$derived` from Lesson 11.6, so the indicator is just `visibleHabits.length` vs `habits.length`. In markup:

```svelte
{#if searchQuery.trim() !== ''}
  <small>Showing {visibleHabits.length} of {habits.length}</small>
{/if}
```

The fact that we already had a named derived value made this a one-liner. That's what *naming* derived values buys you: every later read is free.

> **`{@const}`** — a Svelte-template directive that creates a local constant scoped to a block (e.g. inside an `{#each}` body). Useful for ad-hoc computations you don't want to elevate into a top-level `$derived`. We won't need it for this exercise but you'll see it in the wild.
</details>

---

## Lesson 11.9 — Recurring concepts from earlier chapters

- **`bind:value`** (Ch 8) — used on the search input.
- **Predicate functions** (Ch 8 preview) — `filter`, `find`, `some`, `every` all take them.
- **`{#if cond}...{/if}`** (Ch 6) — wrapping the "Showing X of Y" indicator.
- **`.trim() === ''`** (Ch 4) — checking the search query is non-empty.
- **`Habit` type** (Ch 9) — `habits.filter(...)` returns `Habit[]`, fully typed.

---

## Lesson 11.10 — What you can now read in the wild

After Chapter 11 you can:

- Read **`xs.map((x) => fn(x))`** — transform.
- Read **`xs.filter((x) => pred(x))`** — keep matching.
- Read **`xs.find((x) => pred(x))`** — first match or undefined; remember to narrow.
- Read **`xs.some(pred)`** / **`xs.every(pred)`** — boolean reductions.
- Read **`xs.filter(...).map(...)`** chains and trace data flow.
- Read **`xs.reduce(fn, initial)`** as a fold and tell the accumulator type from the initial value.
- Read **`{@const x = expr}`** as a local constant in markup.

---

## Glossary added in Chapter 11

| Term | Definition |
|---|---|
| `array.map(fn)` | New array of `fn` applied to each element. |
| `array.filter(pred)` | New array of elements satisfying `pred`. |
| `array.find(pred)` | First match or `undefined`. |
| `array.some(pred)` | Boolean: any matches? |
| `array.every(pred)` | Boolean: all match? |
| `array.includes(v)` | Boolean: value present? (primitives) |
| `array.reduce(fn, init)` | Fold a list to one value via an accumulator. |
| predicate | A boolean-returning function. |
| `.toLowerCase()` | Lowercase copy of a string. |
| `%` | Modulo (remainder) operator. |
| `{@const}` | Svelte's local constant in markup. |
| `<input type="search">` | Search-styled text input with built-in clear button. |

---

## End-of-chapter checkpoint

- [ ] Search filters the visible habits as you type.
- [ ] You can read each of `map`, `filter`, `find`, `some`, `every`, `includes` aloud.
- [ ] (After exercise) the "Showing X of Y" indicator works.

---

# Chapter 12 — `const` by default — Part II checkpoint

> *Today's job:* audit every `let` in the codebase. Visible win: `pnpm check` reports zero warnings; many `let`s become `const`.

---

## Lesson 12.1 — The `const` rule, formally

The rule, in one sentence: **`const` until the compiler complains.**

Anything that is reassigned needs `let`. Everything else uses `const`.

`const` does *not* mean *"the value is immutable"* — it means *"the name will not be reassigned to point at a different value."* You can still mutate the contents of an object stored in a `const`:

```ts
const user = { name: 'Billy' };
user.name = 'Bobby'; // this works — we're mutating the object, not reassigning the name
user = { name: 'Charlie' }; // ❌ this fails — reassigning the name is forbidden
```

For mostly-frozen things like our `Habit` records, we use `const`. For `$state` variables we *will* reassign (like the `habits` array), we use `let`.

Bigger sweep: look at our current `+page.svelte`. Every `function`, every helper, every local variable. Most could be `const`. Anything declared with `let` should genuinely be reassigned somewhere.

> **`prefer-const`** — an ESLint rule that flags `let` declarations that are never reassigned. Our scaffold turned it on. Run `pnpm exec eslint src/` to see warnings.

---

## Lesson 12.2 — Run the checker

```bash
pnpm check
```

This runs `svelte-check`, which type-checks every `.svelte` and `.ts` file. The output should be:

```
====================================
svelte-check found 0 errors and 0 warnings
====================================
```

If you see warnings, read them. Fix them. The "no warnings" state is the senior baseline — we don't ship code with type warnings, ever.

(The `sv` scaffolder ships the `check` script in `package.json` already. If for some reason yours doesn't, add it manually:

```json
{
  "scripts": {
    "check": "svelte-kit sync && svelte-check --tsconfig ./tsconfig.json"
  }
}
```
)

---

## Lesson 12.3 — Read this code (and audit it)

```ts
let count = 0;
let title = 'Streak';
let isLoaded = false;

if (someCondition) {
  count = 5;
  isLoaded = true;
}

console.log(title, count, isLoaded);
```

Which `let`s should be `const`? Why?

<details>
<summary>Answer</summary>

`title` is never reassigned — it should be `const`.

`count` and `isLoaded` are both reassigned inside the `if`, so they need `let`. (Or, more elegantly, both could be `const` if you compute them inline: `const count = someCondition ? 5 : 0;`. Senior habit: when a `let` variable's value is determined by a single conditional, refactor to a `const` with the conditional inline.)
</details>

---

## Lesson 12.4 — The Part II out-loud recap

Sit with the entire `+page.svelte` and `formatRelativeTime.ts`. Read both out loud, top to bottom, no notes.

Specifically be able to say:
- What `type Habit = { ... }` declares.
- What `const habits: Habit[] = $state(...)` does (note: we reassign `habits = ...` in our handlers — so it's `let`, not `const`. This is one of the few `let`s we have).
- What `crypto.randomUUID()` returns.
- What `Date.now()` returns.
- What `array.filter`, `array.map`, `array.find` do.
- What `?.` and `??` mean and when to use each.
- What destructuring in function parameters looks like.

If you're confident, you've completed Part II. Streak is now: a list of typed habits, addable, removable, searchable, persisted *only in memory* (we'll fix that in Part VI).

---

## Lesson 12.5 — Now you write it

**The English sentence first:**

> *"Look at every variable in `+page.svelte`. Convert to `const` everywhere `let` isn't strictly required. Run `pnpm check`. Commit the cleanup."*

(We don't formally introduce git/commits until Chapter 61, but if you already use git, this is a fine first commit.)

---

## Lesson 12.6 — Recurring concepts from earlier chapters

Part II's spine, in one place:

- **`$state`** (Ch 1) — only for values that genuinely change.
- **Type annotations** (Ch 4, 9) — `string[]`, `Habit[]`, custom types.
- **Immutable updates** (Ch 8) — spread for add, filter for remove.
- **Helper functions in `$lib/`** (Ch 9) — `formatRelativeTime` is a textbook example.
- **The `const` rule** — formalised here.

---

## Lesson 12.7 — What you can now read in the wild

After Part II you can:

- Read any modern Svelte 5 component that uses one or two `$state` arrays of typed records and explain it line by line.
- Spot when a `let` should be a `const`.
- Spot when an immutable update is needed vs an in-place mutation.
- Recognise the *helper-in-`$lib/`* pattern and its testability benefit.
- Read most TypeScript object-shape declarations and explain optional vs required fields.

You're ready to break the page into components.

---

## Glossary added in Chapter 12

| Term | Definition |
|---|---|
| `const` rule | Default to `const`; use `let` only when reassignment is real. |
| `prefer-const` | ESLint rule flagging unnecessary `let`. |
| `pnpm check` | Run `svelte-check` over the project. |

---

## End-of-Part-II checkpoint

- [ ] Habits are real records with id/name/description/createdAt.
- [ ] Add and remove work; same-name habits don't collide.
- [ ] Search filters the list.
- [ ] `pnpm check` reports zero errors and zero warnings.
- [ ] You read your codebase out loud and explained every line.

You're ready for Part III: components.
