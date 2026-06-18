---
name: hq-weekly-dashboard-sequence
description: >-
  End-to-end orchestrator for the P3 Recovery weekly network dashboard. Runs the
  whole Monday pipeline in order: build the per-location analytics, rebuild the
  network HTML dashboard, publish it to GitHub, verify the live Netlify site,
  archive the week's files to Google Drive, and draft the team notification email
  for sign-off. Use this whenever the user says run the weekly dashboard
  sequence, do the Monday dashboard run, refresh and publish the network
  dashboard, "kick off the weekly pipeline", or drops a fresh Hapana weekly
  extract and wants the full publish-and-notify flow — not just one stage. For a
  single stage, use the underlying skill instead (p3-hq-weekly-dashboards for the
  xlsx analytics, update-p3-network-health-dashboard for the HTML rebuild only).
---

# HQ Weekly Dashboard Sequence

You are the P3 Weekly Dashboard orchestrator. Run the steps below **in order**.
After each step, report a one-line status. Honour every "stop and report" gate —
they exist so a bad or stale run never reaches the team's inbox.

Australian English in any prose. All revenue incl-GST.

## Before you start — preconditions

This pipeline only makes sense when there is a **new completed week** to publish.
Before Step 1, confirm both:

1. **A fresh Hapana "Weekly Performance, All Locations" extract** has been dropped
   in `/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ` (or the user
   has pointed you at one). No new extract → nothing to do; stop and say so.
2. The week it covers is **newer than the latest week already published** (check the
   most recent `YYYY_MM_DD_Week` folder under `HQ_Location_Dashboards_V2/`). If the
   newest extract matches a week that's already live, stop — re-publishing and
   re-emailing the team about a week they already have is the main failure mode this
   guard prevents. Override only if the user explicitly says to re-run a past week.

**One-time connector setup.** Steps 4, 6 and 8 need connectors to be linked first;
without them those steps can't run unattended:

- **GitHub** (Step 4) — an OAuth GitHub connector. **Do not** use a stored personal
  access token to authenticate; this skill never reads or embeds a raw secret token.
  If no GitHub connector is connected, fall back to the Step 4 hand-off.
- **Google Drive** (Step 6).
- **Gmail** (Step 8).

If a needed connector is missing, do that step's documented fallback and flag it in
the final summary — don't silently skip.

## Step 1 — Run the analytics skill

Run the saved skill **`p3-hq-weekly-dashboards`** (this is the installed analytics
skill — the "P3 weekly dashboard" / "HQ weekly dashboards" build). Let it finish
fully. Report:

- Week ending date processed
- Number of locations in the output
- Any errors or warnings

If it fails, **stop here** and report the error. Do not proceed.

## Step 2 — Verify output files exist locally

In `/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ` (inside the new
week's `HQ_Location_Dashboards_V2/YYYY_MM_DD_Week/` folder), confirm both are present
and freshly modified (mtime within this run):

- `P3_HQ_Network_Overview*` — must be present
- at least one `P3_HQ_Dashboard_*` location file — must be present

If anything is missing or stale, **stop** and report which files. Do not proceed.

## Step 3 — Run the dashboard update skill

Run the saved skill **`update-p3-network-health-dashboard`**. Let it finish fully.
Report:

- Week ending date used
- Number of **trading** locations processed (and any excluded as pre-launch — 0
  active members)
- Confirm `index.html` was written to
  `/Users/jonathanmcalees/Documents/GitHub/p3-network-dashboard/`
- Any missing or incomplete location data

If it fails, **stop here** and report the error. Do not proceed.

## Step 4 — Publish index.html to GitHub

Publish the new `index.html` to the repo. Owner/repo come from the existing git
remote (`jonathan-p3recovery/p3-network-dashboard`), branch `main`, path
`index.html`, commit message `Dashboard update — week ending <WEEK_ENDING_DATE>`.

**Never authenticate with a stored/personal access token, and never read a secret
out of project knowledge.** Use, in order of preference:

1. **GitHub OAuth connector** — if one is connected, use it to create/update
   `index.html` on `main` with the file content from
   `/Users/jonathanmcalees/Documents/GitHub/p3-network-dashboard/index.html` and the
   commit message above. This is the path that makes the run fully unattended.
2. **No connector → safe hand-off.** Commit locally
   (`git add index.html && git commit -m "…"`), then **stop** and ask Jonathan to push
   via GitHub Desktop (one click) — the agent sandbox has no push credentials. Do not
   continue to Netlify/Drive/email until the push is confirmed.

If the push fails, **stop** and report. Do not proceed to email.

## Step 5 — Verify Netlify deployment

Wait ~45 seconds after the push, then fetch `https://p3-network-dashboard.netlify.app`
and confirm it has updated — specifically that the header shows the correct
"Week ending <WEEK_ENDING_DATE>". Report the confirmation (or, if it hasn't flipped
yet, note it may still be building and re-check once before flagging).

## Step 6 — Archive the week's files to Google Drive

In the Google Drive folder id `13mAEcqR5bWrCN7x9_8l2l0KH52YFga8s`, create a subfolder
named `YYYY_MM_DD_Week` (the week ending date). Upload that week's output files from
`/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ` (the Network Overview
+ all location dashboards for the week). Confirm each file uploaded. If Drive isn't
connected, note it and continue — this is an archive step, not a blocker for the
email.

## Step 7 — Draft the notification email (review before send)

Draft the email below and **present it to Jonathan**. Do **not** send until he
approves or edits. Pull the highlight numbers from this week's network data on the
**same trading basis the dashboard uses** (pre-launch locations excluded), so the
email matches the live site.

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

Quick pre-send checks: revenue/WoW/members/on-track all reconcile with the dashboard;
`<TOTAL_LOCATIONS>` is the trading count; no health/medical or guaranteed-outcome
claims have crept into any custom highlight wording.

## Step 8 — Send the email

Only after Jonathan confirms, send via Gmail to all recipients above. Sending email
is an explicit-permission action — his confirmation in chat is the go. Report:

- Recipients (To + CC)
- Time sent
- Week ending date in the subject

If Gmail isn't connected, leave the approved draft ready and tell him it needs the
Gmail connector before it can send.

## Step 9 — Final summary report

Report the full run:

- Analytics skill: complete / failed
- Dashboard skill: complete / failed
- GitHub push: complete (connector) / handed to GitHub Desktop / failed
- Netlify deployment: live / still building / not verified
- Google Drive upload: complete / skipped (not connected)
- Email: sent to brad, marc, mark, paul / drafted, awaiting send
- Week ending date
- Total run time
- Any warnings or items needing attention
