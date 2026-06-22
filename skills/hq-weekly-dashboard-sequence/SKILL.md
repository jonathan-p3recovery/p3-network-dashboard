---
name: hq-weekly-dashboard-sequence
description: >-
  End-to-end orchestrator for the P3 Recovery weekly network dashboard ("the
  Monday run"). Pulls the week's raw Hapana extract from Google Drive, builds the
  per-location analytics and the network HTML dashboard, validates the numbers,
  archives everything back to Drive, pre-creates next week's drop folder, and
  drafts the team notification email — leaving two deliberate human clicks
  (publish to GitHub, send the email). Use whenever the user says run the weekly
  dashboard sequence, do the Monday dashboard run, refresh and publish the network
  dashboard, "kick off the weekly pipeline", or has dropped a fresh Hapana extract
  in the Drive drop folder. For a single stage only, use the underlying skill
  instead (p3-hq-weekly-dashboards for the xlsx analytics,
  update-p3-network-health-dashboard for the HTML rebuild).
---

# HQ Weekly Dashboard Sequence (bridge)

You are the P3 Weekly Dashboard orchestrator. Run the steps below **in order**,
reporting a one-line status after each. Honour every "stop" gate — they exist so a
bad or stale run never reaches the repo or the team's inbox.

This is the **bridge** design: everything automatable runs unattended; two
deliberate human touches remain — **publishing to GitHub** (Step 8) and **sending
the email** (Step 10) — because the agent shell can't push to GitHub and the Gmail
connector is draft-only. The cloud-native, zero-touch version is a future project;
see `RECOVERY_RUNBOOK.md` section 6 in the repo.

Australian English in prose. All revenue incl-GST. Today's real date governs "this
week" — confirm with the system clock if unsure.

## Key locations & connectors

- **Google Drive parent (archive + input):** folder id `13mAEcqR5bWrCN7x9_8l2l0KH52YFga8s`.
  Each week is a subfolder `YYYY_MM_DD_Week` (week-ending Sunday). Use the connected
  **Google Drive** connector (search/create/upload/download).
- **Local working folder:** `/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ`
  (where the analytics skill reads/writes).
- **Repo (live dashboard):** `/Users/jonathanmcalees/Documents/GitHub/p3-network-dashboard`,
  file `index.html`, branch `main`, remote `jonathan-p3recovery/p3-network-dashboard`.
- **Email:** the connected **Gmail** connector (draft only).

## Step 1 — Find this week's raw extract in Drive

In the Drive parent folder, locate the new week's raw Hapana "Weekly Performance,
All Locations" extract. It will be in either:

- a **drop folder** named like `YYYY_MM_DD_Week__DROP_RAW_HERE` (pre-created by the
  previous run), or
- the most recent `YYYY_MM_DD_Week` folder that contains a raw extract not yet
  processed.

If no raw extract is found anywhere new, **stop**: "No new weekly extract in Drive —
nothing to run." (This is the normal answer mid-week.)

## Step 2 — Confirm it's genuinely a new week

Read the raw extract enough to derive the **actual week-ending date** from the data
(don't trust the folder label alone). Compare against the latest week already
published (the newest clean `YYYY_MM_DD_Week` folder and the date in the live repo's
`index.html`). If it matches a week already live, **stop** — re-publishing and
re-emailing a week the team already has is the main failure mode this guard prevents.
Override only if the user explicitly says to re-run a past week.

## Step 3 — Download the raw extract locally & normalize the folder name

Download the raw extract from Drive into the local working folder
`/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ` so the analytics
skill can read it. Then **normalize the Drive folder name** to the clean
`YYYY_MM_DD_Week` (strip the `__DROP_RAW_HERE` suffix) using the derived week-ending
date, so a wrong pre-created guess self-corrects.

## Step 4 — Run the analytics skill

Run the saved skill **`p3-hq-weekly-dashboards`**. Let it finish fully. Report:

- Week ending date processed
- Number of locations in the output
- Any errors or warnings

If it fails, **stop** and report the error.

## Step 5 — Verify analytics outputs exist locally

In the new week's `HQ_Location_Dashboards_V2/YYYY_MM_DD_Week/` folder, confirm both
are present and freshly modified (mtime within this run):

- `P3_HQ_Network_Overview*` — must be present
- at least one `P3_HQ_Dashboard_*` location file — must be present

If anything is missing or stale, **stop** and report which files.

## Step 6 — Run the dashboard update skill

Run the saved skill **`update-p3-network-health-dashboard`**. Let it finish fully.
Report:

- Week ending date used
- Number of **trading** locations (and any excluded as pre-launch — 0 active members)
- Confirm `index.html` was written to the repo folder
- Any missing/incomplete location data

If it fails, **stop** and report.

## Step 7 — Validation gates (these replace the human eye)

Before anything is published or emailed, hard-check the new dashboard data against
the prior week. If **any** gate fails, **stop**, do not publish or email, and report
exactly which gate tripped and the figures involved:

1. **Revenue sanity** — network weekly revenue is within ±25% of last week's. A
   swing bigger than that is almost always a data problem, not reality.
2. **Location count** — trading-location count is within ±2 of last week's (a sudden
   jump/drop signals a parse error or a mis-flagged pre-launch site).
3. **No holes** — no null/blank `lastWk`, `pct`, or `active` for any trading location.
4. **Clean render** — `index.html` contains no leftover `[PLACEHOLDER]` or `INSERT_`
   tokens, and the JS data block parses.

These gates are what make a near-hands-off run safe — treat a tripped gate as a hard
stop, not a warning.

## Step 8 — Publish to GitHub  ·  HUMAN TOUCH #1

Commit `index.html` locally with message
`Dashboard update — week ending <WEEK_ENDING_DATE>`. Then hand off the push — the
agent shell **cannot** push and must **never** use a stored token:

- Tell Jonathan to **Pull origin, then Push origin** in GitHub Desktop (pull first —
  the remote may be ahead from web edits, so a plain push can be rejected).
- Wait for confirmation the push is done before continuing.

If the push can't be confirmed, **stop** before email and tell Jonathan the dashboard
is built and committed but not yet live.

## Step 9 — Verify Netlify deployment

~45 seconds after the push, fetch `https://p3-network-dashboard.netlify.app` and
confirm the header shows "Week ending <WEEK_ENDING_DATE>". If it hasn't flipped yet,
note it may still be building and re-check once before flagging. Do not proceed to
email until the live site shows the correct week.

## Step 10 — Archive the week's files to Drive

In the week's `YYYY_MM_DD_Week` Drive folder, upload this week's outputs from the
local working folder (the Network Overview + all location dashboards, and a copy of
the published `index.html`). Confirm each upload. If Drive is unavailable, note it
and continue — archiving is not a blocker for the email.

## Step 11 — Pre-create next week's drop folder

In the Drive parent, create next week's folder named
`YYYY_MM_DD_Week__DROP_RAW_HERE` (week-ending date = this week + 7 days), so Jonathan
always has a clearly-labelled place to drop the next extract. Step 3 of next week's
run will normalize the name.

## Step 12 — Draft the team email (Gmail draft)  ·  HUMAN TOUCH #2

Create a **Gmail draft** (do not send — the connector can't, and a human eye on exec
comms is the design). Pull the highlight numbers on the **same trading basis the
dashboard uses** so the email matches the live site.

- **To:** brad@p3recovery.com, marc@p3recovery.com, mark@p3recovery.com, paul@p3recovery.com
- **CC:** jonathan@p3recovery.com
- **Subject:** `P3 Network Health Dashboard — Updated <WEEK_ENDING_DATE>`

Body:

```
Hi team,

The P3 Network Health Dashboard has been updated for the week ending <WEEK_ENDING_DATE>.

You can view the live dashboard here: https://p3-network-dashboard.netlify.app

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

Then tell Jonathan the draft is in Gmail and he just clicks **Send** (it's pre-addressed).
Quick pre-draft checks: numbers reconcile with the dashboard; `<TOTAL_LOCATIONS>` is
the trading count; no medical/therapeutic or guaranteed-outcome wording.

## Step 13 — Final summary

Report:

- Week ending date
- Analytics skill: complete / failed
- Dashboard skill: complete / failed
- Validation gates: all passed / which tripped
- GitHub: committed — awaiting Jonathan's push / pushed & confirmed
- Netlify: live / still building / not verified
- Drive archive: complete / skipped
- Next week's drop folder: created
- Email: drafted in Gmail, awaiting Jonathan's send
- Total run time
- The two actions still needed from Jonathan: (1) Pull+Push in GitHub Desktop, (2) Send the Gmail draft
- Any warnings or items needing attention
