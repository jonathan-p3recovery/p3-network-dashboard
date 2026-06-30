# P3 Network Dashboard — Process & Recovery Runbook

Single source of truth for the P3 Recovery weekly network dashboard. This file plus
the `skills/` folder rebuilds the whole pipeline on a new machine.

Last updated: 29 Jun 2026.

**Documentation rule:** whenever any P3-owned skill changes, its documentation is
updated immediately (no asking). See §8 for the skill→doc registry.

---

## 1. What this pipeline does

Each week it turns the Hapana "Weekly Performance" extract into the live network
dashboard at **https://p3-network-dashboard.netlify.app**, archives the week's files,
and notifies the leadership team.

Flow: **drop raw extract in Drive → analytics build → dashboard build → validate →
publish to GitHub → Netlify deploys → archive to Drive → email the team.**

The Monthly Matrix tab includes a **sortable Trend column**: the sparkline shows a
4-month shape (Mar→Apr→May→Jun-projection) and the % is **June's projection vs the
Mar–May average** (a smoothed 3-month baseline — steadier than month-on-month). Click
any matrix header to sort — clicking Trend ranks by momentum. The 3 prior full months
(Mar/Apr/May here) are computed from the **raw extract** during the dashboard build
(they're not in the per-location Monthly_View, which only carries the 3 displayed
months), and stored as `prior:[...]` in each MONTHLY entry. Recently-opened locations
(earliest prior month < $15K) are flagged "new" and excluded from the ranking; a
safety cap also drops any |trend| > 200%. The matrix's monthly target column is
labelled **"Plan target"** (the month's plan milestone) — distinct from the round tier
bracket shown as "Tier goal" on the Network Overview tab. Layout is widened and wide
tables show a right-edge fade + "↔ scroll" hint when they overflow.

## 2. Where everything lives

- **Process / skills** — this repo, `skills/` folder (versioned, recoverable).
- **Live dashboard code** — this repo, `index.html` (Netlify deploys on every push to `main`).
- **Inputs + weekly outputs + archive** — Google Drive, synced to the Mac via **Google Drive for Desktop** ("Available offline"):
  `…/Library/CloudStorage/GoogleDrive-jonathan@p3recovery.com/Shared drives/P3 Global/Revenue /HQ_Location_Dashboards_V2`
  (note trailing space in `Revenue `). One `YYYY_MM_DD_Week` folder per week; the pending one is `YYYY_MM_DD_Week__DROP_RAW_HERE`.
- **Live host** — Netlify, auto-deploys from `main`.

Repo, Drive and Netlify are all cloud — no single computer holds the only copy.

## 3. How the raw data flows (Google Drive for Desktop)

The Drive folder is a **real synced folder on the Mac**, set "Available offline," so
files dropped in it are readable by the agent and outputs written back sync up
automatically. **No chat uploads, no connector size limits.** The weekly export comes
as two files: `Weekly Performance, All Locations*.xlsx` (primary) + `Weekly
Performance, NZ*.xlsx` (Mount Maunganui supplement) — both needed for the full 16-location network.

## 4. Connectors

- **Google Drive for Desktop** — the input/output/archive transport (synced folder). The Drive *connector* (MCP) is used only for small ops (folder share links, metadata).
- **Gmail** — drafts the team email. Draft-only (no send tool); sending is a manual click.
- **GitHub** — no connector. Publish via GitHub Desktop on the Mac; never hand a token to the agent, never run git from the agent sandbox (it leaves un-deletable `.git/*.lock` files — clear them in Finder under `.git` if it ever happens).

## 5. Recover on a new machine (~30 min)

1. Install Claude desktop + Cowork; reconnect Google Drive + Gmail.
2. Install Google Drive for Desktop, sign in as jonathan@p3recovery.com, set the `HQ_Location_Dashboards_V2` folder "Available offline".
3. Clone `github.com/jonathan-p3recovery/p3-network-dashboard`; sign into GitHub Desktop.
4. Re-install the skills from `skills/` via Settings → Capabilities.
5. Point Cowork at the repo folder, the P3 Launch HQ working folder, and the CloudStorage Drive folder.

## 6. Publishing to GitHub — the one human-dependent step

Agent writes `index.html` to the repo; **the human commits + Pull origin + Push origin
in GitHub Desktop**. The agent shell can't push (isolated sandbox, no keychain) and
must never use a stored token.

## 7. Target architecture — fully cloud (roadmap)

Move the build off the agent into **GitHub Actions** (scheduled): pull the extract
from Drive (service account), build, commit `index.html` (Netlify deploys), archive to
Drive, send email via Gmail API — with validation gates that email an alert instead of
publishing on bad data. Credentials live in GitHub Actions Secrets (never through the
agent). It's a build project needing dev support; bridge stays until then. Daily/live
data = same workflow on a daily cron, ideally pulling a Hapana API directly.

## 8. Skills & documentation (registry)

P3-owned skills and the doc kept current with each. **When a skill changes, update its
doc here too.**

| Skill | What it does | Doc |
|-------|--------------|-----|
| `hq-weekly-dashboard-sequence` | Orchestrates the full weekly run (this document's flow) | This runbook (§1–6) |
| `p3-hq-weekly-dashboards` | Builds per-location xlsx + network overview from the extract | This runbook §3 + the skill's own SKILL.md |
| `update-p3-network-health-dashboard` | Builds `index.html` from the network overview | This runbook §6 + the skill's own SKILL.md |

This runbook is maintained in **three synced copies** (keep in step): repo
`RECOVERY_RUNBOOK.md` (master), the copy in the Drive `HQ_Location_Dashboards_V2`
folder, and a local copy in `P3 Launch HQ`.

## 9. Weekly run — quick reference

1. Drop the raw export(s) into the `…__DROP_RAW_HERE` folder in Drive.
2. Run the **HQ-Weekly-Dashboard-Sequence** skill.
3. It builds + validates, writes `index.html`, archives to Drive, renames the week
   folder, creates next week's drop folder, and drafts the team email (with the Drive link).
   Note: the skill **renames** the drop folder (the Drive mount allows rename but not
   folder delete) and auto-handles new/pre-launch locations (0 active members → excluded).
4. Two clicks: GitHub Desktop (commit → Pull → Push), then send the Gmail draft once the site is live.
