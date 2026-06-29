---
name: hq-weekly-dashboard-sequence
description: >-
  End-to-end orchestrator for the P3 Recovery weekly network dashboard ("the
  Monday run"). Reads the week's raw Hapana extract from the synced Google Drive
  drop folder, builds the per-location analytics and the network HTML dashboard,
  validates the numbers, archives outputs back to the same Drive week folder,
  renames the folder and pre-creates next week's drop folder, and drafts the team
  notification email — leaving two human clicks (publish to GitHub, send the
  email). Use whenever the user says run the weekly dashboard sequence, do the
  Monday dashboard run, refresh and publish the network dashboard, "kick off the
  weekly pipeline", or has dropped a fresh Hapana extract in the Drive drop
  folder. For a single stage only, use the underlying skill instead
  (p3-hq-weekly-dashboards for the xlsx analytics,
  update-p3-network-health-dashboard for the HTML rebuild).
---

# HQ Weekly Dashboard Sequence

You are the P3 Weekly Dashboard orchestrator. Run the steps in order, reporting a
one-line status after each. Honour every "stop" gate.

Two human touches remain by design: **publishing to GitHub** (Step 8) and **sending
the email** (Step 11) — the agent shell can't push to GitHub and the Gmail connector
is draft-only. Never run git from the sandbox (it leaves un-deletable lock files);
the user commits via GitHub Desktop.

Australian English. Revenue incl-GST. Use the system clock for "today".

## Key paths

- **Drive pipeline folder (synced via Google Drive for Desktop, "Available offline"):**
  `…/Library/CloudStorage/GoogleDrive-jonathan@p3recovery.com/Shared drives/P3 Global/Revenue /HQ_Location_Dashboards_V2`
  (note the trailing space in `Revenue `). In bash, use this session's CloudStorage
  mount equivalent. This is the single home for inputs, outputs and archive — read and
  write it directly with file tools/bash; no Drive connector needed for file transfer.
  Each week is a subfolder `YYYY_MM_DD_Week`; the pending one is `YYYY_MM_DD_Week__DROP_RAW_HERE`.
- **Analytics script:** `p3-hq-weekly-dashboards` skill → `scripts/build_dashboards.py`.
- **Repo (live dashboard):** `/Users/jonathanmcalees/Documents/GitHub/p3-network-dashboard`, `index.html`, branch `main`.
- **Email:** Gmail connector (draft only). **Drive link** for the email = the week folder's share URL (get via the Drive connector `search_files` `title = '<YYYY_MM_DD_Week>'` → `viewUrl`).

## Step 1 — Find the raw extract in the Drive drop folder

In the Drive pipeline folder, open the `YYYY_MM_DD_Week__DROP_RAW_HERE` folder. Expect:

- `Weekly Performance, All Locations*.xlsx` — primary (AU + most locations).
- `Weekly Performance, NZ*.xlsx` — supplement (Mount Maunganui).

If the drop folder is empty / absent, **stop**: "No new extract in the Drive drop folder."
If files look like cloud placeholders (0 bytes / unreadable), the folder isn't
"Available offline" yet — ask the user to enable it.

## Step 2 — Confirm it's a genuinely new week

Read the extract enough to derive the **actual week-ending Sunday** from the data
(last complete week; ignore trailing all-zero future rows). Compare to the newest
clean `YYYY_MM_DD_Week` folder and the date in the live repo `index.html`. If it
matches a week already published, **stop** (override only if the user says re-run).

## Step 3 — Rename the drop folder (rename, never delete)

**Rename** (`mv`) the `…__DROP_RAW_HERE` folder to the clean `<YYYY_MM_DD_Week>`
(date from Step 2). The Drive mount allows **rename but not folder delete**, so we
rename-first rather than create-a-new-folder-then-delete (which leaves an
undeletable empty orphan). The raw extracts now sit in the clean week folder.

## Step 4 — Run the analytics build (into the same folder) + next drop folder

Run `build_dashboards.py` pointing `--output-dir` at the HQ pipeline folder; the
script writes the dashboards + `P3_HQ_Network_Overview_V2.xlsx` **into the existing**
`<YYYY_MM_DD_Week>` folder (same name, derived from the data), alongside the raw
extracts:

```
python <…>/build_dashboards.py \
  --input "<YYYY_MM_DD_Week>/Weekly Performance, All Locations*.xlsx" \
  --supplement "<YYYY_MM_DD_Week>/Weekly Performance, NZ*.xlsx" \
  --output-dir "<Drive pipeline folder>" \
  --no-libreoffice
```

Then create **next week's** drop folder: `<this week + 7 days>_Week__DROP_RAW_HERE`
(mkdir — that's allowed). Report: week ending, location count (expect all incl. Mount
Maunganui), any new/excluded locations, any errors.

## Step 5 — Verify outputs

In `<YYYY_MM_DD_Week>` confirm `P3_HQ_Network_Overview_V2.xlsx` + ≥1
`P3_HQ_Dashboard_*` exist and are fresh. If missing, **stop**.

## Step 6 — Build the network dashboard

Run `update-p3-network-health-dashboard` against the new week's
`P3_HQ_Network_Overview_V2.xlsx`. It writes `index.html` to the repo. Report: week
ending, trading-location count + any excluded as pre-launch (0 active members),
confirm `index.html` written.

## Step 7 — Validation gates (replace the human eye; any fail → stop, no publish/email)

1. **Revenue sanity** — network weekly revenue within ±25% of last week's.
2. **Location count** — trading count within ±2 of last week's.
3. **No holes** — no null `lastWk`/`pct`/`active` for any trading location.
4. **Clean render** — no leftover `[PLACEHOLDER]`/`INSERT_` tokens; JS data block parses.

## Step 8 — Publish to GitHub · HUMAN TOUCH #1

`index.html` is written to the repo working tree (do NOT git from the sandbox). Tell
the user: in **GitHub Desktop** → commit → **Pull origin** → **Push origin**. Wait for
confirmation. If unconfirmed, **stop** before email.

## Step 9 — Verify Netlify

~45s after push, fetch `https://p3-network-dashboard.netlify.app` (cache-buster query)
and confirm the header shows the correct week ending. Don't email until live.

## Step 10 — Archive the published HTML to Drive

Copy the repo's `index.html` into `<YYYY_MM_DD_Week>` as `index_published.html`
(the per-location xlsx + overview are already there from Step 3). Done — the synced
folder uploads automatically.

## Step 11 — Draft the team email (Gmail draft) · HUMAN TOUCH #2

Get the week folder's share URL (Drive connector `search_files title = '<YYYY_MM_DD_Week>'`
→ `viewUrl`). Create a **Gmail draft** (do not send) on the trading basis:

