---
name: lean-surf
description: "Token-lean web browsing for agents. Use BEFORE any browser tool call (Claude in Chrome, the built-in browser, Playwright MCP, Chrome DevTools MCP) and whenever a task needs a web page: picks the cheapest way to get the data, delegates long browsing to a subagent, reads pages surgically instead of dumping them, and keeps screenshots rare and small. Also triggers on 'lean-surf', 'browse cheap', 'save tokens browsing', 'navigation économe', 'économise les tokens'. Stays on for the whole session until the user says 'surf off' or 'normal mode'. Levels: lite, full (default), ultra."
---

# Lean-surf

Browsing is the most expensive thing an agent does. Not because one call is big, but because **everything it returns stays in context and is re-sent on every later turn**. A 1,500-token screenshot taken at step 3 of a 30-step task is paid ~27 more times.

So the goal is not "fewer tool calls". It is: **the least text and pixels ever entering the main context**, without guessing or skipping a check that matters.

Active for the whole session once triggered. Reply in the user's language.

## The one test

Before every browser call, ask: **is there a cheaper call that returns only what I need?** If yes, use it.

## 1. Don't open a browser if you don't need one

Take the first rung that works:

| Need | Cheapest tool |
|---|---|
| Data with an API (GitHub, npm, PyPI, Wikipedia, RSS, a public JSON endpoint) | `gh`, `curl`, the API, never the HTML page |
| Static public page, read-only | WebFetch, `curl`, or a scraper that returns markdown (e.g. firecrawl scrape) |
| Library / framework docs | the docs tool (e.g. Context7), not the docs site |
| JS-rendered page, no interaction | browser, one navigate + one targeted extraction (section 3) |
| Clicks, forms, logged-in session | browser, with every rule below |

## 2. Delegate long browsing to a subagent

Any browsing task likely to take **more than 3 actions** goes to a subagent, ideally on a cheaper model. Screenshots, trees and page dumps then live and die in *its* context; the main context only receives the answer.

The brief must name:
- the goal and the start URL,
- the **exact output format** (e.g. "a JSON array of `{name, price, url}`, max 20 items" or "≤ 10 lines"),
- a step budget (e.g. "stop after 20 actions and report where you are"),
- what to do on a sign-in wall, CAPTCHA or payment page: stop and report.

Independent sites → one subagent each, in parallel.

Do it in the main context only when the user wants to watch step by step, or the task is 1–3 actions.

## 3. Read surgically

Use the first one that answers the question:

1. **`find "<text>"`**: up to 20 matching elements with refs. Enough to click most things.
2. **Targeted JavaScript**: return only the fields you need, capped.
   ```js
   [...document.querySelectorAll('.product')].slice(0, 20).map(e => ({
     t: e.querySelector('h2')?.innerText.trim().slice(0, 80),
     p: e.querySelector('.price')?.innerText.trim(),
     u: e.querySelector('a')?.href
   }))
   ```
   Reader mode in one line: `(document.querySelector('main, article') || document.body).innerText.slice(0, 4000)`
3. **Accessibility tree, scoped**: always pass `filter: "interactive"`, and `ref_id` / `depth` / `max_chars` when the tool has them. Never the full tree of a whole page.
4. **Page text, capped**: always pass `max_chars` (start at 3000–5000, raise only if the answer is not there).
5. **Screenshot**: last resort (section 4).

Same rules for other browser tools: Playwright `browser_snapshot` and DevTools `take_snapshot` are full trees. Prefer `evaluate` / `evaluate_script` with targeted JS. Playwright MCP also returns a snapshot after **every** action, which is another reason to delegate it to a subagent.

Network and console: always filter (`urlPattern`, `pattern`, `onlyErrors`). Never list everything.

## 4. Screenshots: rare and small

Take one only when the question is visual (layout, colors, a chart, a canvas), or the element can't be found with `find` or JS.

- `scale: 0.3–0.5` (cost drops with the square of the scale: 0.5 → ~¼, 0.4 → ~⅙).
- `zoom` on the region instead of the whole viewport.
- **Never screenshot to confirm a click.** Check the effect with text: the new URL, `find` on the expected element, or a one-line JS check (`location.href`, `document.title`, `!!document.querySelector('.success')`).
- No GIF recording unless the user asks.

## 5. Act in fewer turns

Every turn re-sends the whole context, so fewer turns = big savings.

- **URL over UI.** Build the URL instead of typing in a search box and clicking: `?q=`, `?page=`, filters, sort, `?per_page=100`. Jump to deep links directly.
- **Batch** predictable sequences in one call (navigate → click → type → Enter → extract).
- **Fill forms in one go** (`form_input` / `fill_form`) instead of click + type per field.
- **Pagination**: raise the page size via the URL, or find the JSON endpoint the page calls (filtered network list), then `fetch()` it from the page with JS. Never click "Next" 10 times.
- Load the browser tools you need **in one batch**, once.
- Don't re-read a page you already extracted. Don't re-fetch tab context unless a tab error says to.

## 6. Keep the context clean

- Large results (lists, tables, raw page text) go to a **file**; keep only a summary or count in context and read back slices when needed.
- **Stop as soon as you have the answer.** No extra "let me double-check" page loads unless the check matters (see Never cut).
- Close tabs you opened when done.

## Levels

| Level | Behavior |
|---|---|
| **lite** | Sections 1, 3, 4: right tool, surgical reads, small screenshots. No forced delegation. |
| **full** (default) | Everything above. Delegation for > 3 actions. Batch and URL-first. |
| **ultra** | full + no screenshot unless the user asks or the task is visual; delegate any browsing of > 1 action; step budget 15 per subagent; extraction output capped at ~20 items / 2,000 chars, paging through a file if more. |

Switch with "lean-surf lite|full|ultra".

## Never cut (override every level)

- **Before an irreversible action** (submit, send, pay, delete, publish): verify the page state properly, even if it costs a screenshot or a full read. Saving tokens never justifies a blind click.
- **Never guess a ref or a coordinate** to avoid a read. If `find` and JS fail, read or screenshot.
- When the user asks to **see** the page, show it.
- Security and permission rules of the browser tools stay in force: sign-in walls, CAPTCHAs, credentials and payments go to the user.
- Report what you **didn't** check or couldn't load. A cheap run never hides a gap.

## Cost cheat sheet (orders of magnitude)

| Call | Typical cost |
|---|---|
| Full-viewport screenshot | ~1,000–1,500 tokens |
| Same at `scale: 0.4` | ~200 tokens |
| Full accessibility tree / page text, uncapped | up to ~12,000 tokens |
| `find` | ~100–500 tokens |
| Targeted JS extraction | ~50–500 tokens |
| Each extra turn | re-sends everything already in context |

## Self-check before each browser call

- Could an API, `curl` or WebFetch do this without a browser?
- Should this whole task be a subagent with a fixed output format?
- Am I about to dump a tree, a page or a screenshot when `find` or 5 lines of JS would do?
- Can I fold this and the next 2 steps into one URL or one batch?
- Is this an irreversible step? Then verify properly, whatever the level.
