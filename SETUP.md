# Thrive Flagged Notes Tracker

Single-file app (`index.html`) — oversight/compliance tool for managers and directors: shows flagged clinical notes for the staff each signed-in reviewer supervises (via the `roster` table's `manager_id`). Deployed as a static site on Render, auto-deploying from this repo's `main` branch.

## Deploy

1. New Render Static Site, connect this GitHub repo (`Thrive2026/thrive-flagged-notes-tracker`).
2. Build command: none. Publish directory: `/` (repo root — `index.html` is served directly).
3. No environment variables needed — the Supabase URL and anon key are embedded in the page, same as every other Thrive Tools hub.

## What this is

Split out of the `thrive-hub` repo (previously `tracker.html`) on 9/11/26. It doesn't share data with Data & Staff Hub — it reads its own relational `roster` table (provider_email → manager_id) plus the shared `notes` table — so it doesn't belong bundled with either that repo or Program Hub's board/funder-facing reporting tools. Its actual audience is any manager or director across any program, so it gets its own single deployment that every hub links out to.

## Cross-repo links that point here

Once this is live, update these from the old relative path (`tracker.html`) to this deployment's real URL:
- `thrive-hub`'s landing page (`index.html`) — Flagged Notes Tracker tile.
- `thrive-director`'s `clinicaldirectorhub.html` — already has a working quick-nav link (`HUB_LINKS.flaggedNotesTracker`), just needs its URL updated.
- Community Hub, Hospital Hub, Courts & Jail Hub, Crisis Center Hub, BH Linkage Hub, Director Hub — new links added 9/11/26, all pointing at a guessed URL pending confirmation (see below).

## Known guess pending confirmation

Every link above currently points to `https://thrive-flagged-notes-tracker.onrender.com` — a guess following this ecosystem's repo-name-as-subdomain convention, not a confirmed live URL. Once this is actually deployed, confirm the real URL and correct every link listed above if it's different.
