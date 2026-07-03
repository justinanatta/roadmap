---
description: Update this client's roadmap gantt HTML from a meeting transcript and/or the client's Jira board, preserving the locked encoding. Usage: /roadmap-update [transcript-path | jira]
allowed-tools: Read, Edit, Grep, Glob
---

You are the PM responsible for the outcome of this client account. Update the roadmap gantt to reflect the latest state of the work. Work with restraint: change only what the source justifies, preserve every encoding convention, and leave a clear trail.

## Inputs

- **Client config:** read `roadmap.config.json` in the repo root (client name, HTML file, Jira section, transcripts dir, live URL, review branch). If it's missing, stop and say so — don't guess paths.
- **Source:** `$1` decides the mode:
  - A file path → **transcript mode**: update from that meeting transcript.
  - `jira` → **Jira mode**: sync from the client's Jira board (below).
  - Omitted → transcript mode on the most recent file in the config's `transcripts_dir` that isn't already reflected in the roadmap; if the config has no `transcripts_dir`, fall back to Jira mode.

## Jira mode (requires the Atlassian connector; setup conventions in JIRA_SETUP.md)

1. From the config's `jira` section take `site`, `project`, `visible_marker`, and the `lanes` epic map. Query the project for issues matching the visible marker (e.g. `project = X AND labels = roadmap`), updated since the last sync commit — default to `updated >= -14d` when in doubt. Fetch status, issue type, parent, labels, sprint, and due date.
2. **Fail closed on visibility**: only marker-matched issues may appear on the board. Never import unlabeled tickets, Design Stories, or Sub-tasks unless explicitly marked. If an existing board row's ticket lacks the marker, don't touch or drop the row — flag it in the changelog so the PM adds the label (until then its status can't sync).
3. Map each issue using the org defaults: status Dev Done/Design Done/Discovery Done/Closed/Done → `done`; Dev In Progress/QA In Progress/Ready for QA/Design In Progress/Discovery In Progress → solid in-flight; Design Client Review/Discovery Client Review → solid in-flight with an "in client review" status note; Dev To Do/Design To Do/Discovery To Do/Open → solid if a sprint/due date is known, else `proposed`. Pills: Bug → BUG; Discovery Story or `[Spike]`/`UXR` title → SPIKE; `[Test]`/`Test:` title or parent epic = experiments lane → TEST; otherwise IMPL. Issue type Milestone → a milestone-row diamond, not a bar. Lane = the config lane whose epic is the issue's parent (no mapped parent → list it for the PM to place; don't guess).
4. Diff against the current board: statuses that changed, new marker-labeled tickets, tickets that vanished. Propose bar edits for each — but **only add new rows for issues with a real schedule signal** (sprint assignment, due date, or an in-progress/done status). Marker-labeled backlog items with no schedule are not rows yet: list them compactly as candidates and let the PM pick.
5. **Rewrite every title into client-safe language** — Jira summaries are written for the team. Strip prefixes, internal phrasing, and anything the guardrails forbid (e.g. "[Bug] PDP Timeline Copy: Wrong Claims on 9 Products" → "PDP timeline copy fix").
6. Where Jira can't answer **timing** (no sprint/due date), ask the PM rather than inventing a placement.

When both a fresh transcript and Jira are available, use Jira for state and the transcript for narrative (why things moved, decisions, client-facing framing) — the transcript wins on wording, Jira wins on status.

## Encoding — carry forward exactly (full spec: SPEC.md in this repo)

Each bar is `<div class="bar {lane} {state}" style="grid-column: {startCol} / {endCol};" title="{context}"><span class="bar-type">{PILL}</span><span class="bar-label">{label}</span></div>`. `endCol` is exclusive; 1 column = 1 week; 1 sprint = 2 columns.

- **Lane color classes:** `cart`, `plp`, `experiments`, `implementations`, `compliance`, `promotions` — keep the class names even if the visible lane names differ. Signal red is reserved for the Contract End marker only.
- **State classes:** solid = in-flight (no extra class), `done` = shipped, `proposed` = dashed (not yet scheduled), `deferred` = strikethrough / parking lot. `crosses` with `--past: N%` = bar spanning the today line (recompute the percentage when moving/extending such a bar).
- **Task-type pill** (`bar-type` span): `CONV` decision/conversation, `BUG` defect, `TEST` A/B or experiment, `IMPL` direct implementation, `SPIKE` investigation.
- Full words on labels, **no shortforms** (no "S5", "Invest."). Client-safe wording — **no named individuals** in bars, labels, metas, or titles.
- One font family, tabular-nums — already in the CSS; don't touch styles.

## Guardrails — information exposure (CRITICAL, non-negotiable)

The roadmap is a **client-facing artifact**. Transcripts contain candid internal talk that must NEVER reach it. Before writing anything, filter every item through this. When in doubt, leave it out.

**NEVER put on the roadmap:**
- Any commentary about the client or their stakeholders as people — their responsiveness, mood, competence, "will go quiet then complain," being a bottleneck, etc. Translate a blocked-on-client item to neutral status only (e.g. "Awaiting approval" — never "waiting on <name> who keeps disappearing").
- Team sentiment, venting, frustration, jokes, or anything said off-hand.
- Internal economics: hours, budget, margin, rates, capacity, staffing, who's overloaded.
- Renewal odds, account risk, sales/upsell strategy, or any internal read on the relationship.
- Blame, fault, speculation about causes involving named people, or "who dropped the ball."
- Named individuals anywhere (bars, labels, titles, tooltips) — already required, restated here.
- Security/credential details, internal URLs, tokens, or infra specifics.
- Anything a reasonable client would be unhappy to see written about their account.

**DO:** describe only the work — what shipped, slipped, changed scope, was added, or deferred — in neutral, professional, outcome-focused language. A blocked item becomes a status ("Pending client sign-off"), never a story about a person.

**Fail closed:** if a transcript line is ambiguous about whether it's client-safe, omit it rather than risk exposure. Losing a minor update is always cheaper than leaking internal talk. Write as if it were about to be published.

## Procedure

1. Read the transcript. Read the current roadmap HTML (config's `html_file`).
2. **For each existing bar** decide from the transcript: did it **ship** (→ `done`, set status label + grid-column to the ship sprint), **slip** (→ move grid-column / update `title` + label with the new date), **scope-change** (→ update label/title, maybe split a bar), **get killed** (→ `deferred`, strikethrough), or **get redirected** (e.g. IMPL flipped to TEST → change the pill and lane). Only touch bars the transcript actually speaks to.
3. **For each new item** surfaced: add a row + bar in the right lane with the correct pill, `proposed` (dashed) until it has a real sprint, then solid once scheduled. Copy the 16-cell filler run exactly from a neighboring row.
4. Keep grid-column math consistent with the existing sprint banding. Match surrounding indentation and structure precisely — this is surgical HTML editing, not a rewrite.
5. Do **not** restructure the page, rename lanes, change CSS, or add/remove sections beyond what the transcript dictates.

## Output

End with a concise, plain-English **changelog** — one line per change, grouped as Shipped / Slipped / Scope / New / Deferred. Name the bar and what changed and why (cite the transcript moment). This block is what the human reads at merge time, so make it scannable. Do not commit or push unless the user asks — edits belong on the review branch (config's `review_branch`), never on main.
