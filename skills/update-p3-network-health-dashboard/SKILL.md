---
name: update-p3-network-health-dashboard
description: >-
  Rebuild the P3 Recovery Network Health Overview dashboard (a single static
  HTML file) from the latest weekly analytics outputs in the P3 Launch HQ
  folder. Use this skill whenever the user mentions the network dashboard,
  network health overview, "path to tier goal" across all locations, refreshing
  or rebuilding the network dashboard, updating index.html for the
  p3-network-dashboard repo, or drops a refreshed P3_HQ_Network_Overview file
  and wants the network view updated — even if they don't say the word "skill"
  or name the exact file. This is the network-wide roll-up (all locations on one
  page), distinct from the per-location weekly xlsx dashboards. If the user only
  wants the per-location xlsx dashboards rebuilt, use the p3-hq-weekly-dashboards
  skill instead; use THIS skill for the consolidated HTML network overview.
---

# Update P3 Network Health Dashboard

You are the P3 Network Dashboard updater. The job is to read the latest weekly
analytics output files, extract the network and per-location data, drop that
data into a fixed HTML template, and write the finished dashboard to the
project's GitHub folder. The template's CSS, layout, and rendering logic never
change — only the data does. Follow the steps below exactly.

All revenue figures are incl. GST. Use Australian English in any prose you add.

## Step 1 — Locate the latest week and read its analytics files

The weekly extracts live in **dated subfolders**, not at the top level. Under:
`/Users/jonathanmcalees/Documents/Claude/Projects/P3 Launch HQ/HQ_Location_Dashboards_V2/`
each week has its own folder named `YYYY_MM_DD_Week` (e.g. `2026_06_14_Week`),
where the date is the week-ending Sunday.

Pick the **most recent** dated subfolder (highest `YYYY_MM_DD`) and work only from
it — don't read every week. Ignore any `TEST_*` directories; those are scratch
copies, not live data. If the user drops a specific week's file or names a week,
use that one instead.

Inside the chosen week folder, read:

- `P3_HQ_Network_Overview*.xlsx` — the network summary file (**primary data source**)
- `P3_HQ_Dashboard_*_V2.xlsx` — individual location dashboard files (one per location)

For `.xlsx` files, read every sheet (use the `xlsx` skill or a short
pandas/openpyxl script in the sandbox if you need to inspect them). The network
file's `Executive_Summary`, `Location_Health`, and `Path_to_Goal` sheets carry
most of what you need; each location file's `Monthly_View` sheet carries the
monthly actuals/MTD/projection/target. Derive the week-ending date string from the
folder name (or the Network Overview title row).

Extract the following.

**From the Network Overview file:**

- Week ending date
- Network weekly revenue and WoW %
- Active members, new members, cancellations, net adds
- Per location: revenue last week, plan target, % of target, status, WoW %,
  baseline monthly, tier goal, plan weeks, crossover month, active members,
  net adds
- On Track / Close / Behind counts
- Spotlight items (top performer, approaching ratchet, crossover imminent,
  needs attention / pre-ramp)

**From each location file:**

- Location name
- Monthly actuals (last 3 complete months, with month labels)
- Current-month MTD and projection
- Monthly target

If a location file is missing, incomplete, or a location is dormant/pre-ramp,
note it — you'll report it in Step 5 and flag it in the data (`preramp: true`).

## Step 2 — Build the data objects

Construct these JavaScript values from what you read. They get inserted into the
template in Step 3.

```js
NET = { revenue, wow, active, newM, canc, netAdds }

// cross: map of crossover-month label -> sortable numeric key (e.g. {"Sep 2026": 202609, "—": 999999})
// so the "Crossover" column sorts chronologically.
cross = { /* label: sortKey */ }

DATA = [ /* one object per location */ {
  name, status, lastWk, pct, wow, monthly, goal, planWks, crossover,
  active, netAdds, preramp /* true only if dormant/pre-ramp */
} ]

MONTHLY = { /* keyed by location name */
  "Location Name": {
    months: [label, label, label],   // e.g. ["Mar 2026","Apr 2026","May 2026"]
    vals:   [num, num, num],          // matching monthly actuals
    mtd:    num,                       // current-month revenue to date
    proj:   num,                       // MTD projection for full month
    target: num,                       // monthly tier-goal target
    opened: "Mon YYYY",                // only for new locations
    preramp: true                      // only for dormant/pre-ramp locations
  }
}
```

### Exclude pre-launch / not-yet-trading locations

A location that hasn't gone live yet (recently signed, fitting out, or dormant)
shouldn't pollute the network view — it shows as a spurious "Behind" and drags the
status counts even though it has no real plan to miss. The clean signal for "not
trading yet" is **active members = 0**: every live location carries 200+ members, so
a location at 0 active members is pre-launch or dormant, regardless of any token
soft-trade revenue.

