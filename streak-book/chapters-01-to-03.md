# The Streak Book — The May 5, 2026 Bible

## Part I — First words

> The reader writes their first SvelteKit page, learns the four operators that handle 90% of arithmetic, learns `if` and the ternary, and understands what reactivity feels like.
>
> By the end of Part I: the reader can write a `+page.svelte` with reactive state, an event handler, conditional rendering, and an undo button — and explain every line out loud in English.

---

# Chapter 1 — Your first reactive value

> *Today's job:* the home screen counts how many habits you've logged today, with a button to log one more. Click it; the number rises.

You have never written code before. You don't have to pretend otherwise. By the time this chapter ends — about an hour from now if you read carefully and type along — you will have:

- installed the same tools every senior engineer on this stack uses,
- created a real SvelteKit project that runs in your browser,
- written a small page that *responds* to you when you click it,
- and read every line of that page out loud, in English, the way an engineer would explain it to a colleague.

Not a toy. Not a "Hello, world." A real, running, reactive piece of an application called **Streak** that you will keep building, one chapter at a time, until it is deployed on the internet and you have signed up for it on your own production URL.

Let's start.

---

## Lesson 1.1 — What a *terminal* is, and how to open one

You're going to spend a lot of time talking to your computer through a small black window called a **terminal**. The terminal is a place where you type commands as text, hit Enter, and the computer does what you said. It looks intimidating. It isn't. There are about ten commands you'll use 95% of the time, and you already know what most of them mean in English.

> **terminal** *(noun)* — a window where you type commands to your computer as text, instead of clicking buttons. Sometimes called a *shell*, a *console*, or a *command line*. They all mean the same thing for our purposes.

Open yours now:

- **macOS** — Press `Cmd + Space`, type *Terminal*, press Enter. (Or, if you've installed iTerm2 or Warp, open that.)
- **Windows** — Press the Windows key, type *Terminal*, press Enter. If you have *Windows Terminal* installed, use that; if not, *PowerShell* will do.
- **Linux** — You know where it is.

You should see something like this:

```
billy@laptop ~ %
```

That blinking cursor at the end is your **prompt**. It's the terminal saying *"I'm ready — type something."* The text before the `%` (or `$`, or `>`) tells you who you are and where you are. We'll meet that *where* in a moment.

> **prompt** *(noun)* — the symbol the terminal shows when it's waiting for you to type a command. Usually `%`, `$`, or `>` followed by a blinking cursor.

Type this exactly, then press Enter:

```bash
echo "hello"
```

The terminal should reply:

```
hello
```

Read aloud what just happened: *"echo the text 'hello' back at me."* That's the terminal's job — you say things, it does them.

You can quit out of `echo`'s output by just typing the next command. The prompt is back, ready.

### The two terminal commands you absolutely need today

```bash
pwd
```

Read aloud: *"print working directory."* The terminal will reply with something like `/Users/billy` (macOS/Linux) or `C:\Users\billy` (Windows). This is the **directory** — the folder — where your terminal is currently sitting. Every command you type runs *from* this folder.

> **directory** *(noun)* — the same thing as a folder. Engineers say "directory" almost always.

```bash
ls
```

Read aloud: *"list."* You'll see the names of every file and folder in the current directory. (On Windows PowerShell, `dir` works too, but `ls` is universal — please use `ls`.)

That's it. With `pwd` ("where am I?") and `ls` ("what's around me?") you can already orient yourself anywhere on your computer. We'll meet `cd` ("change directory") in two pages.

---

## Lesson 1.2 — Installing `pnpm` (and why not `npm`)

Your project is going to need other people's code — small, well-tested pieces of software you didn't write. Those pieces are called **dependencies**, and the program that fetches them, organises them, and updates them is called a **package manager**.

> **dependency** *(noun)* — a piece of code your project uses but didn't write. Lives in a folder called `node_modules`. You almost never look in there directly; the package manager handles it for you.
>
> **package** *(noun)* — a single dependency, with a name and a version. Example: `svelte` is a package; `5.20.3` is its version.
>
> **package manager** *(noun)* — the program that installs, updates, and removes packages. Reads the list of what your project needs from `package.json`.

There are three popular package managers for the stack we're using: `npm`, `yarn`, and `pnpm`. **In this book we use `pnpm` exclusively.** Never `npm`. Never `yarn`. Reasons (you don't have to fully understand them yet, but they matter):