- **To:** brad@p3recovery.com, marc@p3recovery.com, mark@p3recovery.com, paul@p3recovery.com
- **CC:** jonathan@p3recovery.com
- **Subject:** `P3 Network Health Dashboard — Updated <WEEK_ENDING_DATE>`

```
Hi team,

The P3 Network Health Dashboard has been updated for the week ending <WEEK_ENDING_DATE>.

Live dashboard: https://p3-network-dashboard.netlify.app
Source files (Google Drive): <WEEK_FOLDER_DRIVE_LINK>

This week's highlights:
- Network weekly revenue: <NETWORK_REVENUE> (<WOW_PCT> vs prior week)
- Active members: <ACTIVE_MEMBERS> across the network
- New members: <NEW_MEMBERS> this week, <NET_ADDS> net adds
- Locations on track: <ON_TRACK> of <TOTAL_LOCATIONS>
- Top performer: <TOP_PERFORMER>

The full location breakdown, monthly matrix and trend data are all available in the dashboard.

Prepare. Prevent. Perform.
Jonathan
```

Numbers must match the live dashboard; `<TOTAL_LOCATIONS>` is the trading count; no
medical/therapeutic or guaranteed-outcome wording.

## Step 12 — Final summary

Report: week ending; analytics ✓/✗; dashboard ✓/✗; gates pass/which failed; GitHub
(committed — awaiting push / pushed); Netlify (live / building); Drive (folder
renamed, outputs archived, next drop folder created); email (drafted, awaiting send);
the two actions left for Jonathan (Pull+Push in GitHub Desktop; send the Gmail draft);
any items needing attention.
