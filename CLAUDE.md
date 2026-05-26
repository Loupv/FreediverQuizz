# CLAUDE.md — FreediverQuizz

Notes for future Claude Code sessions working on this repo.

## What this is

A single-file PWA (`index.html`, ~635 KB) that runs a multilingual multiple-choice quiz for freediving certifications. Live at https://loupv.github.io/FreediverQuizz/.

- No build step, no bundler, no runtime deps (beyond one CDN call for confetti)
- HTML + CSS + inline JS, all in `index.html`
- Hosted via GitHub Pages on `main` branch — push to deploy via the custom workflow at `.github/workflows/pages.yml` (bypasses Jekyll)
- 180 questions, 4 languages (FR/EN/ES/DE), 4 progression levels

## File map

```
index.html              ← the entire app (markup, styles, data, logic)
README.md               ← user-facing docs
CLAUDE.md               ← this file
robots.txt              ← AI crawler allowlist (GPTBot, ClaudeBot, etc.)
sitemap.xml             ← hreflang map (currently lists only the FR/EN root)
og-image.{svg,png}      ← OpenGraph card (1200×630)
.gitignore              ← excludes *.pdf, trans-*.json, merge-translations.js
*.pdf                   ← source manuals if any (NOT committed, third-party copyright)
trans-*.json            ← translation chunk artefacts (gitignored)
merge-translations.js   ← merge helper (gitignored)
.github/workflows/      ← custom Pages deploy workflow (no Jekyll)
```

## Architecture inside `index.html`

The file is organised top-to-bottom:

1. **`<head>`** — meta tags, SEO, JSON-LD (`WebApplication` + `FAQPage`), OpenGraph, GA4 cookieless setup
2. **`<style>`** — all CSS, ~700 lines, BEM-ish naming
3. **`<body>`** — minimal markup: `<header>` with logo + 4-button language toggle, `<main id="main">`, footer with coffee button and analytics privacy note
4. **`<script>`** (the main one) — everything in order:
   - `LEVEL_INFO` (lines ~934): per-level metadata (name, prereqs, targets, desc) × 4 languages
   - `TYPES` (~954): question-type chips × 4 langs
   - `CHAPTERS` (~960): chapter names per level × 4 langs
   - `I18N` (~976): all UI strings × 4 langs (~70 keys)
   - `QUESTIONS` (~1274): the data array, 176 entries
   - `ACRONYMS` (~1925): freediving abbreviation glossary × 4 langs (~23 entries)
   - **State** (`let state = { lang, view, examLevel, selectedLevels, … }`)
   - **Routing / render** (`render()`, `go()`, view renderers)
   - **Quiz engine** (`startQuiz`, `answerQuestion`, `nextQuestion`, weighted random sampling)
   - **Flashcards** (`renderFlashcards`, `revealFlashcard`, `nextFlashcard`)
   - **Glossary** (`renderGlossary`)
   - **Report system** (Web3Forms, see below)
   - **`GLOSSARY_GROUPS`** + **init**

## Question data structure

```js
{
  level: 1,        // 1 | 2 | 3 | 4
  ch: 1,           // chapter index within the level
  type: "safety",  // safety|physio|technique|equipment|theory|medical|training|env
  fr: { q: "...", a: [4 strings or 2 for T/F], correct: 0, exp: "...", ref: "topic" },
  en: { ... },
  es: { ... },
  de: { ... }
}
```

Notes:
- `correct` is the index of the right answer. At quiz time the renderer shuffles options and stores the shuffled order in `q._indices` so live language switching can re-map answers.
- HTML inside `q` is allowed: `<strong class='neg'>NOT</strong>` marks a negative MCQ, `<abbr title="…">` produces tooltips. The `annotateAcronyms()` function auto-wraps known acronyms in `<abbr>` tags via the `ACRONYMS` dictionary.
- `q._orig` is set at startQuiz time and points back to the source question object (used by language switching and report system).

## Language switching

`setLang(l)` updates `state.lang`, persists to localStorage, swaps the active button class, then walks `state.quiz` to re-extract `q`/`a`/`exp`/`ref` from `q._orig[l]`, preserving the shuffle order via `q._indices`. The current question stays on screen but text re-renders in the new language.

UI strings are accessed via `t(key)` which reads `I18N[state.lang][key]`.

## Persistence (localStorage)

All keys use the `fq_` prefix (FreediverQuizz).

- `fq_lang` — selected language (`fr`/`en`/`es`/`de`)
- `fq_exam_level` — `1`/`2`/`3`/`4`
- `fq_levels` — JSON array of selected levels (default = `[examLevel]`, exclusive)
- `fq_types` — JSON array of selected types
- `fq_weights` — question-id → weight map for spaced-repetition lite
- `fq_quiz_save` — in-progress quiz state (`qIndex`, `score`, `weakChapters`, etc.) for save/resume
- `fq_reports_queue`, `fq_reports_deadline` — pending report batch (see below)

## Question reporting

