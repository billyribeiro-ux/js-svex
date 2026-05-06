# Part III — Components, props, snippets, and the shape of an app

> *"By the end of Part III the home screen is a component tree. The reader has met every Svelte 5 rune used in modern apps and has internalised: prove `$derived` can't do it before you reach for `$effect`."*

---

# Chapter 13 — Your first component

> *Today's job:* extract the habit row into its own file. Visible win: identical UI; the file tree gains `src/lib/components/HabitRow.svelte`.

The home page works. It also has 100+ lines crammed into one file. Senior engineers smell that — *one file should do one thing*. The habit-row markup is its own concept; pull it out.

---

## Lesson 13.1 — What a component is

A **component** is a self-contained piece of UI in its own `.svelte` file. Other components (and pages) use it like a custom HTML tag.

```svelte
<HabitRow habit={someHabit} />
```

Read aloud: *"render a HabitRow with this habit."*

Components let you:
- name a piece of UI (`HabitRow` is more meaningful than `<li>` with stuff inside),
- reuse it (the same row could appear on the home page and the detail page),
- test it in isolation (we'll do this in Chapter 59),
- replace it without touching anything else.

> **component** *(noun)* — a reusable piece of UI in its own `.svelte` file.

---

## Lesson 13.2 — `src/lib/components/`

Components live in `src/lib/components/`. Naming convention: **PascalCase**, ending in `.svelte`. So: `HabitRow.svelte`, `EmptyState.svelte`, `HabitList.svelte`.

> **PascalCase** — `LikeThis`. (As opposed to camelCase: `likeThis`.) Used for components and types.

Create the file:

```bash
mkdir -p src/lib/components
```

…then in your editor create `src/lib/components/HabitRow.svelte`:

```svelte
<!-- src/lib/components/HabitRow.svelte -->
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';

  type Habit = {
    id: string;
    name: string;
    description?: string;
    createdAt: number;
  };

  // For now we hardcode the habit. We'll meet $props in Chapter 14 and pass it in.
  // We also re-declare `Habit` locally here; Chapter 14 moves it to $lib/types so
  // every component shares one definition.
  const habit: Habit = {
    id: 'demo',
    name: 'Demo habit',
    createdAt: Date.now(),
  };
</script>

<li>
  <div>
    <strong>{habit.name}</strong>
    {#if habit.description}
      <p class="habit-desc">{habit.description}</p>
    {/if}
    <small>{formatRelativeTime(habit.createdAt)}</small>
  </div>
</li>

<style>
  li {
    display: flex; justify-content: space-between; align-items: center;
    padding: 0.5rem 0.75rem; background: #f5f5f5; margin: 0.25rem 0; border-radius: 0.25rem;
  }
  .habit-desc { margin: 0.125rem 0; color: #666; font-size: 0.875rem; }
</style>
```

This is a *self-contained* component. It has its own script, its own markup, its own CSS. Its CSS only affects elements inside it (Svelte scopes styles per component automatically).

But it's not yet useful — it has hardcoded data. Chapter 14 fixes that.

---

## Lesson 13.3 — Importing and using

In `+page.svelte`, at the top of the script:

```ts
import HabitRow from '$lib/components/HabitRow.svelte';
```

And in markup, replace the contents of the `<ul>` with:

```svelte
<ul class="habit-list">
  {#each habits as habit (habit.id)}
    <HabitRow />
  {/each}
</ul>
```

(All habits will look identical because `HabitRow` doesn't accept the `habit` prop yet. That's Chapter 14.)

---

## Lesson 13.4 — Now you write it

**The English sentence first:**

> *"Extract the empty-state card into `src/lib/components/EmptyState.svelte`. The page just renders `<EmptyState />` when `habits.length === 0`."*

<details>
<summary>Worked answer</summary>

Create `src/lib/components/EmptyState.svelte`:

```svelte
<!-- src/lib/components/EmptyState.svelte -->
<div class="empty-state">
  <h2>No habits yet</h2>
  <p>Add your first one above.</p>
</div>

<style>
  .empty-state {
    text-align: center; padding: 2rem; margin: 2rem 0;
    border: 2px dashed #ccc; border-radius: 0.5rem;
  }
  .empty-state h2 { margin-top: 0; color: #666; }
  .empty-state p { color: #888; }
</style>
```

In `+page.svelte`:

```ts
import EmptyState from '$lib/components/EmptyState.svelte';
```

```svelte
{#if habits.length === 0}
  <EmptyState />
{:else}
  ...
{/if}
```
</details>

---

## Lesson 13.5 — Recurring concepts from earlier chapters

- **`{#each}` keyed** (Ch 9) — the home page still iterates over `habits` by ID.
- **`{#if cond}{:else}{/if}`** (Ch 6) — the empty-state branch.
- **Scoped `<style>`** (Ch 5) — the row's CSS doesn't leak.
- **`type Habit`** (Ch 9) — re-declared locally for now; centralised in Ch 14.

---

## Lesson 13.6 — What you can now read in the wild

After Chapter 13 you can:

- Read **`<Component />`** as a self-contained UI element.
- Read **`import X from '$lib/components/X.svelte'`** as a component import.
- Recognise the *PascalCase filename = component* convention.
- Read a small `.svelte` file with script + markup + scoped style and explain each section.

---

## End-of-chapter checkpoint

- [ ] `HabitRow.svelte` and `EmptyState.svelte` exist.
- [ ] The home page imports and uses them.
- [ ] The page still works (with `HabitRow` showing demo data — fine, we fix in Ch 14).

---

# Chapter 14 — `$props` — talking to your component

> *Today's job:* `HabitRow` receives a `habit` prop instead of having a hardcoded one. Visible win: each habit row shows its real data.

---

## Lesson 14.1 — `$props()`, formally

Replace the hardcoded constant in `HabitRow.svelte`:

```svelte
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';

  type Habit = {
    id: string;
    name: string;
    description?: string;
    createdAt: number;
  };

  let { habit }: { habit: Habit } = $props();
</script>
```

Read aloud: *"declare a prop called `habit` of type Habit."*

> **`$props()`** — a Svelte 5 rune that returns this component's props as an object. You destructure them and (mandatorily, in TypeScript) annotate their types.

Two important rules:

1. **You don't mutate props.** A child component should never reassign or modify its parent's data. (The `$bindable` exception is Chapter 18.)
2. **Defaults go in destructuring.** `let { habit, compact = false } = $props<...>();`

---

## Lesson 14.2 — Sharing the `Habit` type

Right now `Habit` is defined in three places: `+page.svelte`, `HabitRow.svelte`, and any future component that touches it. That's bad — if we add a field, we have to remember to update every copy.

Move it to a shared module. Create `src/lib/types.ts`:

```ts
// src/lib/types.ts
export type Habit = {
  id: string;
  name: string;
  description?: string;
  createdAt: number;
};
```

Then in any component:

```ts
import type { Habit } from '$lib/types';
```

The `import type` is a TypeScript hint: *"this import is only used for types; the bundler can drop it."* Senior habit. Keeps bundles tiny.

> **`import type`** — TypeScript syntax for a type-only import. The runtime doesn't see it.

Update `+page.svelte` and `HabitRow.svelte` to import from `$lib/types`. Delete the local copies.

---

## Lesson 14.3 — Wiring the prop in

In `+page.svelte`'s markup:

```svelte
{#each habits as habit (habit.id)}
  <HabitRow {habit} />
{/each}
```

Read aloud: *"render a HabitRow with `habit={habit}`."*

The `{habit}` is shorthand for `habit={habit}` — when the attribute name and the variable name match, you don't have to write both. Senior idiom.

---

## Lesson 14.4 — Optional and defaulted props

You can have props that are optional or have defaults:

```ts
let {
  habit,
  compact = false,
  onSelect,
}: {
  habit: Habit;
  compact?: boolean;
  onSelect?: (id: string) => void;
} = $props();
```

`compact` defaults to `false` if the parent doesn't pass it. `onSelect` is optional — call sites with `onSelect={fn}` can pass it; others can omit.

---

## Lesson 14.5 — Read this code

```svelte
<!-- Button.svelte -->
<script lang="ts">
  let {
    label,
    variant = 'primary',
    onclick,
  }: {
    label: string;
    variant?: 'primary' | 'secondary' | 'danger';
    onclick: () => void;
  } = $props();
</script>

<button type="button" class="btn btn-{variant}" {onclick}>
  {label}
</button>
```

What props does this component take? Which are required?

<details>
<summary>Answer</summary>

- `label: string` — required
- `variant: 'primary' | 'secondary' | 'danger'` — optional, defaults to `'primary'`
- `onclick: () => void` — required

The variant is a *string union type* — only those three exact strings are allowed. Try to pass `variant="warning"` and TypeScript flags it.
</details>

---

## Lesson 14.6 — Now you write it

**The English sentence first:**

> *"Add a `compact?: boolean` prop to `HabitRow`. When compact is true, hide the description and the timestamp; show only the name."*

<details>
<summary>Worked answer</summary>

```svelte
<!-- HabitRow.svelte -->
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';
  import type { Habit } from '$lib/types';

  let {
    habit,
    compact = false,
  }: {
    habit: Habit;
    compact?: boolean;
  } = $props();
</script>

<li>
  <div>
    <strong>{habit.name}</strong>
    {#if !compact && habit.description}
      <p class="habit-desc">{habit.description}</p>
    {/if}
    {#if !compact}
      <small>{formatRelativeTime(habit.createdAt)}</small>
    {/if}
  </div>
</li>
```
</details>

---

## Lesson 14.7 — Recurring concepts from earlier chapters

- **Object destructuring** (Ch 10) — `let { habit } = $props()` is exactly the pattern.
- **Defaults on destructure** (Ch 10) — `compact = false`.
- **`type` alias + union types** (Ch 4, 9) — `variant?: 'primary' | 'secondary' | 'danger'`.
- **`$state` and reactivity** (Ch 1) — props are *reactive*; the parent passes them, the child re-renders when they change.

---

## Lesson 14.8 — What you can now read in the wild

After Chapter 14 you can:

- Read **`let { x, y, z = default } = $props<{ x: T; y: U; z?: V }>()`** as a fully-typed prop declaration. *(Note: in Svelte 5.x, the more idiomatic form is the inline type annotation we used: `let { x, y, z = default }: { x: T; y: U; z?: V } = $props();` — both work; the inline form is preferred.)*
- Read **`import type { X } from '...'`** as type-only imports.
- Read **`<Component {prop} />`** as the shorthand for `prop={prop}`.
- Spot **components defining `type X` locally** as a refactor opportunity (move to a shared `$lib/types.ts`).

---

## End-of-chapter checkpoint

- [ ] `Habit` lives in `src/lib/types.ts` and is imported everywhere.
- [ ] `HabitRow` receives a real habit prop and renders it correctly.
- [ ] `pnpm check` is green.

---

# Chapter 15 — Callback props — talking back to your parent

> *Today's job:* clicking × on `HabitRow` removes that habit from the parent's list — but the deletion logic still lives in the parent. Visible win: deletion works exactly as before; the responsibility lives in the right place.

---

## Lesson 15.1 — Data down, events up

A senior pattern, named:

> **Data down, events up** — props flow from parent to child; events flow from child back to parent. The child doesn't reach into the parent's state directly.

In Svelte 5, the way a child sends an event up is by calling a *function prop* the parent passed in.

---

## Lesson 15.2 — Function props

Add `onDelete` to `HabitRow`:

```svelte
<!-- HabitRow.svelte -->
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';
  import type { Habit } from '$lib/types';

  let {
    habit,
    onDelete,
  }: {
    habit: Habit;
    onDelete: (id: string) => void;
  } = $props();
</script>

<li>
  <div>
    <strong>{habit.name}</strong>
    {#if habit.description}<p class="habit-desc">{habit.description}</p>{/if}
    <small>{formatRelativeTime(habit.createdAt)}</small>
  </div>
  <button type="button" onclick={() => onDelete(habit.id)} aria-label="Remove {habit.name}">×</button>
</li>
```

In `+page.svelte` (and confirm `removeHabit` takes an `id: string`):

```ts
function removeHabit(id: string): void {
  habits = habits.filter((h) => h.id !== id);
}
```

```svelte
{#each habits as habit (habit.id)}
  <HabitRow {habit} onDelete={removeHabit} />
{/each}
```

Read aloud: *"render a HabitRow with this habit; when it asks to delete, call `removeHabit` with the id."*

We standardise on `removeHabit(id: string): void` for the rest of the book — IDs are the natural identity for habits, and we'll lean on this in Part VI when habits live in a database.

---

## Lesson 15.3 — `on<Verb>` naming

Conventions matter. Function props are named `on<Verb>`:

- `onDelete`, `onSave`, `onSubmit`, `onClick`, `onChange`, `onSelect`, `onEdit`.

Not `deleteCallback`, not `handleDelete`, not `removeFn`. Just `on<Verb>`. This is what every senior codebase uses; matching the convention makes your code unsurprising to readers.

---

## Lesson 15.4 — *No* `createEventDispatcher`

Older Svelte tutorials show `createEventDispatcher`. **We never use it.** Reasons:

- It's the legacy API; Svelte 5 prefers function props.
- It's harder to type properly.
- It introduces a separate `dispatch('foo', payload)` syntax that's noisier than just calling a function.

If you see `createEventDispatcher` in old code, replace with a function prop the next time you touch that file.

---

## Lesson 15.5 — Read this code

```svelte
<!-- Modal.svelte -->
<script lang="ts">
  let {
    open,
    onClose,
  }: {
    open: boolean;
    onClose: () => void;
  } = $props();
</script>

{#if open}
  <!-- For brevity this snippet uses click on a div for the backdrop;
       a senior implementation uses a real <dialog> + showModal() (Ch 21)
       and lets the backdrop click be handled via the dialog's `cancel` event,
       so screen readers and keyboards work without extra wiring. -->
  <div class="modal-backdrop" onclick={onClose} role="presentation">
    <div class="modal-content" role="dialog" aria-modal="true" onclick={(e) => e.stopPropagation()}>
      <button type="button" onclick={onClose} aria-label="Close">×</button>
      <p>Modal content</p>
    </div>
  </div>
{/if}
```

What's the `e.stopPropagation()` for?

<details>
<summary>Answer</summary>

It prevents the click on `.modal-content` from bubbling up to `.modal-backdrop` (which would also fire `onClose`). Without it, clicking *inside* the modal would close it — which the user almost never wants.

> **event bubbling** — clicks on a child fire on every ancestor element too, by default. `stopPropagation()` cancels the bubble.

A senior habit when writing modals: think about both the click-the-backdrop-to-close and click-inside-doesn't-close behaviour explicitly.
</details>

---

## Lesson 15.6 — Now you write it

**The English sentence first:**

> *"Add an `onRename: (id: string, newName: string) => void` callback prop to `HabitRow`. When the user double-clicks the habit name, the name swaps to an editable input; pressing Enter (or blurring) commits the new name; pressing Escape cancels."*

We're deliberately *not* using `window.prompt`. Browser-native dialogs are unstyled, ignore your dark mode, and break flow on touch devices. The senior pattern is *inline editing* — swap the display element for an input, in place. This is the same pattern Notion, Linear, and Things use.

<details>
<summary>Worked answer</summary>

```svelte
<!-- HabitRow.svelte -->
<script lang="ts">
  import { formatRelativeTime } from '$lib/formatRelativeTime';
  import type { Habit } from '$lib/types';

  let {
    habit,
    onDelete,
    onRename,
  }: {
    habit: Habit;
    onDelete: (id: string) => void;
    onRename: (id: string, newName: string) => void;
  } = $props();

  let editing = $state(false);
  let draft = $state('');

  function startEdit(): void {
    draft = habit.name;
    editing = true;
  }

  function commit(): void {
    const trimmed = draft.trim();
    if (trimmed !== '' && trimmed !== habit.name) {
      onRename(habit.id, trimmed);
    }
    editing = false;
  }

  function cancel(): void {
    editing = false;
  }

  function onKeydown(event: KeyboardEvent): void {
    if (event.key === 'Enter') commit();
    if (event.key === 'Escape') cancel();
  }
</script>

<li>
  <div>
    {#if editing}
      <input
        type="text"
        bind:value={draft}
        onblur={commit}
        onkeydown={onKeydown}
        aria-label="Rename {habit.name}"
      />
    {:else}
      <strong ondblclick={startEdit} title="Double-click to rename">{habit.name}</strong>
    {/if}
    {#if habit.description}<p class="habit-desc">{habit.description}</p>{/if}
    <small>{formatRelativeTime(habit.createdAt)}</small>
  </div>
  <button type="button" onclick={() => onDelete(habit.id)} aria-label="Remove {habit.name}">×</button>
</li>
```

In `+page.svelte`:

```ts
function renameHabit(id: string, newName: string): void {
  habits = habits.map((h) => h.id === id ? { ...h, name: newName } : h);
}
```

```svelte
<HabitRow {habit} onDelete={removeHabit} onRename={renameHabit} />
```

Three senior touches in there:
- **Cancel via Escape** — every editable field on the web should support it; users learn this once and expect it everywhere.
- **No-op on identical text** — if the user double-clicks but doesn't change anything, we don't fire `onRename`. Saves a server round-trip later (Ch 41).
- **`aria-label` on the input** — when the editing state appears, screen readers know what's being edited.

`habits.map` is the immutable update — return a new object when the ID matches, original otherwise.

We'll meet the `<input>`-ref autofocus trick (so the cursor lands inside the field when editing starts) in Ch 21 with `bind:this`.
</details>

---

## Lesson 15.7 — Recurring concepts from earlier chapters

- **Function types** (Ch 9 preview) — `(id: string) => void` is a typed callback.
- **Arrow functions for parameterised handlers** (Ch 8) — `onclick={() => onDelete(habit.id)}`.
- **`aria-label`** (Ch 8) — accessibility on icon-only buttons.
- **The "data down, events up" pattern** — formalised here.

---

## Lesson 15.8 — What you can now read in the wild

After Chapter 15 you can:

- Read **callback props** named `on<Verb>` and explain the data-down events-up flow.
- Recognise **`createEventDispatcher`** in old code as a refactor target.
- Read **event bubbling** and **`stopPropagation()`** in modal-like UIs.
- Read **inline-edit patterns** (toggle between display and `<input>`) as the senior alternative to `window.prompt` / `window.confirm` / `window.alert` (which are banned in this book — unstyled, ignore dark mode, break on touch).

---

## End-of-chapter checkpoint

- [ ] Delete still works through the callback prop.
- [ ] (After exercise) double-click rename works.
- [ ] You can read `data down, events up` aloud and explain what it means.

---

# Chapter 16 — `$derived` — values that compute themselves

> *Today's job:* show "X habits added today" as a computed total. Visible win: add a habit; the count updates without manual recalculation. Then move the search filter from inline to `$derived` for cleanliness.

---

## Lesson 16.1 — The derived concept

A **derived** value is one computed from other reactive state. When the dependencies change, the derived value automatically recomputes.

```ts
let count: number = $state(0);
let doubled: number = $derived(count * 2);
```

Read aloud: *"let `doubled` be derived as `count * 2`."*

When `count` changes, `doubled` updates. You read it as a normal value: `console.log(doubled)`.

> **`$derived(expression)`** — a Svelte 5 rune for a value computed from reactive dependencies. Recomputes automatically when any read dependency changes.
>
> **`$derived.by(() => { ... })`** — a multi-line variant; the body is a function returning the derived value. Used when computing the value needs more than one expression.

---

## Lesson 16.2 — Wiring it into Streak

You met `$derived` as a preview in Ch 11.6 — `visibleHabits` is already a derived value from the search box. Now we add a *second* derived for the "X added today" indicator, plus the helper that powers it:

```ts
// in +page.svelte script
let searchQuery: string = $state('');

function isAddedToday(epochMs: number): boolean {
  const habitDate = new Date(epochMs);
  const today = new Date();
  return habitDate.getDate() === today.getDate() &&
         habitDate.getMonth() === today.getMonth() &&
         habitDate.getFullYear() === today.getFullYear();
}

const visibleHabits: Habit[] = $derived(
  habits.filter((h) => h.name.toLowerCase().includes(searchQuery.toLowerCase()))
);

const addedTodayCount: number = $derived(
  habits.filter((h) => isAddedToday(h.createdAt)).length
);
```

(Ordering note: helper functions go *above* the `$derived`s that use them. JavaScript hoists function declarations, so the *opposite* order also works — but reading top-to-bottom, it's kinder to the reader to see the helper first.)

In markup:

```svelte
<small>{addedTodayCount} added today</small>

{#each visibleHabits as habit (habit.id)}
  <HabitRow {habit} onDelete={removeHabit} />
{/each}
```

Save. Add a habit; both `visibleHabits` and `addedTodayCount` update. The derived expressions are *named* now, so they're searchable and explainable.

---

## Lesson 16.3 — `$derived.by` for multi-line logic

When the derivation needs setup or a loop:

```ts
const summary: string = $derived.by(() => {
  if (habits.length === 0) return 'No habits yet.';
  if (habits.length === 1) return `Tracking 1 habit.`;
  return `Tracking ${habits.length} habits.`;
});
```

Same as `$derived` but takes a function. Use whichever reads cleaner.

---

## Lesson 16.4 — The "don't sync state with `$effect`" rule, formally

Beginner-trap pattern:

```ts
// ❌ BAD
let total = $state(0);
$effect(() => {
  total = items.reduce((sum, item) => sum + item.amount, 0);
});
```

This works but it's wrong. The correct way:

```ts
// ✅ GOOD
const total: number = $derived(items.reduce((sum, item) => sum + item.amount, 0));
```

Reasons:
- `$derived` is *pull-based* — it computes on read, with caching.
- `$effect` is *push-based* — it runs eagerly after every dependency change.
- For computed values, pull is right; push wastes work.
- `$effect` is for *side effects* (DOM mutations, network calls, logging). It's not for computing values.

> **The rule:** *prove `$derived` cannot do the job before reaching for `$effect`*. If the answer is "I'm computing a value from other values" — `$derived`. If the answer is "I'm reaching out to the DOM, the network, or local storage" — `$effect`.

---

## Lesson 16.5 — Read this code

```svelte
<script lang="ts">
  let prices: number[] = $state([10, 20, 30]);
  let taxRate: number = $state(0.08);

  const subtotal: number = $derived(prices.reduce((s, p) => s + p, 0));
  const tax: number = $derived(subtotal * taxRate);
  const total: number = $derived(subtotal + tax);
</script>

<p>Subtotal: ${subtotal}</p>
<p>Tax: ${tax.toFixed(2)}</p>
<p>Total: ${total.toFixed(2)}</p>
```

If you push a `40` onto `prices`, what updates?

<details>
<summary>Answer</summary>

`subtotal` recomputes (now `100`), then `tax` (now `8`), then `total` (now `108`). All three update on the screen automatically. The chain of `$derived`s flows through.

This *cascading-derived* pattern is one of the things that makes Svelte 5 so pleasant — you describe relationships, the runtime keeps them consistent.
</details>

---

## Lesson 16.6 — Now you write it

**The English sentence first:**

> *"Add a derived `longestRunningHabit: Habit | undefined` — the habit with the smallest `createdAt`. Display 'Longest streak: NAME' when it exists; nothing when there are no habits."*

<details>
<summary>Worked answer</summary>

```ts
const longestRunningHabit: Habit | undefined = $derived.by(() => {
  const [first, ...rest] = habits;
  if (first === undefined) return undefined;
  return rest.reduce((oldest, h) => h.createdAt < oldest.createdAt ? h : oldest, first);
});
```

Read aloud: *"the longest-running habit is the one with the smallest createdAt. Pull off the first habit; if there isn't one, return undefined; otherwise reduce the rest, keeping whichever is older."*

The destructure `const [first, ...rest] = habits` is critical — it lets us hand `first` to `reduce` as the initial accumulator (a real `Habit`, not `undefined`), which keeps the types clean. We *never* reach for `habits[0]!` (the non-null assertion is banned, Bible rule #3) because there's always a clean alternative.

In markup:

```svelte
{#if longestRunningHabit}
  <small>Longest streak: {longestRunningHabit.name}</small>
{/if}
```
</details>

---

## Lesson 16.7 — Recurring concepts from earlier chapters

- **`array.filter`, `array.reduce`** (Ch 8, 11) — derived bodies are usually array-method chains.
- **Destructuring with rest** (Ch 10) — `const [first, ...rest] = habits` for safe head/tail.
- **`Habit` type** (Ch 14) — derived expressions stay type-safe end-to-end.

---

## Lesson 16.8 — What you can now read in the wild

After Chapter 16 you can:

- Read **`const x = $derived(expr)`** and **`const x = $derived.by(() => { ... })`** confidently.
- Spot **the `$effect`-to-sync-state antipattern** and rewrite as `$derived`.
- Read a chain of **cascading derived values** and trace updates from input to output.

---

## End-of-chapter checkpoint

- [ ] `visibleHabits`, `addedTodayCount` are named `$derived` values.
- [ ] You can articulate the "don't sync state with `$effect`" rule.

---

# Chapter 17 — `$effect`, the side-effect rune (used sparingly)

> *Today's job:* the "add habit" input autofocuses when the page loads. Visible win: cursor is in the input, ready to type, with no extra clicks.

This chapter teaches `$effect` and its strict framing: *prove `$derived` can't do it first*.

---

## Lesson 17.1 — When `$effect` is right

`$effect` is for **side effects** — code that touches the world outside this component:

- DOM operations (focusing an element, scrolling, measuring sizes).
- Browser APIs (`localStorage`, `window`, timers).
- Network calls.
- Logging to a service.

It is **not** for computing derived values. That's `$derived`.

```ts
$effect(() => {
  // side effect code here
  // dependencies are tracked from what's read in this body
});
```

---

## Lesson 17.2 — Autofocusing the input

Update `+page.svelte`:

```svelte
<script lang="ts">
  // ... existing ...

  let inputElement: HTMLInputElement | undefined = $state();

  $effect(() => {
    inputElement?.focus();
  });
</script>

<input type="text" bind:this={inputElement} bind:value={newHabit} placeholder="Add a new habit..." />
```

`bind:this={inputElement}` is Chapter 21's lesson previewed — it gets the DOM node into a variable. `inputElement?.focus()` calls `.focus()` *if* the element exists. The `$effect` fires when the component mounts (and the element is ready), and *would* re-fire if `inputElement` ever changed (it doesn't, in this case).

---

## Lesson 17.3 — The cleanup return

`$effect` can return a cleanup function:

```ts
$effect(() => {
  const interval = setInterval(() => console.log('tick'), 1000);
  return () => clearInterval(interval);
});
```

Read aloud: *"on mount, start a tick interval. On unmount (or before the effect re-runs), clear it."*

Senior habit: every time you set a `setInterval`, `setTimeout`, event listener, or subscription inside `$effect`, return the cleanup. Otherwise leaks.

---

## Lesson 17.4 — Dependency tracking: by *read*, not by mention

`$effect` tracks dependencies by what's *read synchronously* during its body. So:

```ts
$effect(() => {
  console.log(count); // count is a dep
});
```

Versus:

```ts
$effect(() => {
  setTimeout(() => console.log(count), 100); // count is NOT a dep — it's read inside async callback
});
```

The latter doesn't re-fire when `count` changes — the read is inside `setTimeout`'s callback, which runs asynchronously. Senior gotcha.

---

## Lesson 17.5 — The infinite-loop trap

```ts
let x = $state(0);
$effect(() => {
  x += 1; // ❌ infinite loop
});
```

`$effect` runs, reassigns `x`, which triggers `$effect` again, which reassigns `x`, ... Svelte 5 catches this at runtime and throws an `effect_update_depth_exceeded` error after a few iterations, so your dev server doesn't burn — but the page goes red and the lesson is loud.

Rule: **don't write to state inside `$effect` if the effect depends on that state.** If you find yourself wanting to, you almost certainly want `$derived`.

---

## Lesson 17.6 — Build, break, fix

Try the infinite-loop deliberately for a few seconds — Svelte will warn loudly. Restore. The discomfort is the lesson.

---

## Lesson 17.7 — Read this code

```svelte
<script lang="ts">
  let query: string = $state('');

  $effect(() => {
    if (query.length < 3) return;
    console.log('searching for', query);
  });
</script>

<input bind:value={query} />
```

When does the effect fire?

<details>
<summary>Answer</summary>

Whenever `query` changes — every keystroke. The `if (query.length < 3) return;` is an early-exit, but the effect *still ran*. The body is re-executed; we just bail early.

For real search, you'd debounce — wait for typing to pause before firing. We'll meet debouncing as we need it.
</details>

---

## Lesson 17.8 — Now you write it

**The English sentence first:**

> *"When the user has typed at least 3 characters in the search box, log 'searching for X' to the console. Don't log for shorter queries."*

<details>
<summary>Worked answer</summary>

```ts
$effect(() => {
  if (searchQuery.trim().length < 3) return;
  console.log('searching for', searchQuery);
});
```

This is a stand-in for analytics or a real search-API call. The early return guards both empty and too-short queries.
</details>

---

## Lesson 17.9 — Recurring concepts from earlier chapters

- **`$state(...)`** (Ch 1) — `inputElement` is reactive state; the effect re-runs when it appears.
- **Optional chaining `?.`** (Ch 4) — `inputElement?.focus()` is the safe DOM call.
- **Early return `return;`** (Ch 2) — guards inside the effect body.

---

## Lesson 17.10 — What you can now read in the wild

After Chapter 17 you can:

- Read **`$effect(() => { ... })`** and tell whether it should be a `$derived` instead.
- Read **`return () => cleanup;`** as the unmount/re-run cleanup.
- Spot **deps read inside async callbacks** as the *not-tracked* gotcha.
- Recognise the **`effect_update_depth_exceeded`** runtime error and trace it to a state-write inside an effect that reads the same state.

---

## End-of-chapter checkpoint

- [ ] The new-habit input autofocuses on page load.
- [ ] You can articulate when `$effect` is right and when `$derived` is right.
- [ ] You felt the infinite-loop trap deliberately and recovered.

---

# Chapter 18 — `$bindable` — when two-way binding is honest

> *Today's job:* a `<TextInput>` component used by the add-habit form (and later the rename form, the search box, etc.). Visible win: identical input UX everywhere.

---

## Lesson 18.1 — `$bindable()`

When you want a child component to participate in two-way binding (parent passes a value; child updates it; parent sees the change), the child marks the prop with `$bindable()`:

```svelte
<!-- src/lib/components/TextInput.svelte -->
<script lang="ts">
  let {
    value = $bindable(''),
    placeholder = '',
    type = 'text',
  }: {
    value?: string;
    placeholder?: string;
    type?: 'text' | 'email' | 'search';
  } = $props();
</script>

<input {type} bind:value {placeholder} />
```

The parent uses it with `bind:value`:

```svelte
<TextInput bind:value={newHabit} placeholder="Add a habit..." />
```

Without `$bindable`, the parent could pass `value={newHabit}` but the child couldn't update it back.

---

## Lesson 18.2 — When NOT to use `$bindable`

Most of the time, callback props are better:

```svelte
<!-- preferred: callback -->
<MyComponent value={someValue} onChange={(v) => someValue = v} />

<!-- only when input-like: $bindable -->
<TextInput bind:value={someValue} />
```

The senior heuristic: `$bindable` for *form inputs* (the parent and child genuinely co-own the value); callback props for everything else.

---

## Lesson 18.3 — Wiring it into Streak

Create `src/lib/components/TextInput.svelte` (above), then use it:

```svelte
<form onsubmit={handleSubmit}>
  <TextInput bind:value={newHabit} placeholder="Add a new habit..." />
  <button type="submit" disabled={newHabit.trim() === ''}>Add</button>
</form>

<TextInput type="search" bind:value={searchQuery} placeholder="Search habits..." />
```

(Remove the original `<input>` elements; they're replaced by `TextInput`.)

---

## Lesson 18.4 — Read this code

```svelte
<!-- NumberStepper.svelte -->
<script lang="ts">
  let {
    value = $bindable(0),
    min = 0,
    max = 100,
  }: { value?: number; min?: number; max?: number } = $props();

  function increment(): void {
    if (value >= max) return;
    value += 1;
  }
  function decrement(): void {
    if (value <= min) return;
    value -= 1;
  }
</script>

<button type="button" onclick={decrement} disabled={value <= min}>−</button>
<span>{value}</span>
<button type="button" onclick={increment} disabled={value >= max}>+</button>
```

Used as `<NumberStepper bind:value={count} min={0} max={10} />`. What does the parent see when the user clicks +?

<details>
<summary>Answer</summary>

`count` updates from the parent's perspective. The child's `value += 1` propagates back through the bind. Parent sees the new number.
</details>

---

## Lesson 18.5 — Now you write it

**The English sentence first:**

> *"Build `NumberStepper` (the snippet above), wire it to a `$state` count in the home page, click + and − buttons, watch both the parent's `count` and the child's display update in lockstep."*

<details>
<summary>Worked answer</summary>

`src/lib/components/NumberStepper.svelte` — exactly the snippet from Lesson 18.4. Save.

In `+page.svelte` (temporarily — remove after the exercise):

```svelte
<script lang="ts">
  import NumberStepper from '$lib/components/NumberStepper.svelte';
  let demoCount = $state(3);
</script>

<p>Demo count: {demoCount}</p>
<NumberStepper bind:value={demoCount} min={0} max={10} />
<button type="button" onclick={() => demoCount = 0}>Reset from parent</button>
```

The reset button is the test: click it, watch the child's `<span>{value}</span>` snap back to 0. That's two-way binding *both ways* — child writes flow up, parent writes flow down.

Senior takeaway: `$bindable` doesn't mean "the child owns the state." It means "the child and parent share one address." The single source of truth is still the parent's `let demoCount = $state(3)`.
</details>

---

## Lesson 18.6 — Recurring concepts from earlier chapters

- **Callback props** (Ch 15) — the *default* tool; `$bindable` is the *exception*.
- **Defaults on destructure** (Ch 10) — `$bindable('')` provides a default.
- **Guard clauses** (Ch 2) — `NumberStepper`'s `if (value >= max) return;`.

---

## Lesson 18.7 — What you can now read in the wild

After Chapter 18 you can:

- Read **`let { value = $bindable() } = $props()`** in a child.
- Read **`<TextInput bind:value={x} />`** in a parent.
- Tell *when* `$bindable` is right (input-shaped two-way) vs callback props (everything else).

---

## End-of-chapter checkpoint

- [ ] `TextInput` component exists and is used in two places.
- [ ] You wrote a `NumberStepper` and felt two-way binding.
- [ ] You can explain when `$bindable` is right vs callback props.

---

# Chapter 19 — `$inspect`, dev tools, reading errors fluently

> *Today's job:* use `$inspect` to investigate a deliberately-introduced bug. Visible win: you fix a bug you didn't write.

---

## Lesson 19.1 — `$inspect(value)`

```ts
$inspect(searchQuery);
```

In dev mode, this logs to the console *every time* `searchQuery` changes, with a stack trace pointing at the cause. In production, it's a no-op.

You can inspect multiple values:

```ts
$inspect(searchQuery, habits.length);
```

You can customise the log:

```ts
$inspect(searchQuery).with((type, value) => {
  if (type === 'update') console.log('search →', value);
});
```

And you can trace what *caused* a derived/effect to re-run:

```ts
const visibleHabits = $derived.by(() => {
  $inspect.trace();
  return habits.filter(/* ... */);
});
```

`$inspect.trace()` logs the dependencies that caused this derivation to re-run. Magical for debugging *"why did this re-fire?"*.

---

## Lesson 19.2 — Dev tools tour

Press F12 in your browser to open dev tools.

- **Console** — where `console.log` and `$inspect` print. Errors appear here too.
- **Sources** — your code, line by line. Set breakpoints by clicking line numbers.
- **Network** — every HTTP request the page makes. Filter by `Fetch/XHR`.
- **Application** — `localStorage`, cookies, service workers (we'll revisit in Ch 35, 38, 55).

The Console alone gets you 80% of the way. Get familiar with it.

---

## Lesson 19.3 — A planted bug

Replace your `removeHabit` with this broken version:

```ts
function removeHabit(id: string): void {
  habits = habits.filter((h) => h.id === id); // ❌ bug — should be !==
}
```

Save. Click × on a habit. *Every other* habit disappears, leaving only the one you tried to delete. The opposite of what we wanted.

Add an `$inspect`:

```ts
$inspect(habits);
```

Click × again. Watch the console — you'll see `habits` change to a 1-element array. Read the trace. The single remaining element is the one you tried to delete. *That's the symptom.* Look at `removeHabit`'s comparison. `===` instead of `!==`. Fix.

---

## Lesson 19.4 — Reading TypeScript errors fluently

A common error early on:

```
src/routes/+page.svelte:42:5 - Argument of type 'string | undefined' is not assignable to parameter of type 'string'.
  Type 'undefined' is not assignable to type 'string'.
```

Decode it:
- **`src/routes/+page.svelte:42:5`** — file, line 42, column 5.
- **The error in plain English** — "you passed something that might be `undefined` to a function expecting a `string`."
- **The fix** — narrow the value first (`if (x === undefined) return;`) or provide a fallback (`x ?? ''`).

Senior habit: read the error *out loud, slowly*. The TypeScript compiler is a tutor; rushing past errors is throwing away free help.

---

## Lesson 19.5 — Now you write it

**The English sentence first:**

> *"Add `$inspect.trace()` inside `visibleHabits`. Type a query — read the trace in the console — explain what dependencies changed."*

<details>
<summary>Worked answer</summary>

In `+page.svelte`'s script:

```ts
const visibleHabits: Habit[] = $derived.by(() => {
  $inspect.trace();
  return habits.filter((h) => h.name.toLowerCase().includes(searchQuery.toLowerCase()));
});
```

(The `$derived` form-conversion to `$derived.by` is required because `$inspect.trace()` is a statement, not part of the expression.)

Now type "rea" in the search box. Open the console. Each keystroke logs a trace like:

```
{
  dependencies: ['searchQuery', 'habits'],
  changed: ['searchQuery'],
}
```

The first line names *every* reactive value the derivation read; the `changed` line names which one(s) just updated. For our case: typing only changes `searchQuery`; `habits` is stable.

Senior takeaway: `$inspect.trace()` is the answer to *"why did this re-fire?"* — the most common debugging question once you have more than three derived values.
</details>

---

## Lesson 19.6 — Recurring concepts from earlier chapters

- **Build-break-fix** (Ch 6) — same drill, now with a real debugging tool.
- **`array.filter`** (Ch 8) — the planted bug is one wrong character in a filter.
- **TypeScript errors as tutors** — formalised here.

---

## Lesson 19.7 — What you can now read in the wild

After Chapter 19 you can:

- Read **`$inspect(x)`** and **`$inspect.trace()`** as debugging primitives.
- Open dev tools and tell which tab does what.
- Decode a TypeScript error message line by line.

---

## End-of-chapter checkpoint

- [ ] You used `$inspect` to spot a bug.
- [ ] You read at least one TypeScript error aloud and decoded it.
- [ ] You know which dev-tools tab to open for the console.

---

# Chapter 20 — Snippets and `{@render}`

> *Today's job:* a reusable `<Card>` with header, body, footer. Visible win: every habit row and every empty-state card uses the same chrome.

---

## Lesson 20.1 — What a snippet is

A **snippet** is *"a function that returns markup."* You define one with `{#snippet name(args)}...{/snippet}` and render it with `{@render name(args)}`.

```svelte
{#snippet greeting(name)}
  <p>Hello, {name}!</p>
{/snippet}

{@render greeting('Billy')}
{@render greeting('Bobby')}
```

That renders two paragraphs.

> **snippet** *(noun)* — a Svelte 5 reusable markup chunk, defined inline or as a prop. Replaces slots from Svelte 4.
>
> **`{@render}`** — invoke a snippet at a location in markup.

---

## Lesson 20.2 — `children` snippet — the default content

When a parent puts content *between* a component's opening and closing tags, that content is automatically a snippet called `children`:

```svelte
<!-- Card.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  let { children }: { children: Snippet } = $props();
</script>

<div class="card">
  {@render children()}
</div>
```

```svelte
<Card>
  <h2>Hello</h2>
  <p>Inside a card.</p>
</Card>
```

The `<h2>` and `<p>` become the `children` snippet; `{@render children()}` renders them inside `.card`.

> **`Snippet<T>`** — TypeScript type for a snippet that takes parameters of types `T`. From `'svelte'`. Use `Snippet` (no params) or `Snippet<[Habit]>` (one Habit param).

---

## Lesson 20.3 — Named snippet props

You can have several snippets:

```svelte
<!-- Card.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  let {
    header,
    children,
    footer,
  }: {
    header?: Snippet;
    children: Snippet;
    footer?: Snippet;
  } = $props();
</script>

<div class="card">
  {#if header}<div class="card-header">{@render header()}</div>{/if}
  <div class="card-body">{@render children()}</div>
  {#if footer}<div class="card-footer">{@render footer()}</div>{/if}
</div>

<style>
  .card { border: 1px solid #ddd; border-radius: 0.5rem; overflow: hidden; }
  .card-header { padding: 0.5rem 1rem; background: #f5f5f5; font-weight: 600; }
  .card-body { padding: 1rem; }
  .card-footer { padding: 0.5rem 1rem; background: #fafafa; font-size: 0.875rem; color: #666; }
</style>
```

Used as:

```svelte
<Card>
  {#snippet header()}<span>Today's habits</span>{/snippet}
  <ul>{#each visibleHabits as h (h.id)}<HabitRow habit={h} onDelete={removeHabit} />{/each}</ul>
  {#snippet footer()}<small>{addedTodayCount} added today</small>{/snippet}
</Card>
```

Read aloud: *"a Card. Its header is the title. Its body is the list. Its footer is the count."*

---

## Lesson 20.4 — Snippets with parameters

```svelte
{#snippet field(label: string, value: string)}
  <div class="field">
    <label>{label}</label>
    <span>{value}</span>
  </div>
{/snippet}

{@render field('Name', habit.name)}
{@render field('Created', formatRelativeTime(habit.createdAt))}
```

Snippets can be exported from `<script module>` and imported elsewhere too — like component-shaped utilities for repeated markup.

---

## Lesson 20.5 — Read this code

```svelte
<!-- DataList.svelte -->
<script lang="ts">
  import type { Snippet } from 'svelte';
  let {
    items,
    item,
    empty,
  }: {
    items: unknown[];
    item: Snippet<[unknown]>;
    empty?: Snippet;
  } = $props();
</script>

{#if items.length === 0}
  {#if empty}{@render empty()}{:else}<p>Empty</p>{/if}
{:else}
  <ul>
    {#each items as i}<li>{@render item(i)}</li>{/each}
  </ul>
{/if}
```

What does this component do?

<details>
<summary>Answer</summary>

It's a generic list renderer. Caller passes `items` (any array) and an `item` snippet (how to render each one). Optional `empty` snippet for the empty state. We'll meet a typed version with generics in Chapter 24.
</details>

---

## Lesson 20.6 — Now you write it

**The English sentence first:**

> *"Wrap the habit list in a `<Card>` with a header snippet showing 'Today's habits' and a footer snippet showing the count added today."*

<details>
<summary>Worked answer</summary>

```svelte
<!-- in +page.svelte -->
<script lang="ts">
  import Card from '$lib/components/Card.svelte';
  // ... existing imports + state ...
</script>

<Card>
  {#snippet header()}
    <span>Today's habits</span>
  {/snippet}

  <ul class="habit-list" bind:this={listEl}>
    {#each visibleHabits as habit (habit.id)}
      <HabitRow {habit} onDelete={removeHabit} />
    {/each}
  </ul>

  {#snippet footer()}
    <small>{addedTodayCount} added today</small>
  {/snippet}
</Card>
```

The list is the unnamed default content (becomes the `children` snippet). Header and footer are explicitly named. The component renders `<div class="card-header">...</div><div class="card-body">...</div><div class="card-footer">...</div>` chrome around them.

Senior takeaway: snippets and slot-style composition are *the same conceptual move* — letting a parent inject markup into named holes a child reserves. Snippets are typed and parameterisable; legacy slots weren't.
</details>

---

## Lesson 20.7 — Recurring concepts from earlier chapters

- **Component composition** (Ch 13) — snippets are the *internal* version of components.
- **Optional props** (Ch 14) — `header?: Snippet`, `footer?: Snippet`.
- **`{#if cond}{/if}`** (Ch 6) — guard whether to render a snippet block.

---

## Lesson 20.8 — What you can now read in the wild

After Chapter 20 you can:

- Read **`{#snippet name(args)}...{/snippet}`** and **`{@render name(args)}`**.
- Read **`children: Snippet`** and **`children?: Snippet<[T]>`**.
- Recognise that **content between component tags becomes the `children` prop** automatically.
- Spot Svelte 4's **`<slot />`** as a refactor target — replace with snippets.

---

## End-of-chapter checkpoint

- [ ] You wrote `Card.svelte`.
- [ ] The home page uses it for the habit list.
- [ ] You can read `{#snippet}` and `{@render}` aloud.

---

# Chapter 21 — `bind:this` and DOM references

> *Today's job:* when a habit is added, smoothly scroll it into view. Visible win: add a habit way down the list — page snaps to it.

---

## Lesson 21.1 — `bind:this`

`bind:this={el}` binds an element reference to a variable:

```svelte
<script lang="ts">
  let inputEl: HTMLInputElement | undefined = $state();
</script>

<input bind:this={inputEl} />
```

The variable starts `undefined`; once the element is in the DOM, the variable holds the actual `HTMLInputElement`.

> **`bind:this`** — Svelte directive: get a reference to the rendered element.

The type is `HTMLInputElement | undefined` because before the component mounts, the element doesn't exist. Senior habit: write the `| undefined`.

---

## Lesson 21.2 — Using the reference inside `$effect`

```ts
$effect(() => {
  inputEl?.focus();
});
```

The `?.` is important — the first time `$effect` runs, `inputEl` might still be `undefined`. `?.focus()` is safe.

Senior pattern: *all* DOM references are typed `T | undefined`, and accessed with `?.`.

---

## Lesson 21.3 — Scrolling on add

In `+page.svelte`:

```ts
let listEl: HTMLUListElement | undefined = $state();
let lastCount = 0; // track previous length so we can detect adds vs deletes

$effect(() => {
  const count = habits.length;
  // scroll only when the list grew (an add); not on deletes/searches
  if (count > lastCount && listEl !== undefined) {
    const last = listEl.lastElementChild;
    if (last instanceof HTMLElement) {
      last.scrollIntoView({ behavior: 'smooth', block: 'center' });
    }
  }
  lastCount = count;
});
```

In markup:

```svelte
<ul class="habit-list" bind:this={listEl}>
  {#each visibleHabits as habit (habit.id)}
    <HabitRow {habit} onDelete={removeHabit} />
  {/each}
</ul>
```

Save. Add a habit — the page scrolls smoothly to it. Delete one — *no* scroll. Search-filter — also no scroll. (Start with three habits and add a few to see it; with a short list you don't need to scroll at all.)

The `lastCount` tracking is one of the few legitimate `$effect`-with-state-write patterns: we're storing a *previous-value* so we can detect a *transition* (grew vs shrank). It's reading `habits.length` (a dependency) and writing `lastCount` (which is *not* read again inside the effect's same run), so no infinite loop.

> **`scrollIntoView({ behavior, block })`** — DOM method: scroll the element into view with options.
>
> **`lastElementChild`** — the last child element (skipping text nodes).
>
> **`instanceof`** — type check for class membership.

---

## Lesson 21.4 — Read this code

```svelte
<script lang="ts">
  let dialogEl: HTMLDialogElement | undefined = $state();

  function openDialog(): void {
    dialogEl?.showModal();
  }
</script>

<button type="button" onclick={openDialog}>Open</button>
<dialog bind:this={dialogEl}>
  <p>Hello</p>
  <button type="button" onclick={() => dialogEl?.close()}>Close</button>
</dialog>
```

What's `<dialog>` doing?

<details>
<summary>Answer</summary>

`<dialog>` is a built-in HTML element for modal dialogs. `.showModal()` opens it as a modal; `.close()` closes it. We'll use it in Chapter 53.
</details>

---

## Lesson 21.5 — Now you write it

**The English sentence first:**

> *"When the user deletes a habit, the page should not scroll-jump. Instead, focus the *previous* habit's delete button (if any) so keyboard users can keep deleting without losing position."*

<details>
<summary>Worked answer</summary>

```ts
// In +page.svelte
let lastDeletedIndex: number | null = $state(null);

function removeHabit(id: HabitId): void {
  const idx = habits.findIndex((h) => h.id === id);
  if (idx >= 0) lastDeletedIndex = idx;
  habits = habits.filter((h) => h.id !== id);
}

$effect(() => {
  if (lastDeletedIndex === null) return;
  if (listEl === undefined) return;

  // After delete, the *previous* habit now sits at the deleted index − 1
  // (or 0 if we deleted the first one).
  const targetIndex = Math.max(0, lastDeletedIndex - 1);
  const target = listEl.children[targetIndex];
  if (target instanceof HTMLElement) {
    const button = target.querySelector('button[aria-label^="Remove"]');
    if (button instanceof HTMLElement) button.focus();
  }
  lastDeletedIndex = null;
});
```

Two senior touches: (1) we *don't* scroll — focus management is the better tool for keyboard users; the page scrolls only if the focused element is off-screen, and the browser handles that correctly via `focus({ preventScroll: false })` defaults. (2) the `lastDeletedIndex = null` reset prevents the effect from re-firing on unrelated re-renders.

A more robust real-world implementation would keep a `WeakMap<Habit, HTMLElement>` of row-to-button refs (see Ch 21.1's `bind:this`) — what we wrote works because of DOM ordering, but breaks if the list is reordered.
</details>

---

## Lesson 21.6 — Recurring concepts from earlier chapters

- **`$state` for refs** (Ch 17 preview) — DOM references are `T | undefined` and stored in `$state`.
- **Optional chaining `?.`** (Ch 4) — every DOM-ref call goes through `?.`.
- **`$effect` for legitimate side effects** (Ch 17) — DOM imperative APIs are exactly the right use.
- **Type narrowing with `instanceof`** — `if (last instanceof HTMLElement)` narrows from `Element | null` to `HTMLElement`.

---

## Lesson 21.7 — What you can now read in the wild

After Chapter 21 you can:

- Read **`bind:this={el}`** and **`el: HTMLXxxElement | undefined = $state()`**.
- Read **`el?.scrollIntoView(...)`**, **`el?.focus()`**, **`el?.showModal()`**.
- Recognise the **`<dialog>`** element as the modern modal primitive.
- Read **`instanceof`** narrowing as a type-safety pattern.

---

## End-of-chapter checkpoint

- [ ] Adding a habit scrolls smoothly to it.
- [ ] Deleting a habit does *not* scroll.
- [ ] You can read `bind:this={el}` and `el?.focus()` aloud.

---

# Chapter 22 — Loading, empty, error — the trio

> *Today's job:* simulate save latency; while saving, show a tiny spinner; on simulated failure, show an inline error. Visible win: a 1.2-second simulated save shows a spinner; toggling "force fail" shows the error.

We're previewing async work. We'll meet real network calls in Part VI.

---

## Lesson 22.1 — The four-state UI rule

For any async operation, there are *four* states:

1. **idle** — nothing in flight.
2. **loading** — request in flight.
3. **success** — request succeeded; we have data.
4. **error** — request failed; we have an error.

Senior engineers always design all four. The naive "two booleans" version (`loading: boolean, error: string | null`) is a trap — both can be true at once, neither tells the truth on its own. The right shape is a **discriminated union** — Chapter 23.

For now, preview:

```ts
type SaveState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success' }
  | { status: 'error'; message: string };

let saveState: SaveState = $state({ status: 'idle' });
```

---

## Lesson 22.2 — Simulating latency

We *deliberately* don't use `Math.random()` to trigger failures. A flaky save makes debugging painful: the user clicks Add, gets *Saving…* for 1.2s, sees an error, refreshes, can't reproduce. **Random failures are a testing antipattern.** Instead we put a "Force fail next save" toggle on the page; the reader controls when failure happens.

```ts
let forceNextFail = $state(false);

async function fakeSave(name: string): Promise<void> {
  await new Promise((r) => setTimeout(r, 1200));
  if (forceNextFail) {
    forceNextFail = false; // reset so the toggle is one-shot
    throw new Error('Network error (simulated)');
  }
  // success — no return
}

async function addHabit(): Promise<void> {
  const trimmed = newHabit.trim();
  if (trimmed === '') return;

  saveState = { status: 'loading' };
  try {
    await fakeSave(trimmed);
    habits = [...habits, makeHabit(trimmed)];
    newHabit = '';
    saveState = { status: 'success' };
  } catch (err) {
    saveState = { status: 'error', message: err instanceof Error ? err.message : 'Unknown' };
  }
}

// handleSubmit needs to await addHabit now that it's async.
// We make handleSubmit itself async; the form's onsubmit accepts async handlers.
async function handleSubmit(event: SubmitEvent): Promise<void> {
  event.preventDefault();
  await addHabit();
}
```

Several new ideas:

> **`async`** — marks a function as asynchronous. Returns a Promise.
>
> **`await`** — pauses an async function until a Promise resolves.
>
> **`Promise<T>`** — a value that will eventually be `T` (or throw an error).
>
> **`try { ... } catch (err) { ... }`** — handle thrown errors gracefully.

We'll deepen async in Part VI. Today the wiring is the lesson.

---

## Lesson 22.3 — The markup

```svelte
<form onsubmit={handleSubmit}>
  <TextInput bind:value={newHabit} placeholder="Add a new habit..." />
  <button type="submit" disabled={saveState.status === 'loading' || newHabit.trim() === ''}>
    {saveState.status === 'loading' ? 'Saving...' : 'Add'}
  </button>
</form>

<label class="dev-toggle">
  <input type="checkbox" bind:checked={forceNextFail} />
  Force the next save to fail
</label>

{#if saveState.status === 'error'}
  <p class="error">{saveState.message}</p>
{/if}
```

```css
.error { color: #c00; padding: 0.5rem; background: #fee; border-radius: 0.25rem; }
.dev-toggle { display: block; margin: 0.5rem 0; color: #888; font-size: 0.875rem; }
```

Save. Click *Add* — *Saving…* for 1.2 seconds, then success. Tick the toggle; click *Add* again — failure, with the inline error. The toggle resets itself after firing, so the next save succeeds again. *Deterministic*. You're in control.

---

## Lesson 22.4 — The CLS landmine, named early

There's a temptation to add a "skeleton" while loading — a grey placeholder where the new habit will appear. **Don't.** The new habit is added optimistically (we'll do this for-real in Chapter 43); the spinner inside the button is enough feedback. A skeleton at the bottom of the list would cause **layout shift** when the real habit replaces it.

> **CLS** — *Cumulative Layout Shift.* Google's metric for how much your page jumps around as it loads/updates. Senior engineers minimize this aggressively.

The senior rule, foreshadowed: *"don't flip a loading flag during a mutation that already has its own optimistic representation."* We'll see this in full in Chapter 43.

---

## Lesson 22.5 — Read this code

```ts
type RequestState<T> =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success'; data: T }
  | { status: 'error'; message: string };
```

Why is the `data` field only on the `success` branch?

<details>
<summary>Answer</summary>

Because data only *exists* on success. If you put `data: T | null` on every branch, you'd be lying — in the `idle` branch, `data: T | null` says *"data might be there"*, but it can't be; we haven't even started. The discriminated-union shape tells the type system the truth, and the compiler will refuse to let you read `data` until you've confirmed `status === 'success'`. Senior win.
</details>

---

## Lesson 22.6 — Now you write it

**The English sentence first:**

> *"Add a 'Retry' button to the error state. When clicked, it re-attempts the save with the most recent name. Disable Retry while a save is in flight."*

<details>
<summary>Worked answer</summary>

Extend the discriminator so `error` carries the name we tried:

```ts
type SaveState =
  | { status: 'idle' }
  | { status: 'loading' }
  | { status: 'success' }
  | { status: 'error'; message: string; name: string };

let saveState: SaveState = $state({ status: 'idle' });

async function trySave(name: string): Promise<void> {
  saveState = { status: 'loading' };
  try {
    await fakeSave(name);
    habits = [...habits, makeHabit(name)];
    saveState = { status: 'success' };
  } catch (err) {
    saveState = {
      status: 'error',
      message: err instanceof Error ? err.message : 'Unknown',
      name, // remember what we tried
    };
  }
}

async function addHabit(): Promise<void> {
  const trimmed = newHabit.trim();
  if (trimmed === '') return;
  await trySave(trimmed);
  if (saveState.status === 'success') newHabit = '';
}
```

In markup, the error block gains a Retry button:

```svelte
{#if saveState.status === 'error'}
  <p class="error">{saveState.message}</p>
  <button type="button" onclick={() => trySave(saveState.name)}>
    Retry
  </button>
{/if}
```

Senior touches: (1) `trySave` is the *single* save path, called from both the form and the retry button — no duplication. (2) The discriminator carries `name` only on the error variant; if we tried to read `saveState.name` while idle, the compiler refuses. That's the discriminated union doing its job.
</details>

---

## Lesson 22.7 — Recurring concepts from earlier chapters

Part III's spine, in one place:

- **Components, `$props`, callback props** (Ch 13–15) — `HabitRow`, `TextInput`, `Card`.
- **`$derived`, `$effect`** (Ch 16, 17) — used for computing visible habits and scroll-on-add.
- **`$bindable`** (Ch 18) — reusable `TextInput`.
- **`$inspect`** (Ch 19) — debugging.
- **Snippets `{#snippet}` / `{@render}`** (Ch 20) — `<Card>` chrome.
- **`bind:this`** (Ch 21) — DOM refs typed `T | undefined`.

All of Part III is now working together inside one app.

---

## Lesson 22.8 — What you can now read in the wild

After Part III you can:

- Read any modern Svelte 5 component tree with components, runes, snippets, callback props, and bindable values.
- Write a four-state UI (idle/loading/success/error) using a discriminated union (formal coverage Ch 23).
- Read `async` / `await` / `try`/`catch` and explain happy-path vs error-path flow.
- Spot the **CLS landmine** in optimistic UI code.

---

## End-of-chapter checkpoint

- [ ] *Add* shows a spinner. Untoggled, success; toggle "Force the next save to fail", click *Add*, see the error.
- [ ] You read the four-state pattern aloud.
- [ ] You feel why "two booleans" is wrong.

End of Part III. Next: TypeScript-strict deepening.
