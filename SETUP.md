# Thrive Flagged Notes Tracker

Single-file app (`index.html`) — oversight/compliance tool for managers and directors: shows flagged clinical notes for the staff each signed-in reviewer supervises (via the `roster` table's `manager_id`). Deployed as a static site on Render, auto-deploying from this repo's `main` branch.

## Deploy

1. New Render Static Site, connect this GitHub repo (`Thrive2026/thrive-flagged-notes-tracker`).
2. Build command: none. Publish directory: `/` (repo root — `index.html` is served directly).
3. No environment variables needed — the Supabase URL and anon key are embedded in the page, same as every other Thrive Tools hub.

## What this is

Split out of the `thrive-hub` repo (previously `tracker.html`) on 9/11/26. It doesn't share data with Data & Staff Hub — it reads its own relational `roster` table (provider_email → manager_id) plus the shared `dap_note_analytics` table — so it doesn't belong bundled with either that repo or Program Hub's board/funder-facing reporting tools. Its actual audience is any manager or director across any program, so it gets its own single deployment that every hub links out to.

## Live URL

`https://thrive-flagged-notes-tracker.onrender.com`, confirmed live 9/11/26. These link here:
- the `thrive-hub` landing page
- Community, Hospital, Courts & Jail, Crisis Center and BH Linkage hubs
- Director Hub
- Clinical Director Hub (`HUB_LINKS.flaggedNotesTracker`)

If the URL ever changes, update every one of those.

## Flag definitions (changed 10/29/26, from Sarah Neff's feedback)

The Provider Scorecard is now the **Provider Coaching** scorecard (Leadership Dashboard → last 90 days). A "flag" means a documentation pattern worth a coaching conversation, not that a PRS reported something:

- **Documentation grade**: Needs Improvement / Non-Compliant from the UM rubric (Grade Next Batch).
- **Repetitive language**: 50%+ of 3-word phrases match another note for the same client within 90 days (whole note, or Assessment + Plan alone). Standard notes only.
- **Possible copy-paste**: the same match against a different client's note.
- **Safety concern not self-reported**: `ai_concern_flagged` without `concern_flagged`.

Self-reported concerns and ITP flags no longer count toward Flag Rate. They're shown in their own column.

All thresholds live in the `COACHING` and `SIM_*` constants in `index.html`, and the page's "How flags work" explainer is generated from them, so the two stay in sync.

**Manager column**: for `org_wide` reviewers this reads the full `roster` table plus `reviewers.id, name`. If RLS blocks either read, the column shows "No manager on roster" and nothing else breaks. Check that `org_wide` reviewers have SELECT on both tables.

**Rubric change**: new grades tag each issue with its standard (`[Goal Link] …`) so Coaching Focus can group them exactly. Older untagged grades are grouped by keyword, which is approximate.

## Triage workflow (changed 10/29/26)

This is a utilization management and oversight tool. Nobody reads every note. Every note gets scanned, and managers only work the notes that come back flagged.

1. **Scan.** The safety scan (`ai_concern_flagged`) runs when the note is submitted. When any reviewer opens the tracker, it grades the ungraded notes in their scope in the background against the UM rubric, newest first, up to 60 per visit (`AUTO_SCAN` in `index.html`). It also checks for repetitive or copied language.
2. **Two views.** **Team Notes** (the default) shows every note from the providers on the reviewer's roster, flagged or not, 100 at a time, filtered by flag type, provider and date window. Org-wide reviewers see every division. **Flagged Notes** shows only flagged notes, with the review queues: Needs Review (sorted by urgency: risk the coach didn't report → reported concern → copy-paste across clients → Non-Compliant → repetitive → Needs Improvement → ITP update), Sent Back to Coach, Director, Clinical Director, Utilization Review, and All Flagged.
3. **Prompt.** Each flagged card has a **What to review** box that says what to read and what the likely next step is.
4. **Act.** The options are Approve, **Send Back to Coach**, **Escalate to Director**, **Escalate to Clinical Director**, or **Refer to Utilization Review**. Each has its own queue tab and open count.

Send Back to Coach still writes `review_outcome = 'rejected'` so the other hubs keep working. Only the labels changed.