- `pnpm` is faster (often 2–3× faster than `npm`).
- `pnpm` uses dramatically less disk space because packages are stored once on your computer and *linked* into each project, not copied.
- `pnpm` is stricter about which dependencies a package is allowed to see — that prevents a class of bugs called *phantom dependencies* that bites senior engineers.
- The Bible rule (rule #1): *`pnpm` only.*

Install it now. Pick the line for your operating system, paste it into the terminal, hit Enter:

**macOS / Linux:**

```bash
curl -fsSL https://get.pnpm.io/install.sh | sh -
```

**Windows (PowerShell):**

```powershell
iwr https://get.pnpm.io/install.ps1 -useb | iex
```

If PowerShell refuses with an *execution policy* error, run this once first, then retry:

```powershell
Set-ExecutionPolicy -Scope CurrentUser -ExecutionPolicy RemoteSigned
```

This is a one-time per-user setting; it tells Windows you're allowed to run installer scripts. Senior habit on Windows: do it once on a fresh machine, never think about it again.

After it finishes, **close your terminal completely and open a fresh one.** This is important — the new terminal will know where `pnpm` lives; the old one won't.

Verify it works:

```bash
pnpm --version
```

You should see a version number, something like `9.15.0`. If you see *"command not found"*, the new-terminal step didn't take — close it again, open another fresh one, retry.

> If you get stuck here, take a screenshot, write down what you typed and what the terminal said, and walk away for ten minutes. Coming back fresh is faster than fighting an installer at 2am. Senior engineers do this constantly.

---

## Lesson 1.3 — Creating the Streak project

Now we make a real project. Pick a folder where you keep code — a lot of people use `~/projects` (macOS/Linux) or `C:\Users\you\projects` (Windows). If you don't have one, make one:

```bash
mkdir -p ~/projects
cd ~/projects
```

Read aloud: *"make a directory called projects (and don't error if it already exists). Then change into it."*

> **`mkdir`** — *make directory.* The `-p` is short for *parents* — it means "make any parent folders too if they're missing, and don't complain if it already exists."
>
> **`cd`** — *change directory.* Moves your terminal *into* the named folder. After this, `pwd` will show the new location.

Now scaffold the project. The current Svelte CLI is called `sv` (May 2026):

```bash
pnpm dlx sv create streak
```

> **`pnpm dlx`** — *download and execute.* Like running a one-off package without installing it permanently. We use it for installers and scaffolders.
>
> **scaffold** *(verb)* — to generate the empty starting structure of a project. Like a builder framing a house before they put walls on it.

The CLI will ask you a few questions, then show one big screen of optional **add-ons** as checkboxes. Answer exactly as below. (Wording may vary slightly between versions; match the *meaning*.)

| Question | Your answer |
|---|---|
| Which template? | **SvelteKit minimal** (the empty one — *not* the demo) |
| Type checking? | **Yes, TypeScript** |
| Add-ons (one screen, multi-select with space) | Tick **prettier**, **eslint**, **vitest**, **playwright**. Leave the rest. |
| Package manager? | **pnpm** |

When it finishes, it tells you to do this:

```bash
cd streak
pnpm install
pnpm dev
```

Do exactly that. Read aloud:

- *"change into the streak folder"*,
- *"install all the dependencies the project needs"* (this will take 10–30 seconds the first time, and creates a `node_modules` folder full of those dependencies),
- *"run the dev script"* — short for `pnpm run dev`. `pnpm` lets you drop the `run` for any script declared in `package.json`. We use the short form for the rest of the book.

You should see something like:

```
  VITE v6.X.X  ready in 412 ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
  ➜  press h + enter to show help
```

That `http://localhost:5173/` is the address of your project, running on your own computer. **Open it in your browser now.** If your terminal supports it, `Cmd+click` (macOS) or `Ctrl+click` (Windows/Linux) the URL. Otherwise copy-paste it into a fresh browser tab.

> **`localhost`** *(noun)* — the standard nickname for *your own computer*. Visiting `localhost:5173` means "talk to a server running on this very machine." It's only visible to you.
>
> **dev server** *(noun)* — the program (`pnpm run dev` started it) that watches your code, recompiles it when you save, and serves the result to your browser. It's running in your terminal *right now*. Don't close that terminal.

You should see a tiny welcome page. Excellent — Streak is alive.

---

## Lesson 1.4 — A short tour of what the scaffolder gave you

Open a *second* terminal window (the first one is busy running the dev server — leave it alone). In the new window:

```bash
cd ~/projects/streak
ls
```

You'll see something like:

```
README.md
e2e/
node_modules/
package.json
playwright.config.ts
src/
static/
svelte.config.js
tsconfig.json
vite.config.ts
```

Most of these you can ignore for now. The two you care about today:

- **`package.json`** — the list of every package this project depends on, and the *scripts* you can run. (`dev` is one of those scripts; that's why `pnpm dev` works.)
- **`src/`** — your code. Everything you write goes inside `src/`.

> **`.git`** — the scaffolder also created a hidden `.git` folder. `ls` won't show it; `ls -a` will. That's where Git tracks your project history. You don't touch it directly. We'll meet Git formally in Chapter 61.
>
> **`.gitignore`** — a hidden file listing things Git should *not* track (like `node_modules/`, which is huge and reproducible from `package.json`). Already filled in for you.

Look inside `src/`:

```bash
ls src
```

```
app.d.ts
app.html
lib/
routes/
```

What each one is, briefly:

- **`app.html`** — the HTML shell every page is rendered into. Has placeholders like `%sveltekit.head%`. You'll edit it once or twice in the whole book.
- **`app.d.ts`** — TypeScript declarations for global types (we'll fill it in Chapter 45 when auth needs `App.Locals`).
- **`lib/`** — your reusable code. Imports use the alias `$lib`.
- **`routes/`** — every URL of your site.

The one we open today is `src/routes/+page.svelte`.

> **`+page.svelte`** *(noun)* — a special filename SvelteKit recognises. The `+page.svelte` inside `src/routes/` becomes *the home page of your site*. The `+` at the start tells SvelteKit *"this is a routing convention, not just any file."*
>
> **route** *(noun)* — a URL on your site, paired with the file that serves it. The route `/` is served by `src/routes/+page.svelte`. The route `/about` would be served by `src/routes/about/+page.svelte`. We'll meet more in Part V.
>
> **convention** *(noun)* — a rule the framework agrees to follow if you do. SvelteKit's *file conventions* are things like *"a file named `+page.svelte` becomes a page."* They're how the framework reads your folder structure.

Open the project in your code editor. (If you don't have one, install **VS Code** — it's free and the best in class for this stack.) From inside the `streak` folder:

```bash
code .
```

If `code .` says "command not found", open VS Code by hand, then *File → Open Folder…* and pick `streak`. (To make `code` work next time: in VS Code press `Cmd+Shift+P` → type `Shell Command: Install 'code' command in PATH` → Enter.)

Also install the official **Svelte for VS Code** extension. The Extensions tab is the squares icon in the left sidebar; search for *Svelte*; install the one published by `svelte.dev`. This gives you syntax highlighting, type errors inline, and component autocomplete.

Find `src/routes/+page.svelte` in the file tree and click it. You'll see a tiny placeholder. We'll replace it shortly.

### One small migration: `svelte.config.js` → `svelte.config.ts`

The scaffolder gave you `svelte.config.js`. The Bible rule (#2) is *`.ts` only, never `.js`*. We migrate it now in 30 seconds so the rule holds from line one.

In the editor, **rename** `svelte.config.js` to `svelte.config.ts`. (Right-click → Rename, or `F2`.) Open the renamed file. Replace its contents with this:

```ts
// svelte.config.ts
import adapter from '@sveltejs/adapter-auto';
import { vitePreprocess } from '@sveltejs/vite-plugin-svelte';
import type { Config } from '@sveltejs/kit';

const config: Config = {
  preprocess: vitePreprocess(),
  kit: { adapter: adapter() },
};

export default config;
```

Read aloud: *"the Svelte config: use the auto adapter, preprocess with Vite, export it."*

The behaviour is identical to the `.js` version; we now have a typed config. Bible rule held.

(There's also `vite.config.ts` and `tsconfig.json`. Both stay as they are — Vite's config is already `.ts`, and `.json` is fine for `tsconfig.json` and `package.json` per Bible rule #2.)

---

## Lesson 1.5 — Writing the counter

Select everything in `src/routes/+page.svelte` and delete it. Then paste exactly this:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);

  function logHabit(): void {
    habitsLoggedToday += 1;
  }
</script>

<h1>Today</h1>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>
  Log a habit
</button>
```

Save the file with `Cmd+S` (macOS) or `Ctrl+S` (Windows/Linux). Now look at your browser at `http://localhost:5173/` — **without refreshing it.** You should see the new page appear automatically.

> **hot reload** *(noun)* — when the dev server notices you saved a file and updates the browser without you having to refresh. The first time you see it, it feels like magic. After a week, you can't live without it. SvelteKit (via Vite) does this for free.

Click the **Log a habit** button. The number rises. The word switches between *habit* (after the first click) and *habits* (every other count). You've shipped a working, reactive page.

That sentence is worth re-reading: **you've shipped a working, reactive page.** Reactive state, an event handler, conditional text — three of the things people new to web development find magical, all in one file.

---

## Lesson 1.6 — Reading the code aloud, line by line

This is the most important lesson in the entire book, and it's going to repeat in every chapter. The point of reading code aloud is to map symbols → meaning. Not symbol → sound. *Meaning.*

Read each line below out loud, in English, before you read the explanation that follows it. Go slowly.

| The line | Read aloud as |
|---|---|
| `<!-- src/routes/+page.svelte -->` | *"This is the home page of the site."* (HTML comment — for our eyes only.) |
| `<script lang="ts">` | *"Begin a script block written in TypeScript."* |
| `let habitsLoggedToday = $state(0);` | *"Let the count of habits logged today be reactive state, starting at zero."* |
| `function logHabit(): void {` | *"Define a function called logHabit; it returns nothing."* |
| `habitsLoggedToday += 1;` | *"Increase the count by one."* |
| `}` | *"End of the function."* |
| `</script>` | *"End of the script block."* |
| `<h1>Today</h1>` | *"A big heading that says Today."* |
| `<p>` | *"Begin a paragraph."* |
| `You've logged` | *"The literal text 'You've logged'."* |
| `<strong>{habitsLoggedToday}</strong>` | *"In bold, show the current count."* |
| `{habitsLoggedToday === 1 ? 'habit' : 'habits'}` | *"If the count is exactly one, the word is 'habit'; otherwise 'habits'."* |
| `so far.` | *"The literal text 'so far.'."* |
| `</p>` | *"End of the paragraph."* |
| `<button type="button" onclick={logHabit}>` | *"A button — when clicked, run the logHabit function."* |
| `Log a habit` | *"The button's label."* |
| `</button>` | *"End of the button."* |

Now read the *whole file* out loud, top to bottom, no notes. If you stumble, that's normal. Go again. By the third pass it should feel almost natural.

---

## Lesson 1.7 — The new vocabulary

You just saw a lot of new words. Here's each one with a one-sentence definition. We'll be re-using them constantly.

> **`<script lang="ts">`** — the wrapper around TypeScript code in a `.svelte` file. The `lang="ts"` part tells the editor and the build tools *"this is TypeScript, not plain JavaScript."*
>
> **TypeScript** — JavaScript with a type system bolted on. We use it everywhere; it catches a huge class of bugs at the moment you type the wrong thing, before the browser ever sees the code.
>
> **`let`** — a keyword that introduces a *variable* — a named container for a value. *"Let `habitsLoggedToday` be …"*
>
> **variable** — a name pointing at a value. The name stays the same; the value can change.
>
> **`$state(...)`** — a *rune* (a special Svelte thing). It marks the value as *reactive*. When you change a `$state` value, every place that reads it on the page updates automatically.
>
> **rune** — a special Svelte 5 keyword that starts with `$`. Runes are how you tell Svelte *"this isn't ordinary code — please do something special with it."* You'll meet `$state`, `$derived`, `$effect`, `$props`, `$bindable`, `$inspect` in this book.
>
> **reactive** — the property of a value that, when changed, causes anything watching it to update. The whole reason we use Svelte instead of writing pages by hand.
>
> **function** — a reusable instruction. You give it a name (`logHabit`), you specify what goes in (here, nothing), what comes out (here, nothing — `: void`), and what it does (the lines between `{` and `}`).
>
> **`: void`** — the type "this function returns nothing." We write it explicitly because being explicit about return types catches bugs and reads as professional intent.
>
> **`+=`** — the *compound assignment operator* meaning *"increase by"*. `x += 1` and `x = x + 1` produce the same result. We always write `+=` because it states the intent ("increment") instead of restating the variable.
>
> **`{x}`** — inside markup (the HTML-looking part of a `.svelte` file), curly braces mean *"insert the value of `x` here."*
>
> **`===`** — *strict equality*. Compares value AND type. We always use `===`, never `==`. (`==` does silent type coercion that has caused real production bugs; `1 == '1'` is `true`. Never.)
>
> **ternary** — the `condition ? thenValue : elseValue` expression. It's an inline if/else for picking one of two values.
>
> **`onclick={fn}`** — a Svelte 5 event binding. When the user clicks this element, run `fn`. **Lowercase `onclick`** — the modern syntax. Some old Svelte tutorials use `on:click`; we never do.
>
> **`type="button"`** — an HTML attribute. *"This button is a plain button, not a form-submit button."* When buttons are inside forms (we'll meet forms in Ch 41), the default is `submit`, which causes silent bugs. We always write `type="button"` unless we genuinely want submit.

---

## Lesson 1.8 — Why we wrote it the way we did (and not the way books usually show)

A typical beginner book would have written this:

```ts
let count = 0;
count = count + 1;
```

We wrote:

```ts
let habitsLoggedToday = $state(0);
habitsLoggedToday += 1;
```

Six choices in there are deliberate, principal-engineer-level habits. They will save you real bugs across the rest of this book. Each one is small. Together they're the reason you'll be reading senior code by chapter 12.

**1. The variable is named after what it represents.** Not `count`. `habitsLoggedToday`. When you read this line in chapter 14 — *"send `habitsLoggedToday` to the server"* — the line tells the truth about what's flowing. `count` would have lied. *Names that lie are the most expensive kind of bug.*

**2. We use `+=`, not `count = count + 1`.** Same answer; cleaner intent. `+=` says *"increase by"*. The longer form makes your eyes travel the variable name twice for no reason. Engineers reading a diff want the intent on the page in the fewest characters that still read like English. From this lesson onward: every "add to a number that already exists" uses `+=`. Every "subtract from" uses `-=`. Every "multiply" uses `*=`. The shape of the operator matches the shape of the thought.

**3. We use `===`, not `==`.** `==` does silent type-coercion that has caused real production incidents. `1 == '1'` is `true`. `0 == ''` is `true`. `null == undefined` is `true`. We're not interested. `===` compares without funny business. We never write `==` in this book.

**4. The function has a return type.** `function logHabit(): void` — the `: void` part. We're being explicit even though TypeScript could infer it. Reasons: (a) at code-review time, the return type tells the reviewer at a glance whether the function is supposed to return a value; (b) if someone later edits the function and accidentally `return`s something, the compiler will flag the contradiction. Habit on day one is habit forever.

**5. The button has `type="button"`.** Right now the button isn't inside a `<form>`, so it doesn't matter. But chapter 41 will introduce forms. The day you put a `<button>` inside a `<form>` without `type="button"`, browsers default it to `type="submit"` and your button will silently submit the form when clicked. That's a future bug. Writing `type="button"` now costs nothing and prevents it. *Senior habit: write the safer thing always, even when it doesn't matter yet.*

**6. The plural-aware text.** `{habitsLoggedToday === 1 ? 'habit' : 'habits'}` — we never display *"You've logged 1 habits"*. Tiny detail. But the user will see this 50 times today; if it reads "habits" when it should read "habit" once, your app feels amateurish. Senior engineers care about how their work *reads*.

You won't internalise all six on the first read. That's fine. The book repeats every one of them — in different real contexts — until they're reflex.

---

## Lesson 1.9 — Read this code

Two short snippets. Read each one out loud, then answer the question. Don't peek at the answer until you've tried.

### Snippet A

```svelte
<script lang="ts">
  let coffeesToday = $state(2);
</script>

<p>{coffeesToday} {coffeesToday === 1 ? 'coffee' : 'coffees'} today.</p>
```

**Question:** What does the page show?

<details>
<summary>Answer</summary>

It shows: *"2 coffees today."*

The variable starts at 2; the ternary picks `'coffees'` because 2 is not exactly 1. There's no button, so nothing changes — but the text is still rendered using the same reactive pattern you wrote.
</details>

### Snippet B

```svelte
<script lang="ts">
  let unreadMessages = $state(0);

  function markOneRead(): void {
    unreadMessages -= 1;
  }
</script>

<button type="button" onclick={markOneRead}>
  Mark one read ({unreadMessages} unread)
</button>
```

**Question:** What's wrong with this code? (Hint: think about what happens after enough clicks.)

<details>
<summary>Answer</summary>

`unreadMessages` starts at 0. Click the button once and it becomes `-1`. Click it twice and it becomes `-2`. There's no guard preventing it from going negative. *"Negative one unread messages"* is nonsense.

That's *exactly* the kind of bug Chapter 2 will fix using `if`.
</details>

If you got the second one without peeking, you've already noticed the thing the next chapter is about. Good.

---

## Lesson 1.10 — Now you write it

Your turn.

**The English sentence first.** Before you type anything, read this sentence aloud:

> *"Add a second button below the first that **decreases** `habitsLoggedToday` by one — and don't worry yet about it going negative; we'll handle that in Chapter 2."*

Now translate it. You need:

- a second function (let's call it `unlogHabit`),
- using the *"decrease by"* operator (it's `-=`, the same shape as `+=`),
- a second `<button type="button">` element,
- with `onclick={unlogHabit}`,
- and a sensible label like *"Undo"*.

Type it in. Save. Click. Watch the number go up and down. Notice that — already — clicking *Undo* enough times sends the count negative. Sit with that for a moment. It's the perfect lead-in to Chapter 2.

When you're done, your `+page.svelte` should be roughly the shape we started with, plus the new pieces you wrote. Don't peek at the worked answer below until you've tried.

<details>
<summary>Worked answer</summary>

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);

  function logHabit(): void {
    habitsLoggedToday += 1;
  }

  function unlogHabit(): void {
    habitsLoggedToday -= 1;
  }
</script>

<h1>Today</h1>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>
  Log a habit
</button>

<button type="button" onclick={unlogHabit}>
  Undo
</button>
```

If your version looks different in small ways (a different function name, a different button label) but it works — that's fine. The point is the shape: a second `let`-free function (we kept the same `let habitsLoggedToday`), a second `function` declaration, a second `<button>`. Two of each, mirrored.
</details>

---

## Lesson 1.11 — What you can now read in the wild

You've absorbed a surprising amount today, often without being lectured at. Take a victory lap.

You can now, when you encounter them in someone else's open-source code:

- **Recognise a `.svelte` file** and know the three sections (`<script>`, markup, optional `<style>`).
- **Spot `<script lang="ts">`** and know it means TypeScript.
- **Read `let foo = $state(initial);`** and know it means *"a reactive variable"*.
- **Read `function fooBar(): void { ... }`** and know it means *"a function that returns nothing"*.
- **Read `+=`, `-=`, `===`** and translate them aloud.
- **Read `{value}` inside markup** and know it means *"insert this value"*.
- **Read `<button onclick={fn}>`** and know it means *"on click, run `fn`"*.
- **Read `condition ? a : b`** and know it picks `a` when the condition is true, `b` otherwise.

That's about a third of what you'll need to read most modern Svelte 5 components on GitHub. We'll have the other two-thirds by the end of Part III.

You also have a feel — a feel, not yet a vocabulary — for:

- *Reactivity* (you change a value, the screen updates),
- *Events* (you click a button, a function runs),
- *Conditional rendering* (the word *habit* vs *habits* changing live),
- *Hot reload* (you save, the browser updates without refreshing).

Three of those four are the things people new to web development find magical. You wrote all three on day one.

---

## Glossary added in Chapter 1

A running list. Every chapter adds to it. By the end of the book you'll have a fluent vocabulary covering the full Svelte 5 + SvelteKit + TypeScript + Postgres + Stripe stack.

| Term | One-sentence definition |
|---|---|
| terminal | A window where you type commands as text. |
| prompt | The blinking-cursor symbol indicating the terminal is ready for a command. |
| directory | A folder on your computer. |
| `pwd` | The terminal command for *"where am I?"* |
| `ls` | The terminal command for *"what's around me?"* |
| `mkdir` | The terminal command for *"make a directory."* |
| `cd` | The terminal command for *"change directory."* |
| dependency | A piece of code your project uses but didn't write. |
| package | A single dependency, with a name and a version. |
| package manager | The program that installs/updates/removes packages. |
| `pnpm` | The package manager we use in this book. |
| scaffold | To generate the empty starting structure of a project. |
| `localhost` | Nickname for *your own computer*, used as a server address. |
| dev server | The program that watches your code and serves it to the browser. |
| route | A URL paired with the file that serves it. |
| convention | A rule the framework follows if you follow it too. |
| `+page.svelte` | The SvelteKit file that becomes the page at `/`. |
| TypeScript | JavaScript with types, used to catch bugs early. |
| `<script lang="ts">` | The wrapper around TypeScript code in a `.svelte` file. |
| `let` | The keyword that introduces a variable. |
| variable | A name pointing at a value. |
| rune | A `$`-prefixed Svelte 5 keyword. |
| `$state` | The rune that marks a value as reactive. |
| reactive | A value whose changes auto-update the page. |
| function | A reusable named instruction. |
| `: void` | The "returns nothing" type annotation. |
| `+=` | Compound assignment: *"increase by"*. |
| `-=` | Compound assignment: *"decrease by"*. |
| `===` | Strict equality: same value AND same type. |
| ternary | The `cond ? a : b` inline if/else. |
| `{x}` | Inside markup: *"insert the value of `x`"*. |
| `onclick={fn}` | Svelte 5 event binding for clicks. |
| `type="button"` | The "this is a plain button" attribute. |
| hot reload | Browser auto-update on file save. |

---

## End-of-chapter checkpoint

Before moving on to Chapter 2, you should be able to do all of these without looking back:

- [ ] Open a terminal, run `pwd`, run `ls`, run `cd`.
- [ ] Tell someone what `pnpm` is and why we use it.
- [ ] Start the dev server with `pnpm run dev` and open `http://localhost:5173/`.
- [ ] Open `src/routes/+page.svelte` in your editor.
- [ ] Read the entire file from your version of Streak out loud, in plain English, with no notes.
- [ ] Explain why we write `+=` instead of `= x + 1`.
- [ ] Explain why we write `===` instead of `==`.
- [ ] Explain why we write `type="button"` even when it doesn't matter yet.

If any of those are hesitant, reread the lesson they came from. There's no rush. Chapter 2 is short.

---

# Chapter 2 — `if`, and why we use early returns

> *Today's job:* the **Undo** button you wrote at the end of Chapter 1 must not let `habitsLoggedToday` go below zero. *"Negative one habits"* is nonsense; the button should simply do nothing when there's nothing to undo.

If you finished Chapter 1's exercise, you already noticed the bug. Click *Undo* enough times and the count goes negative. We're going to fix that with the most fundamental decision-making tool in any programming language: **`if`**.

But more importantly, we're going to learn the *style* in which senior engineers write `if`. Because there are two ways to write the same logic, and the difference between them is the difference between code that's pleasant to read at 2am and code that isn't.

---

## Lesson 2.1 — The `if` statement

Here's `if` in its plainest form:

```ts
if (condition) {
  // do something
}
```

Read aloud: *"if the condition is true, do this thing."*

In Streak's case, our condition is *"there's at least one habit to undo."* In code:

```ts
if (habitsLoggedToday > 0) {
  habitsLoggedToday -= 1;
}
```

Read aloud: *"if today's count is greater than zero, decrease it by one."*

Six new things in two lines. Let's name them.

> **`if`** — the keyword that starts a conditional. Followed by `(...)`, then `{...}`.
>
> **`>`** — *greater than*. Returns `true` if the left side is bigger than the right.
>
> **`<`** — *less than*. The mirror of `>`.
>
> **`>=`** — *greater than or equal to*.
>
> **`<=`** — *less than or equal to*.
>
> **`!==`** — *strict inequality*. The opposite of `===`. Same rule applies: never `!=`, always `!==`.
>
> **block** — the `{...}` part. The lines inside the braces only run if the condition is true.

A few rules we'll keep for the entire book:

- **Use braces for any `if` whose body has more than one statement, ever.** Some languages let you write `if (x > 0) doThing(); doOtherThing();` with the second line silently *not* part of the `if`. We never write that ambiguous form.
- **For pure guard-clause `return` / `throw` / `break`, single-line is acceptable.** `if (trimmed === '') return;` is idiomatic and used throughout this book's helper functions. The rule of thumb: single-line is fine *only* when the body is one short statement that exits the function (or breaks the loop). Anything with state mutation, side effects, or multiple statements gets braces.
- **The condition has parentheses.** Always. This isn't optional.
- **The opening brace is on the same line as `if`.** This is just a convention; consistency matters more than which side you pick. We pick same-line.

---

## Lesson 2.2 — Wiring it into Streak

Replace your `unlogHabit` function with this:

```svelte
<script lang="ts">
  let habitsLoggedToday = $state(0);

  function logHabit(): void {
    habitsLoggedToday += 1;
  }

  function unlogHabit(): void {
    if (habitsLoggedToday > 0) {
      habitsLoggedToday -= 1;
    }
  }
</script>
```

Save (`Cmd+S` / `Ctrl+S`). Click *Undo* once at zero — nothing happens. Click *Log a habit* twice — count is two. Click *Undo* twice — count is zero. Click *Undo* a third time — nothing happens. Click *Undo* forty more times — still zero. Bug fixed.

Read the new function aloud:

| Line | Read aloud |
|---|---|
| `function unlogHabit(): void {` | *"Define unlogHabit; it returns nothing."* |
| `if (habitsLoggedToday > 0) {` | *"If today's count is greater than zero,"* |
| `habitsLoggedToday -= 1;` | *"decrease it by one."* |
| `}` | *"end of the if-block."* |
| `}` | *"end of the function."* |

---

## Lesson 2.3 — The two ways to write the same logic

Now we get to the lesson that actually matters in this chapter.

Look at this version of `unlogHabit` — same behaviour, written differently:

```ts
function unlogHabit(): void {
  if (habitsLoggedToday <= 0) {
    return;
  }

  habitsLoggedToday -= 1;
}
```

Read aloud: *"If there's nothing to undo, we're done. Otherwise, decrease by one."*

This is called an **early return**. The function checks for the *unhappy* case first; if it's hit, the function exits immediately (`return;` with no value, because the function returns nothing). Anything below that point only runs when the happy case is true.

Compare side by side:

```ts
// Version A — the nested style
function unlogHabit(): void {
  if (habitsLoggedToday > 0) {
    habitsLoggedToday -= 1;
  }
}
```

```ts
// Version B — the early-return style
function unlogHabit(): void {
  if (habitsLoggedToday <= 0) {
    return;
  }

  habitsLoggedToday -= 1;
}
```

Both produce identical behaviour. Both are correct. **From now on we always write Version B.** Reasons:

1. **Version B reads top-to-bottom like a sentence.** *"If there's nothing to undo, we're done. Otherwise, decrease."* Version A makes you mentally hold the condition open while you read what's inside the brace.
2. **Version B keeps the "real work" of the function unindented.** As soon as a function gains *two* preconditions, Version A becomes a pyramid; Version B stays flat.
3. **Version B is how senior engineers write.** Reading their code becomes easier when you write the same way.

This pattern has a name: the **guard clause**.

> **guard clause** *(noun)* — an `if`-with-early-return at the top of a function that handles a precondition. *"If the precondition is wrong, bail out now; from this point onward we can assume the happy case."*

Update your file to Version B:

```svelte
<script lang="ts">
  let habitsLoggedToday = $state(0);

  function logHabit(): void {
    habitsLoggedToday += 1;
  }

  function unlogHabit(): void {
    if (habitsLoggedToday <= 0) {
      return;
    }

    habitsLoggedToday -= 1;
  }
</script>
```

(Notice the blank line after the `if`-block. That's a small senior habit too — visually separates the guard from the work.)

---

## Lesson 2.4 — Read this code

Three versions of the same function. Read each out loud, then rank them from easiest-to-read to hardest. There are no tricks; one of them is genuinely the best, one is mediocre, one is bad.

### Version 1

```ts
function deleteAccount(user: User): void {
  if (user.isLoggedIn) {
    if (user.confirmedDeletion) {
      if (!user.hasUnpaidInvoices) {
        actuallyDelete(user);
      }
    }
  }
}
```

### Version 2

```ts
function deleteAccount(user: User): void {
  if (!user.isLoggedIn) return;
  if (!user.confirmedDeletion) return;
  if (user.hasUnpaidInvoices) return;

  actuallyDelete(user);
}
```

### Version 3

```ts
function deleteAccount(user: User): void {
  if (user.isLoggedIn && user.confirmedDeletion && !user.hasUnpaidInvoices) {
    actuallyDelete(user);
  }
}
```

<details>
<summary>The ranking, with reasons</summary>

**Best: Version 2.** Each precondition is its own line; each reads like English. *"If they're not logged in, we're done. If they haven't confirmed, we're done. If they have unpaid invoices, we're done. Otherwise, delete."* The "real work" is the last line — unindented, easy to find.

**Second: Version 3.** It's compact, but the moment any of those three checks needs special handling (a different error message, a logging call), the whole thing has to be rewritten as Version 2 anyway. Compactness now becomes pain later.

**Worst: Version 1.** The pyramid. Three levels deep. To understand whether `actuallyDelete` runs, you have to mentally hold three open conditions at once. If a fourth check is added, the pyramid grows. This style is called *"arrow code"* and senior engineers go out of their way to refactor it on sight.

If you ranked them differently, that's fine; this is taste, not law. But the consensus among engineers reading code at 3am is: Version 2 is the kindest.
</details>

---

## Lesson 2.5 — Build, break, fix

Time to deliberately break the code so you feel the bug, then unbreak it.

In your `unlogHabit`, change the comparison from `<=` to `<`:

```ts
function unlogHabit(): void {
  if (habitsLoggedToday < 0) { // <- changed from <= to <
    return;
  }

  habitsLoggedToday -= 1;
}
```

Save. Now click *Log a habit* once — count is `1`. Click *Undo* once — count is `0`. Click *Undo* again. What happens?

The count goes to `-1`. Click *Undo* a third time — `-2`. The bug is back.

Why? Because `<` is *strictly less than*. `0 < 0` is `false`, so the guard doesn't trigger when the count is exactly zero. We needed `<=` (less than *or equal to*) to bail out at zero too.

Read aloud the difference:
- `if (x <= 0) return;` → *"If x is zero or negative, we're done."*
- `if (x < 0) return;` → *"If x is already negative, we're done."* — but `0` isn't negative, so we keep going and decrement to `-1`.

This is called an **off-by-one error** — the most common bug class in *boundary comparison* code. You'll write one of these every couple of weeks for the rest of your career; learning to *feel* the boundary cases now is part of the work.

Change it back to `<=`. Click *Undo* a few times to confirm it's stuck at zero. Done.

> **off-by-one error** — a bug where a boundary value (zero, the last index of a list, the final iteration of a loop) is included when it shouldn't be, or excluded when it should be. The classic. Look for them every time you write a comparison.

---

## Lesson 2.6 — Now you write it

Your turn again.

**The English sentence first:**

> *"Add a 'Reset' button that sets `habitsLoggedToday` back to zero — but only if it's currently greater than zero. There's no point doing work that doesn't need doing."*

The bones you'll need:
- a third `function`, called `resetHabits`, returning `void`,
- a guard clause that exits early if `habitsLoggedToday` is already zero (or negative — though it can't be, given Chapter 2's other guard),
- a single line of work: `habitsLoggedToday = 0;` (this *is* a reassignment, so we use `=`, not `+=` — there's nothing to add to),
- a third `<button type="button">` with `onclick={resetHabits}` and a label like *"Reset"*.

Try it before you peek.

<details>
<summary>Worked answer</summary>

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);

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

<h1>Today</h1>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>Log a habit</button>
<button type="button" onclick={unlogHabit}>Undo</button>
<button type="button" onclick={resetHabits}>Reset</button>
```

A small thing to notice: `resetHabits` and `unlogHabit` have the *same* guard clause — `if (habitsLoggedToday <= 0) return;`. That repetition is not a problem yet; we'd only extract a shared helper if a third function needed it (the *Rule of Three* — a senior habit we'll meet formally later).

Right now, the duplication tells the truth: *"both functions need the same precondition, and that precondition is one line."* Premature abstraction is worse than honest duplication.
</details>

---

## Lesson 2.7 — Recurring concepts from earlier chapters

Chapter 2 used these things you already knew from Chapter 1, in new ways:

- **`function … (): void`** (Ch 1) — every new function we wrote keeps the explicit `void` return type.
- **`-=`** (Ch 1) — the *"decrease by"* operator inside the guard.
- **`type="button"`** (Ch 1) — every new button still has it.
- **`onclick={fn}`** (Ch 1) — wiring the new buttons.

Repetition in *new contexts* is how fluency builds. By chapter 6 you'll have used these primitives in a dozen different shapes.

---

## Lesson 2.8 — What you can now read in the wild

After Chapter 2, when you encounter someone else's code, you can:

- **Read `if (...) { ... }`** and explain its branches.
- **Read all six comparison operators** (`<`, `<=`, `>`, `>=`, `===`, `!==`) and explain what they return.
- **Recognise the early-return / guard-clause style** as the senior pattern.
- **Spot pyramid (arrow) code** as a refactor target.
- **Spot likely off-by-one errors** when you see boundary comparisons.

You've also internalised something subtle: the idea that *the way code reads* matters separately from *whether the code works*. That's the line between programmer and engineer.

---

## Glossary added in Chapter 2

| Term | Definition |
|---|---|
| `if` | The keyword that starts a conditional. |
| `>`, `<`, `>=`, `<=` | Numeric comparison operators. |
| `!==` | Strict inequality (the opposite of `===`). |
| block | The `{...}` after an `if`, function, etc. |
| early return | Exiting a function early when a precondition fails. |
| guard clause | An `if`-with-early-return that handles a precondition. |
| arrow code | Deeply nested `if` blocks, considered bad style. |
| off-by-one error | A bug at a boundary value. |

---

## End-of-chapter checkpoint

- [ ] You wrote `unlogHabit` with a guard clause.
- [ ] You added a `resetHabits` button that follows the same pattern.
- [ ] You can explain *out loud* why early-return reads better than nested `if`.
- [ ] You felt the off-by-one bug yourself by changing `<=` to `<` and watching the count go negative.
- [ ] You changed it back.

---

# Chapter 3 — Booleans and the four operators that matter

> *Today's job:* when there's nothing meaningful to do, the buttons grey out — they're disabled. Specifically, *Undo* and *Reset* should be unclickable when `habitsLoggedToday` is zero. The user shouldn't be able to even *try* to do something that won't work.

This is a small visual change with a surprisingly deep lesson behind it. We're going to meet the **boolean** type — values that are either `true` or `false` — and the four logical operators (`&&`, `||`, `!`, and a bonus one we'll preview). We're also going to set up Chapter 4's punchline by showing you the foot-gun that `||` becomes when combined with strings.

---

## Lesson 3.1 — `boolean`, `true`, `false`

A **boolean** is a value that has exactly two possibilities: `true` or `false`.

You've already seen booleans without knowing it. *Every comparison returns a boolean.*

```ts
habitsLoggedToday > 0   // returns either true or false
habitsLoggedToday === 1 // returns either true or false
```

You can store one in a variable:

```ts
let isLoggedIn = $state(false);
let canSubmit = $state(true);
```

Read aloud: *"let isLoggedIn be reactive state, starting at false."*

> **`boolean`** — a type whose values are only `true` or `false`. (Lowercase, like all primitive types in TypeScript.)
>
> **`true` / `false`** — the two literal values of the boolean type. Lowercase. **Never `True` or `TRUE`.** Those would be syntax errors.

In TypeScript, you can be explicit about the type:

```ts
let canSubmit: boolean = $state(true);
```

But you usually don't need to — TypeScript figures out from `true` that the type is `boolean` automatically. We use explicit annotations when they aid readability or when TypeScript can't infer (we'll see those cases later).

---

## Lesson 3.2 — The three logical operators (and the bonus fourth)

There are three logical operators that combine booleans, plus a fourth coming next chapter that combines a value with a fallback. Their truth tables are below — you should fill them in by hand on paper before reading the answers.

### `&&` — *AND*

```ts
true && true     // ???
true && false    // ???
false && true    // ???
false && false   // ???
```

Read `&&` aloud as *"and"*. The result is `true` only when **both** sides are `true`.

<details>
<summary>Answers</summary>

```ts
true && true     // true
true && false    // false
false && true    // false
false && false   // false
```
</details>

### `||` — *OR*

```ts
true || true     // ???
true || false    // ???
false || true    // ???
false || false   // ???
```

Read `||` aloud as *"or"*. The result is `true` when **either** side is `true`.

<details>
<summary>Answers</summary>

```ts
true || true     // true
true || false    // true
false || true    // true
false || false   // false
```
</details>

### `!` — *NOT*

```ts
!true     // ???
!false    // ???
```

Read `!` aloud as *"not"*. It flips the boolean.

<details>
<summary>Answers</summary>

```ts
!true     // false
!false    // true
```
</details>

### `??` — *NULLISH COALESCING* (preview, full coverage Chapter 4)

`??` is *not* a boolean operator in the same sense as the other three; it deals with *missing* values. We'll meet it formally next chapter. Mentioning it now so you've seen the symbol when it shows up.

---

## Lesson 3.3 — Short-circuit evaluation

Here's a senior-engineer-level fact about `&&` and `||` you'll want to know on day one:

> **They stop evaluating as soon as the answer is determined.** This is called *short-circuit evaluation.*

- `false && anything` — JavaScript doesn't even look at `anything`; the answer is `false` regardless. So `&&` *stops at the first false*.
- `true || anything` — JavaScript doesn't even look at `anything`; the answer is `true` regardless. So `||` *stops at the first true*.

This matters for safety. Look at this:

```ts
if (user !== null && user.name === 'Billy') {
  // ...
}
```

If `user` is `null`, the first half is `false`, so JavaScript never evaluates `user.name`. Which is good — accessing `.name` on `null` would crash. The `&&` *guards* the second half. This is a pattern you'll see in essentially every codebase.

We won't lean on this trick in Chapter 3 because we have a better tool (`?.` — optional chaining) coming in Chapter 4. But you should recognise it when you read it.

---

## Lesson 3.4 — Disabling buttons

Now the application work. We want *Undo* and *Reset* to be visually disabled when there's nothing to undo. HTML supports this directly via the `disabled` attribute:

```html
<button type="button" disabled>Can't click me</button>
```

A button with `disabled` is greyed out, doesn't fire `onclick`, and is skipped by keyboard tab-navigation. Browsers handle the styling and the behaviour; we just need to set the attribute.

In Svelte, we want `disabled` to be **derived from state** — *"the button is disabled when the count is zero."* Here's the cleanest way:

```svelte
<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday === 0}>
  Undo
</button>
```

Read aloud: *"a button — when clicked, run unlogHabit. It's disabled when today's count is exactly zero."*

The expression inside `disabled={...}` is a boolean. `habitsLoggedToday === 0` evaluates to `true` or `false` depending on the current state. Because `habitsLoggedToday` is a `$state` (Chapter 1), this re-evaluates automatically whenever the count changes. Click *Log* — *Undo* lights up. Click *Undo* until zero — *Undo* greys out.

Update your file:

```svelte
<!-- src/routes/+page.svelte -->
<script lang="ts">
  let habitsLoggedToday = $state(0);

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

<h1>Today</h1>

<p>
  You've logged
  <strong>{habitsLoggedToday}</strong>
  {habitsLoggedToday === 1 ? 'habit' : 'habits'} so far.
</p>

<button type="button" onclick={logHabit}>Log a habit</button>

<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday === 0}>
  Undo
</button>

<button type="button" onclick={resetHabits} disabled={habitsLoggedToday === 0}>
  Reset
</button>
```

Save (`Cmd+S` / `Ctrl+S`). Watch the buttons grey in and out as you click *Log a habit* and *Undo*. That's a real piece of UX you've shipped.

---

## Lesson 3.5 — A pattern we'll name properly later

We have the same condition in two places: `disabled={habitsLoggedToday === 0}` on both *Undo* and *Reset*. Two repetitions isn't a problem (the *Rule of Three* — only extract when you have three users of a piece of logic), but it's worth naming the pattern.

What we *want* is a single name — something like `hasNothingToUndo` — that automatically tracks the count. Plain `const` doesn't reach because `const` is computed once, at component-mount time:

```ts
// ❌ doesn't update when habitsLoggedToday changes
const hasNothingToUndo: boolean = habitsLoggedToday === 0;
```

The right tool is a **derived value** — a special kind of variable that re-computes whenever its dependencies change. Svelte 5 calls this `$derived`. We meet it formally in **Chapter 16**, when we have a real reason for it (a "habits done today" total computed from a list).

For now, **inline is fine.** Senior habit: don't reach for an abstraction before you have the right primitive. Repetition that tells the truth (*"both buttons share this guard"*) is better than a workaround.

---

## Lesson 3.6 — Reading senior code: the `||` foot-gun

Here's a piece of code you'll see in someone else's project. It looks innocent. It's been the source of real production bugs.

```ts
const displayName = user.name || 'Anonymous';
```

The author's intent: *"if `user.name` is missing, default to 'Anonymous'."*

Read aloud, that sounds reasonable. So what's wrong?

The problem is that `||` doesn't ask *"is the left side missing?"* — it asks *"is the left side falsy?"*. And in JavaScript, several non-missing values are falsy:

- `0` is falsy
- `''` (the empty string) is falsy
- `false` is falsy
- `null` is falsy
- `undefined` is falsy
- `NaN` is falsy

So the moment `user.name` is `''` (an empty string — *"the user explicitly chose a blank name"*) or even just zero (in a different field — say a `score`), `||` falls back to the default. The user sees *Anonymous* even though they have a name field, just an empty one.

This is the **`||` foot-gun**, and it's the reason the next chapter formally introduces `??` (the nullish coalescer). `??` only falls back when the left side is `null` or `undefined` — true *missing* values. It's the right tool for the job; `||` is the wrong one *most of the time*.

For Chapter 3, you only need to know:

- `||` falls back on **any falsy value**, including `0` and `''`.
- That's almost never what you want.
- Chapter 4 fixes this with `??`.

Senior engineers reach for `??` reflexively whenever they want a *default for missing*. The reason that habit exists is because they've all had the `||` foot-gun shoot them at least once.

> **falsy** *(adjective)* — a value that JavaScript treats as `false` in a boolean context. Specifically: `false`, `0`, `''`, `null`, `undefined`, `NaN`, and `-0`. Everything else is **truthy**.

---

## Lesson 3.7 — Read this code

Three small snippets. Predict the output of each before peeking.

### Snippet A

```ts
const a = true;
const b = false;
const c = a && !b;
console.log(c);
```

<details>
<summary>Answer</summary>

`true`. `!b` is `!false` which is `true`. Then `a && true` is `true && true`, which is `true`.
</details>

### Snippet B

```ts
const score = 0;
const displayScore = score || 'No score yet';
console.log(displayScore);
```

<details>
<summary>Answer</summary>

`'No score yet'` — and that's the bug. The user has a real score of zero. The `||` operator treats `0` as falsy and falls back to the string. If the author had used `??` here, it would correctly print `0`.
</details>

### Snippet C

```svelte
<script lang="ts">
  let isOnline = $state(true);
  let hasUnsavedChanges = $state(false);
</script>

<button type="button" disabled={!isOnline || hasUnsavedChanges}>
  Publish
</button>
```

When is the button *disabled*? Read it carefully.

<details>
<summary>Answer</summary>

The button is disabled when **either**:
- the user is offline (`!isOnline` is `true`), OR
- there are unsaved changes (`hasUnsavedChanges` is `true`).

In other words, the button is *enabled* only when the user is online AND there are no unsaved changes. This is a real-world UX pattern — you can't publish if you're offline, and you can't publish stale data.

**A note on `||` here.** Lesson 3.6 will warn you about the `||` foot-gun — but that's about using `||` to *default a missing value* (`name || 'Anonymous'`). That's the wrong tool there. Using `||` to combine *two booleans* (as in `!isOnline || hasUnsavedChanges`) is exactly the right tool. The line is *type*-shaped: `||` between booleans is great; `||` for missing-value defaults is wrong. By chapter 4 you'll feel the difference.
</details>

---

## Lesson 3.8 — Now you write it

**The English sentence first:**

> *"The 'Log a habit' button should be disabled if the count has somehow become negative — even though our guards make that impossible right now, defending against it costs nothing. Use the `<` operator and a boolean expression."*

Why this exercise? It's *defensive coding* — you write a guard for a state that *shouldn't* happen, but if it ever does (e.g. someone messes with the page in the dev tools, or a future bug sneaks in), the UI degrades gracefully instead of cascading.

Try it before you peek.

<details>
<summary>Worked answer</summary>

```svelte
<button type="button" onclick={logHabit} disabled={habitsLoggedToday < 0}>
  Log a habit
</button>
```

Read aloud: *"a button — when clicked, run logHabit. It's disabled when today's count is somehow already negative."*

Bonus question: should we add the same `disabled={habitsLoggedToday < 0}` to *Undo* and *Reset*? Why or why not?

The answer is *no need* — the existing condition `habitsLoggedToday === 0` is *more strict* (it disables both at zero AND positive-zero), and our guards already prevent negatives. We could combine the two with `||`:

```svelte
<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday === 0 || habitsLoggedToday < 0}>
```

…but that whole expression is just *"the count is at most zero"*, which is `<=`. So:

```svelte
<button type="button" onclick={unlogHabit} disabled={habitsLoggedToday <= 0}>
```

…is the cleaner version. The same kind of cleanup pays off in real code: when two boolean conditions can be expressed as a single comparison, do it.
</details>

---

## Lesson 3.9 — Recurring concepts from earlier chapters

Chapter 3 leaned on these in new shapes:

- **`===`** (Ch 1) — used inside `disabled={habitsLoggedToday === 0}` for the same strict-equality reason.
- **Guard clauses** (Ch 2) — every function still uses them; `disabled` on the markup is the *UI-side* twin of the function-side guard. Two layers of defence.
- **`type="button"`** (Ch 1) — every button still has it.
- **The 'plural-aware' habit** (Ch 1) — your buttons now also use language carefully (*"Undo"* vs *"Reset"*).

You also added two senior habits:
- **Defensive coding** — guarding for states that *shouldn't* happen because they sometimes do.
- **Defence in depth** — the function guard *and* the UI guard, working together.

---

## Lesson 3.10 — What you can now read in the wild

After Chapter 3, you can:

- **Read `boolean`, `true`, `false`** as the type and its two values.
- **Read `&&`, `||`, `!`** and explain each.
- **Recognise short-circuit evaluation** when you see it (`x && x.foo`).
- **Spot the `||` foot-gun** in code that defaults strings or numbers.
- **Read `disabled={someExpression}`** as a reactive HTML attribute bound to a boolean.
- **Combine boolean expressions** with the right operators to model real-world UX rules.

You also have a feel for *defensive coding* — writing guards against states that "shouldn't happen" because you know they sometimes do.

---

## Glossary added in Chapter 3

| Term | Definition |
|---|---|
| `boolean` | A type whose values are `true` or `false`. |
| `true` / `false` | The two boolean literals. |
| `&&` | Logical AND; both sides must be true. |
| `\|\|` | Logical OR; either side may be true. |
| `!` | Logical NOT; flips the boolean. |
| short-circuit evaluation | Stopping early when the answer is determined. |
| `disabled` (HTML attribute) | Marks a button or input as unclickable. |
| falsy | Values JavaScript treats as `false` in boolean context (`0`, `''`, `null`, `undefined`, `NaN`, `false`). |
| truthy | Anything not falsy. |

---

## End-of-chapter checkpoint

- [ ] You added `disabled` attributes to *Undo* and *Reset* (and optionally *Log a habit*).
- [ ] You can read `&&`, `||`, `!` aloud and explain each.
- [ ] You can name three falsy values besides `false`.
- [ ] You can explain why `score || 'No score'` is broken when `score` is genuinely `0`.
- [ ] You ran the page and watched the buttons grey in and out as you clicked.

---

# End of Part I, Chapters 1–3

The rest of Part I (Chapters 4, 5, 6) lives in `chapters-04-to-06.md`. After Chapter 6 you'll have a real reactive page, an undo with a guard, disabled buttons, a name input, a day-of-week strip, and a clean empty state. Open the next file when you're ready.