So, after building the raw lists: **exclude every location with 0 active members**
from `DATA`, `MONTHLY`, the status counts, the spotlight performance items, and the
Monthly Matrix. Do **not** use a revenue threshold as the exclusion test — a
genuinely live location can have a weak week under $5K, and you never want to hide a
real underperformer. Active members is the trigger; a location re-enters
automatically the first week it records members.

When a location is excluded, recompute the **network rollups on the trading-set
basis** so the KPI strip ties to the visible rows:

- `NET.revenue` = sum of the trading locations' `lastWk`.
- `NET.wow` = (this-week trading sum ÷ prior-week trading sum) − 1, where prior-week
  per-location revenue comes from the `Revenue_Trend` sheet's prior W/E column for the
  same trading set. (Don't reuse the network WoW from the file — that's the all-sites
  figure.)
- `NET.active`, `newM`, `canc`, `netAdds` = sums over trading locations. Pre-launch
  sites contribute zero, so these usually match the file's totals unchanged.

Keep the excluded locations on the radar without giving them a performance row: name
them in the **spotlight "Pre-ramp / needs review"** slot (`[PRERAMP_LOCATIONS]`,
e.g. "Hamilton &amp; Warners Bay — pre-launch, excluded until first active members")
and in the Monthly Matrix `[FOOTNOTE]`. The header `[LOCATION_COUNT]` is the trading
count (excluded sites don't count), and the subtitle reads "N trading locations".

Also derive:

- **Week ending date string** — for the header subtitle.
- **Generated date** — today's date, for the tag.
- **MTD_DAYS / MONTH_DAYS** — the day-count basis the extract was taken on, **not
  today's date**. The source files compute each location's MTD and projection on a
  fixed basis (e.g. "Jun 2026 through day 14 of 30", so projection = MTD × 30/14).
  Read that basis from any location file's `Monthly_View` note ("...through day X of
  Y") and set MTD_DAYS=X, MONTH_DAYS=Y. This keeps the dashboard's projection maths
  and "Day X/Y" labels consistent with the underlying numbers. (The data is a weekly
  snapshot, so this is usually a few days behind today — that's expected.)
- **Location count** — number of **trading** locations (after exclusion).
- **On Track / Close / Behind counts** — over the trading set, each as a % of the
  trading count for the status-bar widths. Status rule: On Track ≥ 95% of this week's
  plan target; Close 80–95%; Behind < 80%.
- **Spotlight items** — top performer this week, approaching tier ratchet,
  crossover imminent (≤10 weeks), pre-ramp / needs review.

`pct`, `wow`, and pace values in the template are **decimals** (e.g. 0.97 for
97%, -0.034 for −3.4%), because the render helpers multiply by 100. Revenue,
members, and day counts are plain numbers.

## Step 3 — Generate the HTML

Read the template from `assets/dashboard_template.html` (bundled with this
skill). Replace **only** the data and the bracketed `[PLACEHOLDER]` tokens — do
not touch any CSS, layout, structure, or the render code below the
`// ── RENDER — do not modify below this line ──` comment.

Placeholders to fill in the template:

- `[LOCATION_COUNT]`, `[WEEK_ENDING_DATE]`, `[GENERATED_DATE]`
- Status bars: `[ON_TRACK_PCT]` / `[ON_TRACK_COUNT]`, `[CLOSE_PCT]` /
  `[CLOSE_COUNT]`, `[BEHIND_PCT]` / `[BEHIND_COUNT]`
- Spotlight: `[TOP_PERFORMER]`, `[RATCHET_LOCATION]`, `[CROSSOVER_IMMINENT]`,
  `[PRERAMP_LOCATIONS]`
- Monthly Matrix note: `[LIVE_MONTH]`, `[MTD_DAYS]`, `[MONTH_DAYS]`,
  `[FOOTNOTE]`
- The data block between the `<script>` tags:
  `[INSERT_NET]`, `[INSERT_CROSS]`, `[INSERT_DATA]`, `[INSERT_MONTHLY]`,
  `[INSERT_MTD_DAYS]`, `[INSERT_MONTH_DAYS]`

Replace `const NET=[INSERT_NET];` with `const NET={...};` (an object literal),
`const cross=[INSERT_CROSS];` with `const cross={...};`, `const DATA=[INSERT_DATA];`
with the array literal, and `const MONTHLY=[INSERT_MONTHLY];` with the object
literal. `MTD_DAYS` and `MONTH_DAYS` become plain integers.

Sanity-check before writing: the data is valid JavaScript (well-formed objects/
arrays, no trailing commas that break parsing, all strings quoted), every
`[PLACEHOLDER]` has been replaced, and the counts in the status bars add up to
the location count.

## Step 4 — Write the file

Write the completed HTML to:
`/Users/jonathanmcalees/Documents/GitHub/p3-network-dashboard/index.html`

Overwrite the existing file completely. If the folder doesn't exist, create it.

## Step 5 — Confirm

Report back concisely:

- Week ending date used
- Number of trading locations shown, and which locations (if any) were **excluded as
  pre-launch** (0 active members)
- Any locations where data was missing or incomplete (and how you handled them)
- Confirmation that `index.html` was written to the GitHub folder
