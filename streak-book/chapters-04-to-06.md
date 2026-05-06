# Part I (continued) — Chapters 4, 5, 6

---

# Chapter 4 — Strings, template literals, and the nullish coalescer

> *Today's job:* greet the user by name in the page title. If we don't know their name yet, say *"friend"*. By the end you will have written your first piece of input, your first interpolated greeting, and your first bug-resistant default.

Streak is going to have users. Not yet — Part VII does that work — but the *page* needs to *believe* in users before the database does. Today we plant the placeholder.

---

## Lesson 4.1 — Three quotes, three uses

JavaScript and TypeScript give you three ways to write a piece of text — three quote characters that all produce a `string`:

```ts
const a: string = 'single quotes';
const b: string = "double quotes";
const c: string = `backticks`;
```

> **`string`** *(noun)* — a piece of text. The type. Always lowercase in TypeScript: `string`, not `String`. (Capital-`S` `String` is something different and you'll basically never use it.)

For our book the rule is simple:

- **Default to single quotes (`'...'`).** Cleanest visually, matches our Prettier config.
- **Use backticks (`` `...` ``) when you need to interpolate a value or write a multi-line string.** That's the new one — see the next lesson.
- **Use double quotes (`"..."`) inside JSX-like attributes or inside a string that already contains a single quote.** *"It's nice"* needs `"It's nice"` or `'It\'s nice'`. Pick the one that needs no escaping.

You'll meet senior code that mixes them; that's fine. The codebase's Prettier config will pick one default and force it. Ours picks single.

---

## Lesson 4.2 — Template literals: putting values inside text

The most useful of the three is the backtick. Inside backticks, you can drop in a value with `${...}`:

```ts
const name: string = 'Billy';
const greeting: string = `Welcome back, ${name}.`;
// greeting is 'Welcome back, Billy.'
```

Read aloud:

| Line | Read aloud as |
|---|---|
| `const name: string = 'Billy';` | *"Let `name` be the string 'Billy', and don't reassign it."* |
| `const greeting: string = \`Welcome back, ${name}.\`;` | *"Let `greeting` be the text 'Welcome back, ' followed by the value of `name`, followed by a period."* |

> **template literal** *(noun)* — a string written with backticks (`` ` ``) that supports `${expression}` interpolation and multi-line text. The most flexible string syntax. Engineers use them constantly.
>
> **interpolation** *(noun)* — putting a value into the middle of a string. The `${...}` part of a template literal is the interpolation site.

You can also span lines:

```ts
const message: string = `
  Welcome back, ${name}.
  You've got habits to log.
`;
```

The whitespace and line breaks are preserved exactly. We use this rarely, but it's good to know.

---

## Lesson 4.3 — `const` vs `let`, briefly

You've now seen both `let` (Chapter 1) and `const` (Chapter 4):

- **`const`** — *"this name will not be reassigned to point at a different value."* Use it by default.
- **`let`** — *"I will reassign this name later."* Use it when you have to.

Senior rule: **`const` until the compiler complains.** If it never complains, it stays `const`. Most variables in real code never get reassigned.

What `const` does *not* mean: *"the value is immutable."* You can still mutate the *contents* of an object stored in a `const`. We'll meet that nuance properly in Chapter 12. For now: `const` for things that don't get reassigned (like `name`, `greeting`); `let` for things that do (like `habitsLoggedToday`, which we keep changing).

---

## Lesson 4.4 — `null` and `undefined` (and the difference real engineers care about)

Two values represent *"there is no value here"* in JavaScript: `null` and `undefined`. They're not the same, and the difference matters.

> **`undefined`** — *"a value was never assigned."* This is the default. An uninitialised variable is `undefined`. A function with no `return` returns `undefined`. A property that doesn't exist on an object reads as `undefined`.
>
> **`null`** — *"there is intentionally no value here."* You assign it on purpose to mean *"empty"*.

The rule of thumb most senior engineers in this stack follow:

- **For optional fields you control** — use `undefined`. The TypeScript way is `field?: string`, which makes the field's type `string | undefined`.
- **For "we know it's missing"** — use `null` deliberately. Think of database `NULL`s, or *"the user hasn't picked a name yet."*

For the rest of this book we'll usually use `null` for *"absent on purpose"* and `undefined` for *"not yet"*. You'll see both in the wild.

---

## Lesson 4.5 — `??` — the nullish coalescer

Now the punchline Chapter 3 set up.

```ts
const name: string | null = $state(null);
const greeting: string = `Welcome back, ${name ?? 'friend'}.`;
```

Read the second line aloud: *"the greeting is 'Welcome back, ' followed by name — or 'friend' if name is missing — and a period."*

> **`??`** — the **nullish coalescing operator**. `a ?? b` returns `a` unless `a` is `null` or `undefined`, in which case it returns `b`. Read aloud as *"or, if missing"*.

Compare to `||` (Chapter 3's foot-gun):

```ts
const score = 0;
const display1 = score || 'No score'; // 'No score' — wrong! score is 0, a real value
const display2 = score ?? 'No score'; // 0 — correct
```

`||` falls back on **any falsy value**. `??` falls back **only on `null` or `undefined`**. For *"default the missing"*, `??` is right. We use it from now on for that purpose.

There's also `??=`, the *"if missing, assign"* shorthand:

```ts
let displayName: string | null = null;
displayName ??= 'friend';
// displayName is now 'friend'

let realName: string | null = 'Billy';
realName ??= 'friend';
// realName is still 'Billy' — the assignment was skipped
```

Read aloud: *"if displayName is missing, default it to 'friend'."*

---

## Lesson 4.6 — Optional chaining (`?.`) — preview

While we're on the topic of *missing*, let's name the symbol you'll see all the time:

```ts
const length = name?.length;
```

If `name` is `null` or `undefined`, `name?.length` evaluates to `undefined` instead of crashing. If `name` is a real string, it returns the length.

> **`?.`** — the **optional chaining operator**. `a?.b` is *"if `a` exists, return `a.b`; otherwise return undefined and don't crash."*

We'll use `?.` reflexively from now on. Without it, you'd be writing `name === null ? null : name.length` everywhere — annoying, error-prone, and ugly.

---

## Lesson 4.7 — Wiring it into Streak

Today's job in code. We'll add a name field, an input bound to it, and a greeting.

A small TypeScript subtlety first. We *want* to model "the user's name might be missing" as `string | null`. But `<input type="text">` always has a `string` value (an empty input is `''`, never `null`). Binding `string | null` to a text input would fight TypeScript. So we bind a plain `string` and treat the empty string as "not set" *only at the display boundary* (the `{userName.trim() === '' ? 'friend' : userName}` line below).

This is a real-world pattern: **the data model and the input model are sometimes different shapes, and you convert at the edge.** The lessons on `null`, `??`, and `?.` you just learned still apply everywhere else in the book — Chapter 9 onward you'll see them with real `string | null` shapes from the database.

Update `src/routes/+page.svelte`:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);
  let userName = $state('');

  function logHabit(): void {
    habitsLoggedToday += 1;
  }

  function unlogHabit(): void {
    if (habitsLoggedToday <= 0) {
      return;
    }

    habitsLoggedToday -= 1;
  }

  function resetHabits(): void {
    if (habitsLoggedToday <= 0) {
      return;
    }

    habitsLoggedToday = 0;
  }
</script>

<h1>Welcome back, {userName.trim() === '' ? 'friend' : userName}.</h1>

<label>
  Your name
  <input type="text" bind:value={userName} placeholder="What should we call you?" />
</label>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>Log a habit</button>

<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday <= 0}>
  Undo
</button>

<button type="button" onclick={resetHabits} disabled={habitsLoggedToday <= 0}>
  Reset
</button>
```

Read aloud the new lines:

| Line | Read aloud as |
|---|---|
| `let userName = $state('');` | *"Let userName be reactive state, starting as an empty string."* |
| `<h1>Welcome back, {userName.trim() === '' ? 'friend' : userName}.</h1>` | *"If the trimmed name is empty, show 'friend'; otherwise show the name."* |
| `<input type="text" bind:value={userName} ...>` | *"A text input — its value is bound two-way to userName."* |
| `placeholder="What should we call you?"` | *"Show this hint inside the input when it's empty."* |

Save (`Cmd+S` / `Ctrl+S`). Open the browser. Type your name — watch the heading update letter by letter. Clear the input — heading reverts to *"Welcome back, friend."*.

That input wiring is `bind:value` — a Svelte feature for two-way input binding. You typed; the variable updated; the heading re-rendered. We'll cover `bind:value` formally in Chapter 8.

> **`bind:value`** *(preview)* — a Svelte directive that binds an input element's value to a variable. When the user types, the variable updates. When code reassigns the variable, the input updates. We'll meet it formally Chapter 8.

---

## Lesson 4.8 — Why we wrote it this way

The deliberate choices in today's code:

1. **`let userName = $state('')`, with `string` not `string | null`.** Here, the *display* concept of "missing" is "the trimmed input is empty" — and HTML inputs already use `''` for that. We respect the input model. Where the data genuinely *can be null* (database fields, server responses), we'll model with `string | null` and reach for `??`. The two patterns coexist.

2. **`{userName.trim() === '' ? 'friend' : userName}`, not `{userName || 'friend'}`.** Even though both `''` and falsy give the same result here, the explicit `.trim() === ''` says exactly what we mean — *"the user has not entered a non-blank name"* — and survives the day a typo elsewhere makes `userName` something other than a plain string.

3. **`.trim()` before checking.** A user who types just spaces hasn't given us a name; they get *"friend"*, not the literal three-space string. Senior habit: trim before you decide.

4. **`<label>` wrapping the input.** Accessibility 101: every input needs a label. The screen-reader habit. We'll deepen this in Chapter 53.

5. **`placeholder` for the hint, not the value.** A common beginner mistake is putting the *default* into `placeholder`. Placeholder text *vanishes* when the user types — you can't read it back. It's a hint, not a value.

---

## Lesson 4.9 — Read this code

### Snippet A

```ts
const a: string | null = null;
const b: string = a ?? '';
const c: string = a || '';
```

What are `b` and `c`?

<details>
<summary>Answer</summary>

`b` is `''` (the empty string). `a` is `null`, so `??` falls back to the right side.

`c` is also `''`. Here `||` happens to give the right answer because `null` is falsy. But — and this is the subtle bit — if `a` were the empty string `''` instead of `null`, then `b` would still be `''` (because `??` only falls back on `null`/`undefined`), while `c` would *also* be `''` (because `''` is falsy and `||` falls back).

The difference shows up when the "real" value is something like `0`:

```ts
const score: number | null = 0;
const display1 = score ?? -1; // 0 — keeps the real zero
const display2 = score || -1; // -1 — wrong, throws away a real zero
```

This is why `??` is the correct tool *most of the time*.
</details>

### Snippet B

```ts
type User = { name: string | null };
const u: User = { name: null };
const len = u.name?.length ?? 0;
```

What is `len`?

<details>
<summary>Answer</summary>

`0`. `u.name` is `null`. `u.name?.length` short-circuits to `undefined`. `undefined ?? 0` falls back to `0`.

This pattern — `value?.field ?? fallback` — is one of the three or four most common idioms in modern TypeScript. Memorise it.
</details>

---

## Lesson 4.10 — Now you write it

**The English sentence first:**

> *"When the user has a name, also include it in the paragraph that says how many habits they've logged. Like 'You've logged 3 habits today, Billy.' When they don't have a name, the sentence should still read naturally — no awkward 'You've logged 3 habits today, friend.' just drop the name part."*

You'll need:
- a template literal,
- a check (in markup, with `{#if}` — preview from Chapter 6, but you can use the ternary you already know),
- and `userName ?? '...'` somewhere — *or* a different shape that omits the name entirely when missing.

Try before peeking.

<details>
<summary>Worked answer (one version)</summary>

```svelte
<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far{userName.trim() !== '' ? `, ${userName}` : ''}.
</p>
```

Read aloud: *"so far, comma, name — if there is one — and then a period."*

The `{userName.trim() !== '' ? \`, ${userName}\` : ''}` part: when there's a non-blank name, it inserts `, Billy` (with the leading comma and space). When there isn't, it inserts an empty string. The full sentence reads naturally either way.

A second valid answer uses Chapter 6's `{#if}` block (which you haven't met formally yet):

```svelte
<p>
  You've logged <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far
  {#if userName.trim() !== ''}, {userName}{/if}.
</p>
```

Both are fine. The second is more idiomatic for longer conditional sections.
</details>

---

## Lesson 4.11 — Recurring concepts from earlier chapters

Chapter 4 used these:

- **`$state(...)`** (Ch 1) — `userName` is reactive state too; the heading updates live as you type.
- **`===`** (Ch 1) — strict equality on the trim check.
- **Ternary `cond ? a : b`** (Ch 1) — picks `'friend'` vs the typed name.
- **Falsy values** (Ch 3) — we *don't* lean on `''` being falsy here; we check `.trim() === ''` explicitly, so the intent is on the page.

---

## Lesson 4.12 — What you can now read in the wild

After Chapter 4 you can:

- Read **`'single'`**, **`"double"`**, and **`` `template ${value}` ``** strings and pick the right one.
- Read **`null`** and **`undefined`** and explain the difference.
- Read **`a ?? b`** (nullish-coalesce) and **`a?.b`** (optional-chain) confidently.
- Spot **the `||` foot-gun** and rewrite with `??`.
- Read **union types** like `string | null` — and know that input controls bind to one specific shape, not both.
- Read **`bind:value={x}`** as two-way input binding, even though we'll formalise it Chapter 8.

---

## Glossary added in Chapter 4

| Term | Definition |
|---|---|
| `string` | The text type. |
| template literal | A string in backticks supporting `${...}` interpolation. |
| interpolation | Inserting a value into a string. |
| `const` | Variable declaration that won't be reassigned. |
| `null` | "There is intentionally no value." |
| `undefined` | "A value was never assigned." |
| `??` | Nullish coalescer — fallback when left is `null`/`undefined`. |
| `??=` | Assign-if-missing shorthand. |
| `?.` | Optional chaining — safe property access. |
| union type (`A \| B`) | A type that can be either `A` or `B`. |
| `<label>` | The HTML element pairing a label with an input. |
| `placeholder` | Hint text inside an empty input. |
| `.trim()` | String method removing surrounding whitespace. |

---

## End-of-chapter checkpoint

- [ ] You typed your name in the input and watched the heading update.
- [ ] You cleared the input and saw it revert to "friend".
- [ ] You can read the difference between `||` and `??` aloud.
- [ ] You can name two falsy values that `??` would *not* fall back on (it only falls back on `null` and `undefined`).
- [ ] You can explain why `string | null` is better than just `string` for an "optional name" field.

---

# Chapter 5 — `for...of` and the loop you'll actually write

> *Today's job:* a strip of seven dots across the home screen — one per day of the week — with today's dot highlighted. *Visible win:* a tiny weekly indicator at the top of the page.

You're going to meet your first **array** today. We'll cover arrays formally in Chapter 7; today we just need enough to render seven things in a row. The rune of the chapter is `for...of`, the loop you'll actually use, and `{#each}`, its Svelte-template cousin.

---

## Lesson 5.1 — Arrays at a glance

An **array** is an ordered list of values. You write one with square brackets:

```ts
const days: string[] = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];
```

Read aloud: *"let days be a list of strings: Mon, Tue, Wed..."*

> **array** *(noun)* — an ordered, indexed list. Elements are accessed by position, starting at `0`.
>
> **`string[]`** — the type "array of strings". Equivalent to `Array<string>`. We use the `[]` shorthand.

Two things you can do with an array right now:

1. **Get the length** — `days.length` is `7`.
2. **Read by index** — `days[0]` is `'Mon'`, `days[6]` is `'Sun'`.

(There's a wrinkle in TypeScript-strict mode that `days[0]` is `string | undefined` — because the compiler doesn't trust that the array isn't empty. We'll meet that properly in Chapter 7. For now, no indexing; we'll loop instead.)

---

## Lesson 5.2 — `for...of` — the loop you'll write 95% of the time

There are three loops in JavaScript: `for`, `while`, and `for...of`. **In modern code, the one you reach for is `for...of`.** Reasons in a moment.

```ts
for (const day of days) {
  console.log(day);
}
```

Read aloud: *"for each day in the days list, log the day."*

> **`for...of`** — a loop that iterates over the elements of an array (or any iterable). Reads cleanly, no index management.
>
> **`console.log(...)`** — print a value to the browser's developer-console. The simplest debug tool you'll use.

Compare to the older C-style:

```ts
for (let i = 0; i < days.length; i++) {
  const day = days[i];
  console.log(day);
}
```

This works, but it's three things at once: index management (`let i = 0`), bounds (`i < days.length`), step (`i++`). You almost never need any of those. `for...of` is the senior default.

When *do* you need an index? When you genuinely care about position. For that, you use `.entries()`:

```ts
for (const [i, day] of days.entries()) {
  console.log(`${i}: ${day}`);
}
```

This logs `0: Mon`, `1: Tue`, etc. We'll meet this exactly when we need it.

The other two loops:
- **`for (...)` C-style** — you'll see it in old code; you almost never need to write it. The exception: when you must `break` mid-loop based on the index. Even then, `for...of` + `if` + `break` reads better.
- **`while (...)`** — useful when you don't know how many iterations you'll need (waiting for a condition). Rare in component code.

---

## Lesson 5.3 — `{#each}` in markup

`for...of` is for *script* code. In *markup*, you use `{#each}`:

```svelte
<ul>
  {#each days as day}
    <li>{day}</li>
  {/each}
</ul>
```

Read aloud: *"for each day in the days list, render a list-item containing the day."*

The variants you'll meet:

```svelte
{#each items as item}             <!-- just the value -->
{#each items as item, i}          <!-- value and index -->
{#each items as item (item.id)}   <!-- keyed each — we'll meet this Chapter 7 -->
{#each items as item}
  <p>...</p>
{:else}
  <p>No items.</p>                <!-- the empty-state slot -->
{/each}
```

The `{:else}` slot is the empty-state pattern Chapter 6 will formally name.

---

## Lesson 5.4 — Today's date, briefly

We need to know which day is today. JavaScript has a built-in `Date` object:

```ts
const today: Date = new Date();
const dayIndex: number = today.getDay();
```

`getDay()` returns:
- `0` for Sunday
- `1` for Monday
- ...
- `6` for Saturday

That order is annoying — most calendars start on Monday, not Sunday. We'll convert: a small helper function that returns *"day-of-week, Monday=0, Sunday=6"*.

> **`Date`** *(class)* — JavaScript's built-in time type. `new Date()` creates one for *now*. We'll meet it again in Chapter 9 when habits get timestamps.

Add this to your script:

```ts
function getMondayBasedDayIndex(): number {
  const sundayBased: number = new Date().getDay();

  if (sundayBased === 0) {
    return 6; // Sunday becomes 6
  }

  return sundayBased - 1; // Mon=1 becomes 0, Tue=2 becomes 1, ...
}
```

Read aloud: *"get today's day-index in JavaScript's Sunday-based numbering. If it's Sunday, return 6. Otherwise, subtract 1."*

This is a tiny example of what real code looks like — a small helper that translates one representation to another with a clear name. It's not flashy. It's just correct.

---

## Lesson 5.5 — Wiring it into Streak

Update `+page.svelte`:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);
  let userName = $state('');

  const days: string[] = ['Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun'];

  function getMondayBasedDayIndex(): number {
    const sundayBased: number = new Date().getDay();

    if (sundayBased === 0) {
      return 6;
    }

    return sundayBased - 1;
  }

  const todayIndex: number = getMondayBasedDayIndex();

  function logHabit(): void {
    habitsLoggedToday += 1;
  }

  function unlogHabit(): void {
    if (habitsLoggedToday <= 0) {
      return;
    }

    habitsLoggedToday -= 1;
  }

  function resetHabits(): void {
    if (habitsLoggedToday <= 0) {
      return;
    }

    habitsLoggedToday = 0;
  }
</script>

<h1>Welcome back, {userName.trim() === '' ? 'friend' : userName}.</h1>

<label>
  Your name
  <input type="text" bind:value={userName} placeholder="What should we call you?" />
</label>

<div class="day-strip">
  {#each days as day, i}
    <span class="day" class:today={i === todayIndex}>{day}</span>
  {/each}
</div>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>Log a habit</button>

<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday <= 0}>
  Undo
</button>

<button type="button" onclick={resetHabits} disabled={habitsLoggedToday <= 0}>
  Reset
</button>

<style>
  .day-strip {
    display: flex;
    gap: 0.5rem;
    margin: 1rem 0;
  }

  .day {
    padding: 0.25rem 0.5rem;
    border-radius: 999px;
    background: #eee;
    font-size: 0.875rem;
  }

  .day.today {
    background: #2563eb;
    color: white;
    font-weight: 600;
  }
</style>
```

A few new things appeared. Read each new line aloud:

| Line | Read aloud as |
|---|---|
| `const days: string[] = ['Mon', 'Tue', ...];` | *"Let days be a list of strings: Mon through Sun."* |
| `const todayIndex: number = getMondayBasedDayIndex();` | *"Let todayIndex be the result of getMondayBasedDayIndex, computed once at module load."* |
| `<div class="day-strip">` | *"A container with the class day-strip."* |
| `{#each days as day, i}` | *"For each day in days, with i as its index."* |
| `<span class="day" class:today={i === todayIndex}>` | *"A span with class day, plus class today when i equals today's index."* |
| `{day}` | *"Insert the day name."* |
| `{/each}` | *"End of the each block."* |

> **`<style>`** — a `<style>` block inside a `.svelte` file scopes its CSS to *only this component*. The `.day-strip` selector here doesn't leak to other components. This is one of Svelte's killer features.
>
> **`class:today={...}`** — Svelte directive: *"add the class `today` to this element when the expression is true."* You'll also see Svelte 5's array-class form: `class={[base, condition && 'today']}`. Both are valid; pick one and stay consistent. We use `class:` for boolean toggles because it reads cleaner.
>
> **`{#each days as day, i}`** — the index variant.

Save (`Cmd+S` / `Ctrl+S`). Look at the page. You should see a row of seven pill-shaped tags, with today's pill highlighted in blue.

> **A small caveat:** `todayIndex` is computed once when the page loads. If you leave Streak open across midnight, the strip won't refresh until you reload. Real apps fix this with a periodic check or a server `load` (Chapter 40). For now, refresh works.

---

## Lesson 5.6 — Why we wrote it this way

1. **`const` for `days` and `todayIndex`.** Both are computed once and never reassigned. `let` would have lied.

2. **The helper function `getMondayBasedDayIndex`.** Pulled out instead of inlined. The name *tells the reader what it does*. If it were `const x = new Date().getDay() === 0 ? 6 : new Date().getDay() - 1;` inline, every future reader would have to re-derive *why* the math. Names earn their keep.

3. **`class:today` instead of inline ternary in `class={...}`.** Both work. `class:today={...}` reads better, especially when you have multiple conditional classes.

4. **CSS in component scope.** No global stylesheet pollution. We'll grow this naturally.

5. **`for...of` was *not* used here — we used `{#each}` instead.** Why? Because we're rendering, not computing. `for...of` is for script logic; `{#each}` is for markup. Right tool for the job.

---

## Lesson 5.7 — Read this code

### Snippet A

```ts
const fruits: string[] = ['apple', 'banana', 'cherry'];

for (const fruit of fruits) {
  console.log(fruit.toUpperCase());
}
```

What does the console show?

<details>
<summary>Answer</summary>

```
APPLE
BANANA
CHERRY
```

`for...of` iterates over each fruit; `.toUpperCase()` is a string method that returns the uppercase version. We'll meet string methods properly in Chapter 11.
</details>

### Snippet B

```svelte
<script lang="ts">
  const scores: number[] = [88, 92, 75, 100];
  const passingThreshold = 80;
</script>

{#each scores as score, i}
  <p>Question {i + 1}: {score} {score >= passingThreshold ? '✓' : '✗'}</p>
{/each}
```

What appears on the page?

<details>
<summary>Answer</summary>

Four paragraphs:
- *Question 1: 88 ✓*
- *Question 2: 92 ✓*
- *Question 3: 75 ✗*
- *Question 4: 100 ✓*

The `i + 1` makes the display 1-based even though `i` starts at 0. Senior habit: when displaying counts to humans, start at 1. When indexing for code, use 0.
</details>

---

## Lesson 5.8 — Now you write it

**The English sentence first:**

> *"Make the day strip distinguish past, today, and future. Past days (Mon–Wed if today is Thursday) should be slightly faded; today should be the bright pill we already have; future days should be the default grey."*

You'll need:
- a second `class:` directive — `class:past={...}` — based on `i < todayIndex`,
- a CSS rule for `.day.past` that sets a lower opacity (or a lighter background),
- nothing else.

Try before peeking.

<details>
<summary>Worked answer</summary>

In the markup:

```svelte
<span class="day"
      class:today={i === todayIndex}
      class:past={i < todayIndex}>
  {day}
</span>
```

In the style block:

```css
.day.past {
  opacity: 0.5;
}
```

The result: past days are faded; today is the bright pill; future days stay default. You've shipped your first piece of *temporal* UI — a small visual that tells the user *where they are in the week*. Senior engineers love these. Users feel them subliminally.
</details>

---

## Lesson 5.9 — Recurring concepts from earlier chapters

- **Early-return guard** (Ch 2) — `getMondayBasedDayIndex` uses one for the Sunday case.
- **`===`** (Ch 1) — `i === todayIndex` for the highlight.
- **`function … (): number`** (Ch 1) — explicit return type.
- **`const`** (Ch 4) — `days` and `todayIndex` never get reassigned.

---

## Lesson 5.10 — What you can now read in the wild

After Chapter 5 you can:

- Read **`const xs: string[] = [...]`** — array literal with a type annotation.
- Read **`for (const x of xs) { ... }`** — the modern loop.
- Read **`for (const [i, x] of xs.entries())`** — when you need the index.
- Read **`{#each xs as x, i (x.id)}`** — the keyed `each` (Chapter 7 next).
- Read **`<style>`** as scoped CSS.
- Read **`class:foo={cond}`** and the array-form `class={[...]}` and explain both.

---

## Glossary added in Chapter 5

| Term | Definition |
|---|---|
| array | An ordered, indexed list. |
| `string[]` | The type "array of strings". |
| `for...of` | A loop iterating over an iterable's elements. |
| `for (...)` C-style | The classic three-part loop; rarely needed. |
| `while` | A loop that runs while a condition is true. |
| `console.log(...)` | Print a value to the dev console. |
| `{#each}` | Svelte's loop in markup. |
| `Date` | JavaScript's built-in time type. |
| `getDay()` | Returns 0–6 (Sun–Sat). |
| `<style>` | A scoped CSS block in a `.svelte` file. |
| `class:foo={...}` | Svelte directive for conditional class. |
| `.toUpperCase()` | String method returning the uppercase version. |

---

## End-of-chapter checkpoint

- [ ] You see seven pills across your screen, one per day of the week.
- [ ] Today's pill is highlighted.
- [ ] (After the exercise) past days are faded, future days are not.
- [ ] You can explain why we wrote `for...of` instead of the C-style `for`.
- [ ] You can explain why we used `{#each}` in markup instead of `for...of` in script.

---

# Chapter 6 — The empty state, and the Part I checkpoint

> *Today's job:* when `habitsLoggedToday` is `0`, the page shows a friendly empty-state card instead of a row of three buttons sitting next to *"You've logged 0 habits so far."*. *Visible win:* a clean *"No habits yet — log your first one!"* card.

This is a recap chapter. Almost every concept from Chapters 1–5 reappears in one componentised piece of UI, and we add the formal `{#if}` / `{:else}` blocks that finish off Part I.

---

## Lesson 6.1 — `{#if}`, `{:else if}`, `{:else}` formally

You've seen `{#if ... }{/if}` glance through Chapter 4's exercise. Here it is officially:

```svelte
{#if condition}
  <!-- shown when condition is true -->
{:else if otherCondition}
  <!-- shown when otherCondition is true -->
{:else}
  <!-- shown when none of the above is true -->
{/if}
```

Read aloud: *"if the condition holds, show this. Otherwise if the other condition holds, show that. Otherwise, show the third thing."*

Three observations:
- The block opens with `{#if}` and closes with `{/if}`. The `#` marks the *opener*, the `/` marks the *closer*.
- Branches use `{:else if}` and `{:else}` — note the colon `:`.
- `{:else if}` chains can be as long as you need; we usually keep them short and reach for a discriminated union (Chapter 23) when they grow.

---

## Lesson 6.2 — Wiring it into Streak

We want a different page when `habitsLoggedToday === 0`. The simplest version:

```svelte
{#if habitsLoggedToday === 0}
  <div class="empty-state">
    <h2>No habits yet</h2>
    <p>Log your first one!</p>
    <button type="button" onclick={logHabit}>Log a habit</button>
  </div>
{:else}
  <p>
    You've logged
    <strong>{habitsLoggedToday}</strong>
    {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
  </p>

  <button type="button" onclick={logHabit}>Log a habit</button>

  <button type="button" onclick={unlogHabit} disabled={habitsLoggedToday <= 0}>
    Undo
  </button>

  <button type="button" onclick={resetHabits} disabled={habitsLoggedToday <= 0}>
    Reset
  </button>
{/if}
```

Read aloud the new wiring:

| Line | Read aloud as |
|---|---|
| `{#if habitsLoggedToday === 0}` | *"If today's count is exactly zero..."* |
| `<div class="empty-state">` | *"...show the empty-state card..."* |
| `<h2>No habits yet</h2>` | *"...with this heading..."* |
| `<p>Log your first one!</p>` | *"...this prompt..."* |
| `<button onclick={logHabit}>Log a habit</button>` | *"...and a single button to start."* |
| `{:else}` | *"Otherwise..."* |
| *populated layout* | *"...show the count, the buttons, the controls."* |
| `{/if}` | *"End of the conditional."* |

Save (`Cmd+S` / `Ctrl+S`). At zero, you see the empty-state card. Click *Log a habit* — the page snaps to the populated layout. Click *Reset* — back to the empty state.

> **empty state** *(noun)* — the UI shown when a list, dashboard, or page has no data yet. Senior engineers always design one. Without it, a brand-new user sees something broken-looking and bounces.

Three states matter for almost every list-shaped UI: *empty*, *populated*, and *error*. We've handled empty and populated; error lands in Part III. Make a mental note: the *triple* (empty / populated / error) is a senior pattern you'll see in 90% of well-built UIs.

---

## Lesson 6.3 — A small CSS pass for the empty state

Add to the `<style>` block:

```css
.empty-state {
  text-align: center;
  padding: 2rem;
  margin: 2rem 0;
  border: 2px dashed #ccc;
  border-radius: 0.5rem;
}

.empty-state h2 {
  margin-top: 0;
  color: #666;
}

.empty-state p {
  color: #888;
  margin-bottom: 1rem;
}
```

Save. The empty state now reads as a *deliberate* card, not an accident.

That CSS is mediocre. We'll polish in Part VIII. The lesson here is *that we wrote CSS at all* — the empty state should look *intentional* even when it's the first thing a user sees.

---

## Lesson 6.4 — Build, break, fix

Time for the chapter's deliberate bug. We'll do it twice — once to *see* the bug bypass the UI guard, once to confirm the fix.

**Step 1 — open the dev console.** Press `F12` (or `Cmd+Option+I` on macOS) in your browser. Click the **Console** tab.

**Step 2 — break the function.** Open `unlogHabit` and remove the guard:

```ts
function unlogHabit(): void {
  habitsLoggedToday -= 1;
}
```

Save. The button is still `disabled` at zero, so clicking does nothing visible. The bug is *masked* by the UI.

**Step 3 — bypass the UI.** In dev tools, go to the **Elements** (or **Inspector**) tab. Find the *Undo* button. Find the `disabled` attribute on it. Right-click → *Edit attribute* → delete `disabled`. Press Enter. The button is now clickable even at count zero.

Click it. The count goes to `-1`. The UI shows `"You've logged -1 habits so far."`. Click again — `-2`. The bug shipped because the UI guard *alone* isn't enough.

**Step 4 — restore the function guard:**

```ts
function unlogHabit(): void {
  if (habitsLoggedToday <= 0) {
    return;
  }

  habitsLoggedToday -= 1;
}
```

Save. Reload the page so the `disabled` attribute is back. Repeat the dev-tools-edit trick. Click the unbound button — *nothing happens*. The function guard caught it.

This is the lesson the whole chapter rests on: **the UI guard is one layer; the function guard is another.** When one fails, the other catches. *That* is defence in depth.

> **defence in depth** *(noun)* — a senior pattern: don't rely on one layer to prevent a bug; have two or three. UI guard + function guard + (later) database constraint + audit log = belt + braces + suspenders. When one layer fails, the others catch the bug.

---

## Lesson 6.5 — The Part I out-loud recap

You've made it through Part I. Time to prove fluency.

Sit with your `+page.svelte` and read it out loud, top to bottom, no notes, no peeking at this book. If you stumble on a line, pause, look it up in the running glossary at the end of this document, and continue.

Specifically be able to say:
- What `<script lang="ts">` does.
- What `let foo = $state(...)` does.
- What `function logHabit(): void { ... }` declares.
- What `+=`, `-=`, `===`, `!==` mean.
- Why we use `??` over `||` for the name fallback.
- Why we use early-return guards.
- Why the buttons are `disabled` instead of just doing nothing on click.
- What `{#each days as day, i}` does.
- What `{#if ... }{:else}{/if}` does.
- What scoped CSS in `<style>` does.

If any of those is hesitant, reread the relevant chapter. Don't move on yet. Fluency now is the point — speed later.

---

## Lesson 6.6 — Now you write it

**The English sentence first:**

> *"Add a 'Clear all' button that returns the user to the empty state. It should only be available when the count is greater than zero (the same precondition as Reset). Use the existing pattern."*

This isn't *new*; this is the same `resetHabits` you wrote in Chapter 2 with a different button label. The exercise is in *recognising* that you already have the function, and just adding a second button bound to the same handler.

<details>
<summary>Worked answer</summary>

In the markup, where the buttons are:

```svelte
<button type="button" onclick={resetHabits} disabled={habitsLoggedToday <= 0}>
  Clear all
</button>
```

You can name it differently, or even rename `resetHabits` to `clearAllHabits` if you prefer. Senior habit: when two buttons would do *exactly* the same thing, you don't need two buttons. Pick the wording the user expects ("Clear all" is more natural for *"empty the list"*; "Reset" is more natural for *"set back to default"*) and ship one.

If your version replaced the *Reset* button with *Clear all*, that's fine. If you kept both, that's fine too — though a code reviewer might ask why you have two buttons doing the same thing.
</details>

---

## Lesson 6.7 — End of Part I — what you can now read in the wild

After Part I, you can:

- Read any single-state Svelte 5 component (one `$state`, basic event handlers, conditional rendering, lists) and explain it.
- Identify when an early-return guard is missing and add one.
- Identify the `||` foot-gun on sight.
- Spot off-by-one errors in boundary comparisons.
- Recognise the empty-state pattern as deliberate.
- Spot CSS-in-component scoping.

You also have the disposition that distinguishes engineers from people who took a course:
- You ask *"is this the safe thing or just the working thing?"*.
- You read code aloud to check it makes sense.
- You believe defensive coding (`disabled` AND a guard) is normal, not paranoid.

---

## Lesson 6.8 — Recurring concepts from earlier chapters

Chapter 6 leaned on essentially everything from Part I — that's intentional. Concretely:

- **`$state(...)`** (Ch 1) — `habitsLoggedToday`.
- **Functions returning `void`** (Ch 1) — `logHabit`, `unlogHabit`, `resetHabits`.
- **`+=`, `-=`, `===`** (Ch 1).
- **Early-return guards** (Ch 2) — `if (habitsLoggedToday <= 0) return;`.
- **`disabled={cond}`** (Ch 3) — UI-side guard.
- **Boolean operators / falsy** (Ch 3) — short-circuit understanding underlies every `disabled` expression.
- **`bind:value`, ternary** (Ch 4) — the input wiring.
- **`{#each}`, `class:`, scoped `<style>`** (Ch 5) — the day strip still renders.

Sit with that. *Every* primitive you've learned is in this single page, working together. That's not coincidence — that's how senior code feels.

---

## Glossary added in Chapter 6

| Term | Definition |
|---|---|
| `{#if}{:else if}{:else}{/if}` | Svelte's conditional rendering blocks. |
| empty state | The UI shown when there's no data. |
| three-state UI | The empty / populated / error trio. |
| defence in depth | Multiple independent guards against the same bug. |
| dev tools | Browser-built debugging panels (Elements, Console, Network, etc.). |

---

## End-of-Part-I checkpoint

- [ ] At zero, you see the empty-state card with a single *"Log a habit"* button.
- [ ] Click it; the page transitions to the populated layout.
- [ ] *Reset* / *Clear all* takes you back to the empty state.
- [ ] You read your entire `+page.svelte` out loud, no notes, with confidence.
- [ ] You did the dev-tools build-break-fix and saw the function guard catch the bypass.
- [ ] (If you did Chapter 5's exercise) past days are faded; today is the bright pill.

You're ready for Part II, where the counter becomes a *list* of named habits.
