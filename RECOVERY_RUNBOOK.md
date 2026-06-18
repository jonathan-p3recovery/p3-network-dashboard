# P3 Network Dashboard — Process & Recovery Runbook

This repo is the **single source of truth** for the P3 Recovery weekly network
dashboard process. If a laptop dies, this file plus the `skills/` folder is enough
to rebuild the whole pipeline on a new machine. Keep it current.

Last updated: 18 Jun 2026.

---

## 1. What this pipeline does

Each week it turns the Hapana "Weekly Performance, All Locations" extract into the
live network dashboard at **https://p3-network-dashboard.netlify.app**, archives the
week's files, and notifies the leadership team.

Flow: **raw extract (Google Drive) → analytics build → network dashboard build →
publish to GitHub → Netlify deploys → archive to Drive → email the team.**

## 2. Where everything lives (the cloud copies that matter)

- **Process / skills** — this repo, `skills/` folder (versioned, recoverable).
  - `skills/p3-hq-weekly-dashboards*.skill` — the analytics skill (builds the per-location xlsx + network overview).
  - `skills/update-p3-network-health-dashboard/` — builds `index.html` from the network overview.
  - `skills/hq-weekly-dashboard-sequence/` — the orchestrator that runs the whole weekly sequence.
- **Live dashboard code** — this repo, `index.html` (deployed by Netlify on every push to `main`).
- **Raw inputs + weekly outputs archive** — Google Drive, parent folder id
  `13mAEcqR5bWrCN7x9_8l2l0KH52YFga8s`, one `YYYY_MM_DD_Week` subfolder per week.
- **Live site host** — Netlify, auto-deploys from this repo's `main` branch.

Because the repo, Drive, and Netlify are all cloud, **no single computer holds the
only copy of anything** once a run has been pushed.

## 3. Connectors the pipeline uses

- **Google Drive** — read raw extract, create weekly folders, upload archives. Connected. Can create folders (`canAddChildren` confirmed on the archive parent).
- **Gmail** — draft the team notification. Connected, but **draft-only** (the connector has no send tool). The draft is created; sending is currently a manual click, or routed via the cloud job in the target architecture (section 6).
- **GitHub** — no MCP connector exists. Publishing is done on a trusted machine (see section 5), never by handing a token to the agent.

## 4. Recover on a brand-new machine

1. Install the Claude desktop app + Cowork.
2. Reconnect connectors: Google Drive, Gmail.
3. Clone this repo: `git clone https://github.com/jonathan-p3recovery/p3-network-dashboard.git`
4. Authenticate GitHub once on the machine (GitHub Desktop sign-in, or `gh auth login`) so pushes work.
5. Re-install the skills from `skills/` (open each `.skill` / SKILL.md via Settings → Capabilities).
6. Point Cowork at the local repo folder and the P3 Launch HQ working folder.

That's the whole recovery. Target time: ~30 minutes.

## 5. Publishing to GitHub — current options (the one human-dependent step)

The Claude agent runs in an isolated Linux sandbox that **cannot** reach a Mac's
keychain or push on its own, and there is no GitHub connector. So today, publishing
needs one of:

- **GitHub Desktop** — agent commits, human clicks "Push origin" (~5 sec).
- **Mac-side auto-push watcher** — a `launchd` job that pushes whenever `index.html`
  changes, using the machine's own stored credential. Click-free, but Mac-dependent
  and only runs while that Mac is on.

Neither is the long-term answer (see section 6). The agent must **never** read a
personal access token from project knowledge to push — tokens leak, expire, and
can't be cleanly revoked from inside an automation.

## 6. Target architecture — fully cloud, not people-dependent (roadmap)

Goal: remove every human/Mac dependency. The blocker is that the build currently
runs inside the Claude agent (which needs a host machine). The fix is to move the
build into the cloud:

1. **Convert the two skills into standalone Python** (the dashboard build already is
   essentially deterministic; the analytics build is data processing — both port to
   plain scripts).
2. **GitHub Actions, scheduled (cron)** runs those scripts: pulls the raw extract
   from Google Drive (service account), builds the dashboard, commits `index.html`
   (Netlify then auto-deploys), uploads the archive to Drive, and sends the team
   email via SMTP/Gmail API with a service credential.
3. **Validation gates** in the workflow hard-stop on bad data (revenue out of band,
   wrong location count, nulls, Netlify not updated) and **email an alert** instead
   of publishing — a human is involved only on exception.
4. **Daily / live-data future:** same workflow on a daily schedule; if Hapana
   exposes an API, the job pulls live data directly and the manual extract disappears
   entirely.

This is the path to "no one is a dependency." It's a build, not a toggle — track it
as a project.

## 7. Weekly run — quick reference

1. Drop the raw Hapana extract into the pre-created `YYYY_MM_DD_Week__DROP_RAW_HERE`
   folder in Drive.
2. Run the **HQ-Weekly-Dashboard-Sequence** skill.
3. It normalizes the folder name, builds + validates the dashboard, writes `index.html`,
   archives to Drive, creates next week's drop folder, and drafts the team email.
4. Publish (push) and send the drafted email (current manual touch points).