## Required SQL (run once in Supabase)

```sql
-- Utilization Review queue
alter table dap_note_analytics
  add column if not exists ur_referred boolean default false,
  add column if not exists ur_referred_by text,
  add column if not exists ur_referred_date timestamptz,
  add column if not exists ur_reason text,
  add column if not exists ur_resolved boolean default false,
  add column if not exists ur_resolved_by text,
  add column if not exists ur_resolved_date timestamptz,
  add column if not exists ur_outcome text;

-- Optional: route "Escalate to Director" to the director instead of the provider's manager
alter table roster add column if not exists director_id uuid references reviewers(id);
```

If you don't run the UR SQL, everything else still works. The Refer to Utilization Review button shows a "run the SQL in SETUP.md" message instead of failing silently. If `roster.director_id` is empty, Escalate to Director goes to the provider's manager, which is how it worked before.

## How grading runs (Edge Function `grade-notes`)

Grading happens in the Supabase Edge Function `grade-notes` (source: `edge/grade-notes/index.ts`). It holds the Anthropic API key and the UM rubric. It runs three ways:

1. **On a schedule (pg_cron):** does the bulk of the grading, whether or not anyone has the tracker open.
2. **Auto-scan:** when a reviewer opens the tracker, it grades up to 60 ungraded notes in that reviewer's scope, newest first (`AUTO_SCAN`). This catches up recent notes between scheduled runs.
3. **Manual backfill:** org-wide reviewers can click **Grade oldest 25 now**.

Once the scheduled job reliably keeps up, you can set `AUTO_SCAN.enabled = false` to cut API cost.

**Compliance checks (Privacy Officer, 42 CFR Part 2):**
- **The note text goes to Anthropic.** Confirm the API key in the function's secrets belongs to the BAA-covered Anthropic account, the same one the Thrive-Tools note generators use.
- **The function has to check who's calling it.** Supabase accepts the public anon key as a valid token, so `verify_jwt` on its own doesn't block outsiders. The function should look up the caller's session in `reviewers` before grading anything. Otherwise anyone with the page source can run up API cost.

## Who sees what (changed 10/29/26)

The `roster` table decides it. There's no extra setting.

| Reviewer | Rule | Sees |
|---|---|---|
| Manager | `roster.manager_id` = them | Their own team in **Team Notes** and **Flagged Notes**. |
| Director | `roster.director_id` = them | Every team under them in **All Team Notes** and **Flagged Notes**, with a Manager filter, plus an **Escalated to Me** tab: notes a manager escalated to them, open first, with a "Show handled" toggle. |
| Org-wide (`reviewers.role = 'org_wide'`) | Everything | Everything above. Escalated to Me also has **Clinical Director** and **Utilization Review** queues. |

"Escalated to Me" matches a note when `director_escalated_to` is the director's name (set when the note is escalated) **or** the roster lists them as the provider's director.

### Supabase setup for directors

```sql
alter table roster add column if not exists director_id uuid references reviewers(id);

-- Directors need to read their division's roster rows (managers already can read their own).
create policy "directors read their division roster" on roster
  for select using (director_id = auth.uid());
```

**Check this before telling directors it's live:** if `dap_note_analytics` has an RLS policy that only lets a reviewer read notes for providers where they're the *manager*, directors will see an empty page. That policy needs to allow `director_id = auth.uid()` too. I can't see your current policies from here.

### Loading the roster and access rules

Run `roster_import.sql` once in the Supabase SQL Editor. It's safe to re-run. It:
1. Lowercases every `provider_email` on notes and the roster, and adds a trigger that keeps new ones lowercase. Before this, the same person showed up under 2–4 spellings.
2. Adds `roster.director_id`.
3. Adds access rules *alongside* the existing ones, so nothing is taken away: reviewers can read each other's names, and directors can read and update their division's roster rows and notes.
4. Loads the supervision list: real manager plus director for 95 staff. The two Walkers are left out until we know who is `mwalker@` and who is `mmwalker@`.
5. Prints a check-up list: notes from people not on the roster, roster staff with no notes in 30 days, and old roster rows not on the supervision list.

Tested against a local Postgres copy of the schema, not your live database.


