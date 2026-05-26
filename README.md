# FreediverQuizz

Multilingual freediving revision quiz — **180 multiple-choice questions** covering four progression levels, in **French, English, Spanish, and German**.

Built as a single-file PWA. No build step, no runtime dependencies (one CDN call for confetti).

→ **Live**: https://loupv.github.io/FreediverQuizz/

## Features

- **180 questions** across four recreational freediving levels (introduction → master)
- **4 languages** — FR / EN / ES / DE toggle, instant in-quiz switch (questions, answers, explanations, topic refs all translated). Auto-detects from browser locale on first visit, falls back to EN
- **Level selector** at first launch: pick your exam target → quiz focuses **exclusively** on that level's questions (you can still multi-select levels manually via filters)
- **Type filters** — Safety, Physiology, Technique, Equipment, Theory, Medical, Training, Environment
- **Three modes** — Random 10-question quiz, full review with save/resume, or by chapter
- **Flashcard mode** — pressure-free learning with reveal-on-tap. Negative MCQs ("Which is NOT…") show the 4 options before reveal so the question makes sense
- **Multi-correct questions** — some questions accept several valid answers (select-all-that-apply format with checkboxes and a Validate button)
- **Pedagogical** — every answer has a paragraph explanation explaining *why* the right answer wins and why distractors fail, plus a topic reference
- **Acronym tooltips** — hover or tap any freediving acronym (BO, LMC, MDR, RV, etc.) for an inline definition
- **Glossary view** — searchable list of all freediving acronyms and terms
- **Spaced repetition lite** — boost or reduce the frequency of any question with a button after answering; preferences persist in localStorage
- **Question reporting** — one-tap "report this question" with batched email delivery via Web3Forms. Reports include the FR source alongside the translation for translation-issue triage
- **Question IDs** — every question carries a short hash visible in the corner, used in reports and useful for cross-referencing
- **Lively but subtle UI** — gentle fade animations, confetti on correct answers, haptic feedback on mobile, smooth view transitions
- **Works offline** — install as PWA on iOS/Android (Share → Add to Home Screen)
- **Privacy-friendly analytics** — Google Analytics 4 in cookieless mode, IP anonymized, no consent banner needed

## Question structure

Each question carries content for all 4 languages:

```js
{
  level: 1,           // 1, 2, 3, or 4
  ch: 1,              // chapter number within level
  type: "safety",     // safety | physio | technique | equipment | theory | medical | training | env
  fr: { q, a, correct, exp, ref },  // French
  en: { q, a, correct, exp, ref },  // English
  es: { q, a, correct, exp, ref },  // Spanish
  de: { q, a, correct, exp, ref },  // German
}
```

- `q`: question text (may contain `<abbr>` and `<strong class='neg'>` markup)
- `a`: 4 answer strings (or 2 for True/False)
- `correct`: index of the correct answer — an integer for single-answer MCQ (usually 0), or an array of indices for multi-correct questions
- `exp`: paragraph explanation
- `ref`: topic name (used as a short reference label)

## Run locally

Just open `index.html` in any modern browser. No build, no install.

To regenerate translations after editing source-language questions, use the merge helper:

```bash
node merge-translations.js
```

The helper reads `trans-{es,de}-{1,2,3,4}.json` sidecars and splices the `es:`/`de:` blocks into `index.html`. Both the script and the sidecars are gitignored.

## Question report mechanism

When users tap "Report this question", the report is queued in localStorage and flushed to your email (via Web3Forms) either at the end of the quiz or after 10 minutes — whichever comes first. To configure your own destination:

1. Create a free access key at https://web3forms.com (just enter your email)
2. Replace the `WEB3FORMS_KEY` constant in `index.html`

Reports include question ID, level, chapter, type, language used by the reporter, the question and correct answer in that language, **the French source for comparison**, the user's chosen answer (if any), and a timestamp.

## Deployment

Static site hosted via GitHub Pages on `main`. A custom workflow at `.github/workflows/pages.yml` uploads the repo root and deploys without any Jekyll step — push to `main` and the new version is live in ~15 seconds.

## License

The app code is MIT. The pedagogical content is paraphrased general freediving knowledge — no verbatim reproduction of any specific course material.
