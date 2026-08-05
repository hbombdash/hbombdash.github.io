# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A static GitHub Pages site (`hbombdash.github.io`) — no build step, no package manager, no dependencies to install. There are only three HTML files:

- `index.html` — landing page / "start list" for the Noosa Triathlon 2026 squad, linking to each athlete's plan.
- `athletes/chaz.html`, `athletes/ed.html` — one self-contained training-plan app per athlete.

There is no dev server, linter, test suite, or build command. "Running" this project means opening the HTML files directly in a browser or serving the directory statically (e.g. `python3 -m http.server`).

## Architecture of an athlete page

Each file under `athletes/` is a single portable HTML document assembled from four distinct pieces, in this order. When editing an athlete page, know which piece you're touching:

1. **`<script type="text/plain" id="app-bundle">`** — a minified, pre-built Svelte 5 app bundle (imports Firebase JS SDK, renders the whole UI). This is compiled output pasted in verbatim — **do not hand-edit it**; treat it as opaque. It is never executed as a real `<script>` (hence `type="text/plain"`); the bootstrap loader at the bottom of the file turns it into a Blob and `import()`s it at runtime.
2. **`<style>`** — the app's compiled CSS (also generated output, supports light/dark via `[data-theme=dark]`).
3. **`<script type="application/json" id="plan-data">`** — the actual per-athlete content: a structured JSON training plan (see schema below). **This is the piece you'll edit most often** — it's plain data, safe to hand-edit, and the Svelte bundle reads it at load time.
4. **`<script type="module">` bootstrap** (end of `<body>`) — vanilla JS glue that:
   - pulls saved progress from Firestore into `localStorage` on load (with a 4s timeout, falling back to local-only),
   - monkey-patches `localStorage.setItem` so any write to the sync keys (`completed`, `changes`, `settings`, `banner-dismissed`) is debounced (600ms) and pushed back to Firestore,
   - then loads the app bundle from step 1 via a Blob URL.

All three files (`index.html` + both athlete pages) share one Firebase project (`noosa-2026`, config embedded inline — this is a public client-side Firebase key, expected to be restricted by Firestore security rules rather than kept secret). Each athlete's plan is a separate Firestore document at `plans/{meta.id}` (e.g. `plans/noosa-2026-plan`), storing just the synced keys plus a `secret` field and `updatedAt` timestamp — there's no auth; the `secret` is the only gate on writes.

## The `plan-data` JSON schema

Top-level keys: `version`, `meta`, `preferences`, `assessment`, `zones`, `phases`, `weeks`, `raceStrategy`.

- `meta` — plan id (Firestore doc key), athlete name, event, race/plan dates, `totalWeeks`.
- `preferences` — units per discipline (`swim`/`bike`/`run`), `firstDayOfWeek`.
- `assessment` — athlete's training-history/fitness inputs the plan was built from.
- `zones` — HR/pace/power training zones per discipline.
- `phases` — named macro blocks (e.g. Base/Build/Peak/Taper) with `startWeek`/`endWeek`, focus, target weekly hours, key workouts.
- `weeks` — one entry per training week (`weekNumber`, date range, `phase`, `targetHours`, `isRecoveryWeek`) containing `days`, each with a `workouts` array. Each workout has `id`, `sport`, `type`, `name`, `description`, `completed`, plus discipline-specific fields (`durationMinutes`, `distanceMeters`, `primaryZone`, `humanReadable`, etc.).
- `raceStrategy` — pacing/zone targets for race day.

When updating a training plan (new week, adjusted paces, marking structure changes rather than just ticking off workouts — completion state itself lives in Firestore/localStorage, not in this JSON), edit this JSON block directly inside the athlete's HTML file. Keep `meta.updatedAt` current and preserve the existing key structure so the Svelte bundle can parse it unchanged.

## Adding a new athlete

Both existing plans (`plan-data.meta.generatedBy: "Claude Coach"`) were produced by the **`coach-skill`** skill (installed globally at `~/.claude/skills/coach-skill`), not hand-authored. This is the intended process for all future athletes too:

1. Invoke the `coach` skill to interview the athlete (or pull their Strava history) and produce the training plan as JSON matching the `TrainingPlan` schema described above.
2. Use the skill's own render step (`npx claude-coach render plan.json --output plan.html`) to turn that JSON into the interactive HTML viewer — this is what generates the Svelte bundle/CSS layers described above, so don't hand-build those.
3. Drop the resulting HTML into `athletes/<name>.html` in this repo, then splice in this site's Firestore sync bootstrap (`<script type="module">` block, adjust the `SECRET`) since the skill's raw output won't include it — copy that block from an existing athlete file such as `athletes/chaz.html`.
4. Add a matching bib card to the `#roster` grid in `index.html` (`<a class="bib active" href="athletes/<name>.html">`), following the pattern from `4e62e99`.

Editing an athlete's `plan-data` JSON in place by hand (e.g. small pace/date tweaks) is fine for minor fixes, but a new plan or a substantial revision should go back through the coach skill rather than being hand-written to keep the assessment/zones/periodization reasoning consistent.
