---
description: Instantiate a new client roadmap from template.html — walks through naming, timeline geometry, dates, Jira links, and initial content. Usage: /roadmap-new <client-name>
allowed-tools: Read, Edit, Write, Grep, Glob, Bash
---

Stand up a new client roadmap for **$1** from this template repo, following `NEW_CLIENT.md`. Read `SPEC.md` §2 (timeline geometry) before touching the grid.

## Gather from the user first

Ask for anything not already provided: client name, which quarter the board covers and the current sprint number (numbering continues across quarters — never reset to 1), Jira org + project prefix, release cadence, whether the engagement is an ongoing retainer (default; next-quarter horizon column) or bounded (contract-end column), and the initial set of lanes and work items (or a kickoff doc/transcript to derive them from).

## Steps

1. Copy `template.html` to the target repo as `index.html` (create the repo and a `draft` branch if asked).
2. Work through every `SETUP:` comment in the file: client name in all six places, favicon emoji, sprint bands (continuing numbers), the 16 week labels, the two JS dates (`ganttStart`, `ganttEnd`), Jira org/prefix, milestones.
3. The standard grid is 16 columns (2 carryover + 13 quarter weeks + 1 horizon — SPEC §2). If a bounded engagement needs a different count, change `repeat(16, 1fr)` in `.row-track`, every 16-cell filler run, sprint bands, and week labels **together** — they must always agree.
4. Replace the exemplar lanes/rows with the client's real lanes and work items, reusing the template's patterns (done epic, in-flight, crossing, NEW sub-item, SPIKE, TEST, CONV, BUG, proposed, deferred). Keep lane color **class names** as-is; only the visible names change. Keep the legend in sync with the lanes actually used.
5. Write `roadmap.config.json` from `roadmap.config.example.json` with the real values.
6. Verify: no "Sample Client" or `YOURORG`/`PROJ-` strings remain; every bar has exactly one pill; every row-track has the full cell run; grid columns stay within the grid; **no named individuals anywhere** (SPEC §5–6).

## Output

Summarize what was configured (timeline span, lanes, item count) and list the manual steps left for the PM: push to git host, connect Vercel, protect deployments if needed (NEW_CLIENT.md §4–5).
