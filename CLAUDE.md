# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

No build step, no npm, no server. Open `index.html` directly in a browser.

- **Main app:** open `index.html`
- **Standalone test suite:** open `tests.html`
- **AI Rule Builder POC:** open `poc.html`

All third-party libraries are vendored as local files (`mammoth.min.js` for `.docx`, `xlsx.min.js` for `.xlsx`/`.xls`/`.csv`).

## Architecture

This is a 100% client-side browser app. No backend, no bundler, no framework.

### File roles

| File | Purpose |
|------|---------|
| `index.html` | **The real app.** All CSS + HTML + 4 inline `<script>` blocks: shared transform engine, smart rule engine, AI engine, converter/editor/test UI |
| `poc.html` | Standalone demo/documentation page with its own copy of rule parsing logic |
| `tests.html` | Standalone test suite page |
| `app.js` | Legacy file — **not loaded by `index.html`**; older standalone version kept for reference |
| `style.css` | Orphaned CSS for `app.js` — **not loaded by `index.html`**, which has all CSS inline |

`index.html` has four inline `<script>` blocks (in order):
1. **SHARED TRANSFORM ENGINE** — `window.__transform`, `window.__sanitize`, `window.__indent`
2. **SMART RULE ENGINE** — `window.__parseRules`, `window.__applyCustomRules`
3. **AI ENGINE** — `window.__aiGetKey`, `window.__aiSetKey`, `window.__aiEnabled`, `window.__aiApplyRules`
4. **CONVERTER** — file upload, rich-text editor, Smart Rule Builder UI, test suite

### Data flow

1. User uploads a file or types in the rich-text editor
2. For `.docx`: Mammoth.js converts to raw HTML → `window.__transform()` applies built-in + custom rules
3. For `.xlsx`/`.xls`/`.csv`: SheetJS converts each sheet to an HTML table → `window.__transform()` applies rules
4. For the text editor: `window.__sanitize()` normalises browser `contenteditable` noise → `window.__transform()`
5. If AI is enabled and there are unrecognised rules: `window.__aiApplyRules()` sends them to Claude → result replaces the transformed HTML
6. `window.__indent()` formats the final HTML for display
7. After conversion, the inline test suite runs checks and shows a test report button

### The shared transform engine

The SHARED TRANSFORM ENGINE script block in `index.html` exposes `window.__transform`, `window.__sanitize`, and `window.__indent`. `tests.html` also loads these. When adding or changing a rule, update the inline script block in `index.html`. (`app.js` has its own older copy — it is not used by the live app.)

### Claude AI integration

The AI ENGINE script block calls the Anthropic API directly from the browser using `fetch()` with the `anthropic-dangerous-direct-browser-calls: true` header.

- **API key** stored in `localStorage` under key `'dochtml-ai-key'` — same key is shared between `index.html` and `poc.html`
- **Model:** `claude-haiku-4-5-20251001`
- **Endpoint:** `https://api.anthropic.com/v1/messages`
- **When invoked:** only when the user has set a key AND the Smart Rule Builder contains rules that the regex parser (`window.__parseRules`) returned with `ok: false`
- **Graceful degradation:** if the API call fails, the conversion result is shown without AI rules and an error status message is displayed
- **`poc.html`** has its own equivalent `pocCallAI()` function with the same logic

The Smart Rule Builder parse list shows 🤖 (instead of ⚠) for unrecognised rules when a key is set, indicating they will be sent to Claude at convert time.

### Transform rules (applied in order in `transformHtml()`)

| Rule | What it does |
|------|-------------|
| G | Pre-parse: fix unclosed attribute quotes, stray `<`, unclosed block tags |
| 1 | `<h1>` → `<h2>` |
| 4 | `<b>` → `<strong>` |
| 5 | `<i>`/`<em>` ≤120 chars → `<strong>`; longer → unwrap |
| 6 | `<u>` → unwrap (remove underline) |
| D | Remove `<s>`, `<del>`, `<strike>` and markdown `~~text~~` entirely |
| A | Remove `<strong>` directly inside headings |
| B | Bold-first-row tables → `<thead><th scope="col">` + `<tbody>` |
| C | Merge adjacent/nested `<strong>` tags |
| E | Flip `<a><strong>` → `<strong><a>` |
| 7 | Strip MSO `<xml>` blocks and conditional comments |
| 3 | `<sup>`/`<sub>` → unicode superscript/subscript entities |
| 2 | Phone numbers → `<a href="tel:…">` (skips numbers near "fax") |
| F | `file://` hrefs → reconstructed `https://` or `href="#"` |
| 8 | Symbol entities: ®, ™, ©, curly quotes |
| 10 | `Bluecard` → `Bluecard®` (outside link text) |
| 11 | `&nbsp;` and ` ` → regular space; collapse multiple spaces |

### Test suite

The CONVERTER script block in `index.html` contains an inline test suite that runs regex/DOM checks against the converted output. Tests have three statuses: `pass`, `fail`, `fixed` (amber — auto-corrected by Rule G).

The standalone `tests.html` page loads the shared transform engine from `index.html`'s inline script via a `<script src>` tag and runs its own set of fixtures.
