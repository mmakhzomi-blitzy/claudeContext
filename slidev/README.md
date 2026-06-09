# Blitzy Dagster — Slidev pilot

Pilot conversion of the reveal.js deck (`../index.html`) to
[Slidev](https://sli.dev/) using `@slidev/theme-default` plus a
custom CSS block that mirrors the reveal.js black + cyan aesthetic.
Five representative slides to validate theme, layout, code
highlighting, and custom HTML handling before porting the full deck.

## Theme

- **Base:** `@slidev/theme-default` (official, actively maintained)
- **Overrides:** inline `<style>` block on the title slide — dark
  `#191919` background, cyan `#8be9fd` accents, same Inter / JetBrains
  Mono fonts as the reveal.js deck
- **Why this path:** official theme = long-term support; custom CSS =
  exact aesthetic match. If you want to try a different base theme
  later, change `theme:` in the frontmatter — everything else stays.

## Running

```bash
npm install
npm run dev        # opens http://localhost:3030
npm run build      # static site to dist/
npm run export-pdf # PDF export (needs playwright-chromium installed)
```

## What's in `slides.md`

Full deck, ~50 slides grouped into the same sections as `index.html`:

| Section | Slides |
|---|---|
| Title + Agenda | 2 |
| Project status (done / next) | 2 |
| Current state (fleet, chain, shared) | 3 |
| Current flow (sequence diagrams + what's missing + Dagster flow) | 4 |
| Why a workflow engine (runtime + consistency debts) | 2 |
| Modern playbook + old-vs-new + what unlocks | 5 |
| Core primitives (op, sensor, schedule, fit) | 4 |
| Built-in operations + for Blitzy | 2 |
| Job tracking &amp; visibility | 2 |
| Metrics (built-in, custom, mapping) | 3 |
| Dagster vs Airflow 3 + Airflow 3 highlights + where each wins | 4 |
| Argo Workflows (fit, vs Dagster, for Blitzy) | 3 |
| Ranked for Blitzy | 1 |
| Multi-env (verdict, what's true, qualifiers, applied) | 4 |
| Other options | 1 |
| Phased plan (overview + 7 phase slides + out of scope) | 9 |
| AI-augmented ops (diagram, menu, stack, triage hook, retrigger ×2, auto-PR, notify, risks, phases) | 10 |
| Next steps + appendix + thanks | 3 |

## Next steps

1. Run `npm install && npm run dev` and review the full deck in the browser.
2. Report any rendering issues — mermaid diagrams, code blocks, grid layouts, or custom HTML.
3. Once parity is confirmed, set up CI and retire `index.html`.

## Files

- `slides.md` — the deck
- `package.json` — Slidev + theme deps
- `.gitignore` — excludes `node_modules/`, `dist/`, generated components