Tap "⚐ Report" → silent enqueue + toast. Batch is flushed when:
- The result screen renders (immediate)
- 10 minutes pass since the first report in the batch (timer + persisted deadline so it survives reloads/closes)

Endpoint: Web3Forms (https://api.web3forms.com/submit). Access key is the constant `WEB3FORMS_KEY` near the top of the report block. The destination email is whatever you configured at web3forms.com.

Each entry includes language, level/chapter/type, the user-facing question + correct answer in the reporter's language, **plus the FR source `qFr`/`correctFr` for translation-issue triage**, the user's chosen answer if any, and a timestamp. The email subject tags the batch's languages (e.g. `[ES/FR]`).

## Translation workflow

If you edit FR or EN content, the ES/DE translations need to be regenerated for that question. Two paths:

**Light edit** (1-2 questions): edit each `es:` and `de:` block manually in `index.html` to stay in sync.

**Heavy edit** (many questions): use the merge pipeline:
1. Recreate `trans-{es,de}-{1,2,3,4}.json` sidecars (these are gitignored and may not exist in a fresh checkout — translate as needed via an LLM, keeping the index field aligned with question order 1–180)
2. Run `node merge-translations.js` — splices `es:`/`de:` blocks into each question by matching the `en: {…} }` close pattern
3. Validate with the inline `node -e "..."` script in the README, or just open the page

The merge script lives in the gitignore-d `merge-translations.js` — recreate it if needed (it's documented in README under "Run locally").

## TCC / sandbox gotchas (macOS Claude Code)

This repo is in `~/Documents`, which is TCC-protected on macOS. We've hit this before:

- The main Claude Code process's Read/Edit/Write tools work fine
- Bash subprocesses (`cat`, `cp`, `node fs.readFileSync`) sometimes get `EPERM: operation not permitted` — TCC kicks in for processes that don't have Documents access
- **Sub-agents launched via the Agent tool are particularly affected** — they often can't read the project files or write to `/tmp/`. Two workarounds:
  1. Have the sub-agent print large outputs (e.g. translated JSON) directly in its result message instead of writing files; the parent then saves them via Write tool
  2. Move the repo out of `~/Documents` (e.g. to `~/code/AIDATest`) — eliminates TCC friction entirely

If you see "Operation not permitted" on a known-good path, quit and reopen Claude Code (it usually re-asks TCC permission), or grant Documents access in System Settings → Privacy & Security → Files and Folders → Claude Code.

## Commit conventions

Look at `git log --oneline` — the project uses short imperative subject lines plus a body paragraph explaining motivation. No conventional-commits prefixes. The codebase is small so commits are usually scoped to a feature area ("Add Spanish and German translations", "Fix flashcard rendering for negative MCQ questions"). Multi-feature commits are OK if they cluster logically.

## What to avoid

- **Don't commit PDFs** — `.gitignore` excludes `*.pdf` but double-check (third-party copyright)
- **Don't reproduce any source material verbatim** — paraphrase. The original audit specifically flagged questions that were too close to source; we rewrote them. The app is intentionally generic and doesn't claim affiliation with any specific certification body
- **Don't add lower-level questions to a higher-level quiz** — the filter is intentionally exclusive (Level 3 = only L3 questions; the user explicitly requested this)
- **Don't break the FR/EN/ES/DE quartet** — every question must have all 4 languages. The validation `arr.every(q => q.fr && q.en && q.es && q.de)` should always pass

## Common operations

```bash
# Count questions
grep -cE '^  \{ level:' index.html
# Expected: 176

# Validate JS parse and structure
node -e "
const fs=require('fs');
const s=fs.readFileSync('index.html','utf8');
const m=s.match(/const QUESTIONS = \[([\s\S]*?)\n\];/);
const arr=eval('(['+m[1]+'])');
console.log('count:',arr.length);
console.log('all 4 langs:',arr.every(q=>q.fr&&q.en&&q.es&&q.de));
console.log('answer length consistency:',arr.every(q=>q.fr.a.length===q.en.a.length&&q.en.a.length===q.es.a.length&&q.es.a.length===q.de.a.length));
"

# Find a question by FR text fragment
grep -n "fr: { q: \"[^\"]*<some-keyword>" index.html

# List all unique acronyms used in questions (sanity check ACRONYMS coverage)
grep -oE 'class=.neg.>[A-Z]+</strong>' index.html | sort -u
```

## Recent history (May 2026)

The codebase went through several big passes that future sessions should be aware of:

1. **Translation rollout (FR → +EN → +ES/DE)**: started bilingual, now 4 languages
2. **Quality pass (×2)**: removed 15 duplicate/trivia questions, rewrote ~30 distractors that were telegraphing the answer by length or absurdity
3. **Generic level naming**: user-facing UI says "Level 1/2/3/4" rather than naming any specific certification body. Data structure uses `level: 1` etc.
4. **Per-level exclusive filtering**: was inclusive (L3 = L1+L2+L3), now exclusive
5. **Batched reporting**: replaced mailto: with Web3Forms batched delivery
6. **Beginner UX audit**: identified ~20 questions where distractors were too short or terms were forward-referenced — all fixed
