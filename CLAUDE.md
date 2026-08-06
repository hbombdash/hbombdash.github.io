# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A static GitHub Pages site (`hbombdash.github.io`) — no build step, no package manager, no dependencies to install. There are only four HTML files:

- `index.html` — landing page / "start list" for the Noosa Triathlon 2026 squad, linking to each athlete's plan.
- `athletes/chaz.html`, `athletes/ed.html`, `athletes/gubbo.html` — one self-contained training-plan app per athlete.

There is no dev server, linter, test suite, or build command. "Running" this project means opening the HTML files directly in a browser or serving the directory statically (e.g. `python3 -m http.server`).

## Architecture of an athlete page

Each file under `athletes/` is a single portable HTML document assembled from six distinct pieces — pieces 1–3 come from the generated viewer, pieces 4–6 are added by this site. When editing an athlete page, know which piece you're touching:

1. **`<script type="text/plain" id="app-bundle">`** — a minified, pre-built Svelte 5 app bundle (imports Firebase JS SDK, renders the whole UI). This is compiled output pasted in verbatim — **do not hand-edit it**; treat it as opaque. It is never executed as a real `<script>` (hence `type="text/plain"`); the bootstrap loader at the bottom of the file turns it into a Blob and `import()`s it at runtime.
2. **`<style>`** — the app's compiled CSS (also generated output, supports light/dark via `[data-theme=dark]`).
3. **`<script type="application/json" id="plan-data">`** — the actual per-athlete content: a structured JSON training plan (see schema below). **This is the piece you'll edit most often** — it's plain data, safe to hand-edit, and the Svelte bundle reads it at load time.
4. **`<script type="module">` bootstrap** (end of `<body>`) — vanilla JS glue that:
   - pulls saved progress from Firestore into `localStorage` on load (with a 4s timeout, falling back to local-only),
   - monkey-patches `localStorage.setItem` so any write to the sync keys (`completed`, `changes`, `settings`, `banner-dismissed`) is debounced (600ms) and pushed back to Firestore,
   - then loads the app bundle from step 1 via a Blob URL,
   - and finally adds a tap-anywhere-outside backdrop that closes the mobile sidebar via the app's own toggle button.

Two smaller site-specific pieces sit alongside the bootstrap. Like it, they're deliberately kept outside the bundle so generated output never has to be touched:

5. **`<style id="mobile-sidebar-fix">`** (before `</head>`) — mobile-only z-index/backdrop CSS that partners with the backdrop script in step 4.
6. **Back-to-start-list link + `<style id="back-to-list-style">`** (right after `<div id="app"></div>`) — floating pill linking to `../index.html`, reusing the app's theme CSS vars so it tracks light/dark.

All four files (`index.html` + the three athlete pages) share one Firebase project (`noosa-2026`, config embedded inline — this is a public client-side Firebase key, expected to be restricted by Firestore security rules rather than kept secret). Each athlete's plan is a separate Firestore document at `plans/{meta.id}` (e.g. `plans/noosa-2026-plan`), storing just the synced keys plus a `secret` field and `updatedAt` timestamp.

**On the `secret` — read before "fixing" it.** There is no auth. Reads are open to anyone (`allow read: if true`). Each page embeds a per-athlete `secret` that its writes include, and the Firestore rule requires it to match the stored one. This is a speed bump, not access control: the secret is visible in page source and in this public repo, and because `request.resource.data` on an update reflects the *post-merge* document, a merge write that omits `secret` inherits the stored value and passes the check. This posture was reviewed and deliberately accepted (Aug 2026) — the data is workout ticks with no personal or financial exposure. Don't re-raise it as a bug; meaningful hardening would mean Firebase App Check or real auth, which is a product decision, not a cleanup.

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

All existing plans (`plan-data.meta.generatedBy: "Claude Coach"`) were produced by the **`coach-skill`** skill (installed globally at `~/.claude/skills/coach-skill`), not hand-authored. This is the intended process for all future athletes too:

1. Invoke the `coach` skill to interview the athlete (or pull their Strava history) and produce the training plan as JSON matching the `TrainingPlan` schema described above.
2. Use the skill's own render step (`npx claude-coach render plan.json --output plan.html`) to turn that JSON into the interactive HTML viewer — this is what generates the Svelte bundle/CSS layers described above, so don't hand-build those.
3. Drop the resulting HTML into `athletes/<name>.html`, then add this site's four additions, none of which the skill's raw output includes:
   - rename the bundle's opening tag `<script type="module" crossorigin>` → `<script type="text/plain" id="app-bundle">` so the bootstrap controls when it runs;
   - `<style id="mobile-sidebar-fix">` before `</head>`;
   - the back-to-start-list link + `<style id="back-to-list-style">` after `<div id="app"></div>`;
   - the `<script type="module">` sync bootstrap before `</body>`, with a fresh `const SECRET` (`openssl rand -hex 24`).

   Copy all of these **verbatim** from an existing athlete page (`athletes/ed.html`) rather than retyping them, changing only the secret. Then verify: strip out `plan-data` and the bundle from old and new, normalise the secret, and confirm the remaining chrome is byte-identical across all athlete pages.
4. Add a matching bib card to the `#roster` grid in `index.html` (`<a class="bib active" href="athletes/<name>.html">`), following the pattern from `4e62e99`. Goal time comes from the plan's `raceStrategy.targetTime`; the hero's athlete counter derives itself from the DOM, so it needs no edit.

Two things that have tripped this up before: the raw coach-skill HTML may not be where it's said to be (check `~/Desktop/Life Admin/101_Claude /Training Coach/htmls/` as well as `~/Downloads`), and the `.json` that usually accompanies it is redundant — the same plan data is already embedded in the HTML, and the site never reads the `.json`.

Editing an athlete's `plan-data` JSON in place by hand (e.g. small pace/date tweaks) is fine for minor fixes, but a new plan or a substantial revision should go back through the coach skill rather than being hand-written to keep the assessment/zones/periodization reasoning consistent.
