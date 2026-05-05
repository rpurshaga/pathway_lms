# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of **standalone, self-contained HTML widgets** that are embedded as iframes inside TalentLMS units (the LMS at `courses.pathway.training`). Each `.html` file at the root is its own independent app — there is **no build step, no bundler, no package manager, and no shared CSS/JS files**. CSS and JS are inlined in each file.

Deployed via **GitHub Pages on the `main` branch** at `https://rpurshaga.github.io/pathway_lms/`. Pushing to `main` is the deploy. There is no staging environment.

## Workflow

```bash
# Local preview (the only "dev server")
python3 -m http.server 3333
# Then open http://localhost:3333/<file>.html

# Deploy = git push to main
git add <file>.html && git commit -m "..." && git push
```

There are no tests, linters, or builds. The launch config at `~/.claude/launch.json` defines the `python3 -m http.server 3333` runner.

## File categories

- **`capstone-assistant.html`** — the major active project; see architecture below.
- **`nextgen-philosophy-wizard.html`** — multi-step wizard with `docx`/`FileSaver` exports.
- **`ng-video-player.html` / `ng-audio-player.html` / `video-player.html` / `text-reader.html`** — Win98-skinned media players for LMS content units. The dashboard at `~/pathway_dashboard` generates content that gets played here.
- **`coach.html`, `celebrate.html`, `congratulations/`, `pathway_completion_final.html`** — completion / coach overlay screens.
- **`ministry-timeline.html`** — read-only Playfair/parchment-styled landing page (different aesthetic from the rest).
- **`tts/tts_<id>.mp3`** — pre-generated TTS audio referenced by content units; committed in batches by the dashboard's pipeline.

## Repo-wide design conventions

- **Win98 aesthetic** (almost everything except `ministry-timeline.html`): silver `#c0c0c0`, navy title bars `#000080`, Tahoma + VT323 (Google Fonts), inset/outset 3D borders.
- **`background: transparent`** on `html`/`body` is required so the widget blends with the LMS chrome — see commit `1d43f79`. Don't undo this.
- **Mobile responsive** with separate `.mobile-topnav` / `.sidebar-drawer` patterns; iframe height is fluid (`min-height:720px` is the typical embed).
- **No external runtime deps** beyond Google Fonts and a few CDN scripts (`docx`, `FileSaver` in the wizard). No npm packages, no frameworks.

## `capstone-assistant.html` architecture

This is the most complex file (~170KB). Future work usually lands here.

### Single source of truth: the `COURSES` object

`const COURSES = { 1: {...}, 2: {...}, 3: {...} }` — defines all 3 courses, their docs, and ~50 sections. Each section has:
- `id` (used as URL param + localStorage key)
- `title`, `trigger`, `desc`, `ph` (placeholder), `ai` (template-literal AI prompt), `guide`, `wt` (word target)
- Optional flags: `noAI:true`, `interviewReq:true`, `spreadsheetNote:true`

Doc IDs are stable: `d1`–`d8`, `dx`/`dx2`/`dx3` (reflection/disclosure). **Section IDs must remain stable** because they're hardcoded into LMS embed URLs (see `embed-codes.md`).

### Three render modes (driven by URL params)

```
?                          → renderSelector()  (3-card course picker)
?course=N                  → renderMain()      (full app for course N)
?course=N&view=compile     → renderMain() + setMode('download')
?course=N&secs=id1,id2,... → renderMini()      (per-section embed widget)
```

The `renderMini()` mode is what TalentLMS units use — one section per LMS unit. See `embed-codes.md` for the full embed URL list.

### Persistence

`localStorage` keyed per course: `pathway_c1_answers`, `pathway_c2_answers`, `pathway_c3_answers`. Auto-saves on textarea input. Each course's answers are independent; a student can work all three in parallel without bleed.

### Cross-doc references

Several sections reference other docs by number (e.g., missions_cal references "Doc 4" for sending content; leadership_pipe references "Doc 7" for volunteer system). **When restructuring docs, search for these textual references** in `desc`/`ai`/`guide` strings — they don't auto-update.

### Stale-reference hot spots

- `renderDownload`'s format-hint string (around the `Check format` `<li>`) hardcodes per-doc page counts.
- The "spreadsheet note" banner hardcodes which doc number contains the `.xlsx`.
- `interview_notes`, `vision_yr`, and the AI disclosures all reference specific doc numbers in their placeholders.

After any doc renumbering, grep for `Doc \d` outside the COURSES object and update.

## Companion services (not in this repo)

- **`~/pathway_dashboard/cloudflare-worker/`** — Cloudflare Worker `pathway-content-summaries` (deployed at `pathway-content-summaries.director-ec4.workers.dev`) that serves AI-generated content summaries from KV namespace `SUMMARIES` (id `e84360b45baf4abc963e4f45a20b344d`). The capstone-assistant doesn't currently call it, but the players in this repo may.
- **`~/pathway_dashboard/`** — the admin tool that pushes content/TTS into TalentLMS and generates the audio files committed under `tts/`.

## Commit conventions

From `git log`:
- TTS additions: `chore: add TTS audio for pipeline item <N>`
- Feature work on a widget: `<Widget>: <change>` or descriptive imperatives (e.g., `Add per-section mini embed mode (?course=X&secs=id1,id2)`)
- Compatibility fixes for the LMS: explicit (e.g., `Fix mobile & TalentLMS app compatibility for capstone assistant`)
