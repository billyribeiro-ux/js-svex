# The Streak Book — The May 5, 2026 Bible

*From "I've never opened a terminal" to principal-engineer-level-7 specialisation in Svelte 5 + SvelteKit + TypeScript + Postgres + Stripe + Cloudflare R2 + Vercel.*

---

## How to use this folder

The book is split across chapter files for readability. Read in order.

| File | Chapters | Part |
|---|---|---|
| [chapters-01-to-03.md](chapters-01-to-03.md) | 1, 2, 3 | Part I — First words (start) |
| [chapters-04-to-06.md](chapters-04-to-06.md) | 4, 5, 6 | Part I — First words (end) |
| [chapters-07-to-12.md](chapters-07-to-12.md) | 7–12 | Part II — Things, not just numbers |
| [chapters-13-to-22.md](chapters-13-to-22.md) | 13–22 | Part III — Components, props, snippets |
| [chapters-23-to-30.md](chapters-23-to-30.md) | 23–30 | Part IV — TypeScript as a tool |
| [chapters-31-to-37.md](chapters-31-to-37.md) | 31–37 | Part V — SvelteKit routing |
| [chapters-38-to-43.md](chapters-38-to-43.md) | 38–43 | Part VI — Data that survives a refresh |
| [chapters-44-to-48.md](chapters-44-to-48.md) | 44–48 | Part VII — Real users |
| [chapters-49-to-55.md](chapters-49-to-55.md) | 49–55 | Part VIII — Money, polish, full Svelte catalogue |
| [chapters-56-to-67-and-appendices.md](chapters-56-to-67-and-appendices.md) | 56–67 + A/B/C | Part IX — Shipping, operating, graduation, appendices |

Plan file (the curriculum's structure, the 21 Bible rules, the level-7 outcomes checklist): [`/Users/billyribeiro/.claude/plans/claude-i-want-you-zany-rain.md`](../claude-i-want-you-zany-rain.md).

---

## What's in here

- **67 chapters** taking the reader from a terminal to deployment, with a real growing project (Streak — a habit tracker) every step of the way.
- **3 appendices** — Remote Functions (experimental); reading list; Bible card.
- **Engineer-English read-aloud tables** on every snippet.
- **Read-this-code reading exercises** in every chapter.
- **Say-it-in-English-first exercises** at the end of every chapter.
- **Build-break-fix drills** at every Part boundary.
- **Runtime-evidence rules** applied throughout.

---

## The growth arc

| End of Chapter | What Streak can do |
|---|---|
| 1 | A reactive page counts today's habits |
| 6 | Counter, undo, day-strip, empty state |
| 12 | Typed list of named habits, searchable |
| 22 | Componentised with the loading/empty/error trio |
| 30 | TS-strict throughout; `Result`, `HabitStore`, `Money` module |
| 37 | Multi-page app; routing fluency |
| 43 | Real Postgres, atomic mutations, optimistic UI |
| 48 | Real auth, RBAC, audit, security review |
| 55 | Stripe Pro tier, every Svelte primitive used, Lighthouse 100s |
| 64 | CI/CD, deployed, observable, public REST + emails + cron |
| 65 | 600-line code review + ADR + scaling plan |
| 66 | Reader ships Streak Freezes from a one-paragraph brief |
| 67 | Reader writes the post-mortem of their own feature |

---

## The 21 non-negotiables (the Bible rules)

In short:

1. `pnpm` only.
2. `.svelte` and `.ts` only.
3. TS strict from line one — no `any`, `!`, `@ts-ignore`.
4. Svelte 5 runes only — no `export let`, `$:`, `on:click`, `<slot>`.
5. Principal-engineer idioms from line one (`+=`, `===`, `const`, `??`, `?.`, early returns, `type="button"`).
6. Engineer-English narration on every snippet.
7. Every example is load-bearing.
8. English sentence first, code second.
9. No comment lies.
10. Money is integer cents end-to-end.
11. Atomic UPDATE-RETURNING; never SELECT-then-UPDATE.
12. Idempotency on every external-side-effect handler.
13. `.timeout()` and `.connect_timeout()` on every external client.
14. Bounded label sets on every metric.
15. No password/token/secret/PII logged.
16. No `<img>` without width/height.
17. No silent `.catch(() => {})`.
18. Migrations forward-only.
19. Server-only secrets via `$env/static/private`.
20. Every chapter delivers a visible win.
21. Compilation + tests are necessary, not sufficient — demand runtime evidence.

The full text lives in the plan file.

---

## Status

**Complete.** All 67 chapters and 3 appendices written. Date-stamped May 5, 2026.

The book is the bible of the stack as of today.
