# Standing up a new client roadmap

~30 minutes. Prereqs: a git host (GitHub) and Vercel access. Read `SPEC.md` first — especially §2 (timeline geometry) and §6 (guardrails).

## 1. Create the repo

```bash
mkdir ~/Documents/<CLIENT>-Roadmap && cd ~/Documents/<CLIENT>-Roadmap
git init
cp <path-to-roadmap-template>/template.html index.html
git add index.html && git commit -m "Initial roadmap from template"
git branch draft            # the review lane — see step 5
```

Copy this repo's `.claude/commands/` folder too if the PM edits with Claude Code (see step 6).

The initial setup (steps 2–3) can be committed straight to `main` — the review-lane discipline (step 5) starts once the board is deployed and client-facing. Content updates after that always go through `draft`.

## 2. Configure the skeleton (search for `SETUP:` comments in index.html)

1. **Client name** — replace every "Sample Client" (title, meta description, eyebrow, h1, subtitle, footer). Pick a favicon emoji.
2. **Timeline** — the standard board is quarterly: a 16-column grid = 2 carryover weeks from last quarter + 13 quarter weeks + 1 next-quarter horizon column (SPEC §2). Only change the column count for a bounded engagement, and then change `repeat(16, 1fr)` in `.row-track`, the cell runs, sprint bands, and week labels together.
3. **Dates** — set the sprint band labels (sprint numbers CONTINUE across quarters — don't reset to 1), the 16 week labels (Mondays), and the two JS dates at the top of the `<script>` block (`ganttStart`, `ganttEnd`). The current sprint number and dates come from the Jira board's active sprint (e.g. "GOAT Sprint 11") — no need to guess. If the client's sprints aren't Monday-aligned (Wednesday starts are common), snap each band to the nearest Monday columns and put the real dates in the band label — the label is the truth, the columns are the approximation.
4. **Jira** — replace `YOURORG.atlassian.net` and the `PROJ-` prefix with the client's real org/project.
5. **Milestones** — real release cadence; keep the quarter-boundary marker in the last column (relabel it "Contract End" for bounded engagements).
6. **Clean up** — delete the `SETUP:` comments as you resolve them; a configured board should have none left (they contain placeholder strings that trip the final verification grep).
7. **Sticky labels** (nothing to configure) — row titles stay pinned while the timeline scrolls horizontally; it's plain CSS and adapts to any column count.

## 3. Replace the example content

The template's lanes/rows are exemplars — one per pattern (done epic, crossing bar, NEW sub-item, SPIKE, TEST, CONV decision, BUG, proposed, deferred). Duplicate the pattern you need, delete the rest, keep the legend in sync with the lanes you actually use. **Copy the 16-cell filler run exactly** when adding rows. Each quarter thereafter, do the quarter roll (SPEC §2) as its own reviewed commit.

## 4. Deploy

Vercel → New Project → import the repo → framework preset "Other", no build step, output dir `/`. The `main` branch becomes the client-facing URL. Consider Deployment Protection if drafts must not be publicly viewable.

## 5. The draft review lane

All content updates happen on the `draft` branch; Vercel automatically gives it a stable branch-preview URL. Review the rendered preview, then merge `draft` → `main` to publish. Nothing edits `main` directly — this is what makes automated/delegated edits safe.

**Draft lanes (internal-only staging).** A whole swimlane can be marked internal by adding `draft` to the group — `<details class="group draft" open>` (SPEC §4). It shows on the `draft` preview with a `DRAFT · INTERNAL` badge but must never reach `main`. Use it for net-new workstreams you're floating before confirming with the client. Before publishing, run `/roadmap-publish <client>`: it walks each draft lane and makes you **promote** it (remove the `draft` class → client-facing) or **drop** it, and refuses to publish while any remain.

## 5a. Lock the guard (one-time, ~5 min)

The draft-lane gate is enforced two ways; set up the hard one once per repo:

1. **CI check** — `.github/workflows/no-draft-on-main.yml` ships in this repo. It fails any push/PR to `main` whose `index.html` still contains a draft-tagged group. Nothing to configure; it runs on push once the repo is on GitHub.
2. **Branch protection** — GitHub → repo Settings → Branches → add a rule for `main` → require the status check **`no-draft-on-main`** to pass before merging. This is what makes it fail-closed: even a manual UI merge of a draft lane is blocked, not just Claude-driven merges.

## 6. Editing with Claude (optional but recommended)

This repo ships four skills in `.claude/commands/`:

- `/roadmap-new` — walks Claude through steps 1–3 above interactively.
- `/roadmap-update [transcript-path | jira]` — updates the roadmap from a meeting transcript or your Jira board, enforcing SPEC encoding + guardrails, and ends with a merge-review changelog. Unconfirmed net-new workstreams land in a `draft` lane.
- `/roadmap-publish` — the pre-merge gate: resolves every `draft` lane (promote / drop / hold) so the board is client-clean before you merge `draft` → `main`.
- `/roadmap-audit` — the read-only accuracy deep-check: reconciles the board against Jira, the latest transcript, the client's own tracker, and Confluence/Slack; its internal-only report saves outside the repo.

All of them read client specifics from `roadmap.config.json` (copy `roadmap.config.example.json`, fill it in, commit it — `live_url` can stay `""` until the Vercel deploy in step 4 exists). To let `/roadmap-update` pull tasks straight from your Jira board, do the 15-minute setup in `JIRA_SETUP.md` (label client-visible tickets `roadmap`, parent them to lane epics, fill the config's `jira` section).

## 7. Full automation (optional, advanced)

One production setup goes further: meetings auto-record, a daemon extracts the verbatim transcript, and a watcher runs the update headlessly on `draft` with a desktop notification for review. That stack is Mac-specific and lives outside this repo — ask the maintainer before attempting to replicate it. The roadmap format works fine with manual transcripts.
