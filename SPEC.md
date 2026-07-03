# Client Roadmap Gantt — Locked Encoding Spec

This document is the source of truth for the roadmap HTML format used across Anatta client engagements (reference implementation: the Llama Naturals roadmap). `template.html` in this repo is a sanitized, working skeleton of the same file. **The encoding below is locked** — every roadmap must follow it exactly so that clients, PMs, and automation (Claude skills) can all read and edit any client's file the same way.

The file is **hand-authored static HTML** — no build step, no framework, no data file. All content lives in the markup; all styling lives in the single `<style>` block; the only JS positions the today line and wires the expand/collapse + column-resizer controls. Deploy anywhere that serves static files (we use Vercel).

## 1. Page anatomy (top to bottom)

| Section | Markup | Notes |
|---|---|---|
| Topbar | `.topbar` | Eyebrow (client name + quarter), Expand/Collapse all buttons, `DRAFT vX.X · Q3 Planning · June – Sept` meta |
| Title block | `.title-block` | `h1.title` + one-sentence `.subtitle` |
| Gantt | `.gantt-wrap > .gantt` | Today line, column resizer, then the rows below |
| Sprint band row | `.row.sprint-row` | One `.sprint-band` per sprint, `Sprint N · Mon D – Mon D` |
| Week label row | `.row.week-row` | One `.week-label` per column (Mondays) |
| Milestone row | `.row.milestone-row` | Release diamonds + quarter-boundary (or contract-end) marker |
| Swimlane groups | `details.group` | One per lane; `summary` = lane header row, children = work items |
| Legend note + legend | `.legend-note`, `.legend` | Keep in sync with the lanes/pills actually used |
| Footer | `footer` | `Working draft. vX.X · Client × Anatta · …` |

## 2. Timeline geometry (the load-bearing rule)

The standard board is **quarterly** — most clients are ongoing retainers. The track is a CSS grid: `grid-template-columns: repeat(16, 1fr)` — **2 carryover weeks from the prior quarter + 13 quarter weeks + 1 next-quarter horizon column**. Everything is placed with `grid-column: {startCol} / {endCol}`, and **endCol is exclusive** (a bar spanning weeks 1–2 is `grid-column: 1 / 3`).

- **1 column = 1 week; 1 sprint = 2 weeks = 2 columns.** If the client's real sprints aren't Monday-aligned (Wednesday starts are common), snap each sprint band to the nearest Monday columns and carry the true dates in the band label — labels are exact, columns approximate.
- **Columns 1–2 (carryover)** are the last two weeks of the previous quarter, always `prior`-shaded. They exist to hold recently-shipped work — the client should open the board and see things done, not an empty runway.
- **Columns 3–14** are the six two-week sprints of the quarter. **Sprint numbers continue across quarters** (Sprint 13, 14, … — never reset to 1); that's what says "ongoing relationship."
- **Column 15** is the quarter's 13th week — the wrap / next-quarter-planning week band.
- **Column 16** is the next-quarter horizon ("Q4 →" band + "Q4 KICKOFF" milestone, `renewal-band`/`renewal` classes). For a **bounded engagement**, relabel this column "Contract End" instead — same classes, different words.
- Weeks that have elapsed carry the `prior` class on their cells, week labels, and sprint bands (darker tint). Update `prior` shading as part of content edits, not on its own schedule.
- Every `.row-track` begins with the same run of 16 filler `<div class="cell">` divs (with `prior` on the elapsed ones) — copy the run exactly when adding a row.
- Changing the column count means changing `repeat(16, 1fr)` in `.row-track`, every cell run, the sprint bands, and the week labels together. Do it only at a quarter roll, never mid-quarter.

**The quarter roll** (once per quarter, ~30 min): re-baseline the board for the new quarter — new carryover columns (the outgoing quarter's last two weeks, holding its shipped work), new week labels and sprint bands (numbering continues), new JS dates, unfinished bars re-placed on the new grid, Proposed items promoted or dropped, and the version bumped. The old quarter's board survives in git history; don't try to make one file show two quarters.

**Today line:** positioned by JS from the two dates at the top of the `<script>` block — `ganttStart` (Monday of the first carryover column) and `ganttEnd` (end of the horizon column). Set them at each quarter roll; the line then tracks in real time, and its flag shows just "TODAY · date" (no day-of-quarter counter — removed in v1.1). Bars that straddle the line get `crosses` + an inline `--past: N%` (percentage of the bar already elapsed — recompute it whenever you move/extend a crossing bar).

## 3. Bar encoding

Every bar is:

```html
<div class="bar {lane} {state}" style="grid-column: {start} / {end};" title="{safe context}">
  <span class="bar-type">{PILL}</span><span class="bar-label">{label}</span>
</div>
```

Epic bars on lane headers use bare text + a trailing `<span class="bar-status">Sprint N–M</span>` instead of `bar-label`.

**Lane color classes** (keep the class names; your visible lane names are free):

| Class | Color | Hex |
|---|---|---|
| `cart` | dark teal-blue | `#2C5266` |
| `plp` | dark amber | `#7A5520` |
| `experiments` | dark bluish green | `#1F5E45` |
| `implementations` | dark plum | `#614258` |
| `compliance` | dark slate blue | `#3D5F73` |
| `promotions` | dark teal | `#1F5D5D` |

The palette is a darkened Okabe-Ito set: colorblind-distinguishable hue families, luminance dropped for WCAG AA+ white-text contrast. Don't substitute colors. Signal red (`--signal`, `#8B3D14`) is reserved for the quarter-boundary/contract-end marker and proposed-bar accents — never for lanes.

**State classes** (added after the lane class):

| State | Class | Rendering | Meaning |
|---|---|---|---|
| In-flight | *(none)* | solid lane color | scheduled and being worked |
| Shipped | `done` | gray, 70% opacity | done — set label/status to "Shipped" + date |
| Proposed | `proposed` | dashed red outline, hatched | surfaced, not yet scheduled (no lane class needed) |
| Deferred | `deferred` | dashed gray outline, strikethrough | parked; kept visible for the record |
| Crossing today | `crosses` + `--past: N%` | left portion darkened | in-flight bar straddling the today line |
| Epic | `epic` | taller, bolder | lane-header summary bar (combine with lane/state) |

**Task-type pills** (`bar-type` span, exactly one per bar):

| Pill | Meaning |
|---|---|
| `IMPL` | direct implementation |
| `TEST` | A/B test / experiment |
| `BUG` | defect |
| `SPIKE` | investigation |
| `CONV` | conversation / decision needed |

## 4. Row encoding

```html
<div class="row">
  <div class="row-label indented">
    <span class="label-top">
      <a href="https://YOURORG.atlassian.net/browse/PROJ-NN" target="_blank" rel="noopener">Item Title</a>
      <a class="id" href="…same…" target="_blank" rel="noopener">PROJ-NN</a>
    </span>
    <span class="row-meta">status note · context</span>
  </div>
  <div class="row-track"> {11 cells} {bar} </div>
</div>
```

- `indented` = normal work item; `sub-indented` = child of another item (renders a `↳`).
- Unticketed items: plain `<span>` title + `<span class="id">NEW</span>` instead of Jira links. Give them a real ticket link once filed.
- `row-meta` is a short italic status line — neutral, outcome-focused (see guardrails). Hidden on mobile, so never put load-bearing info only there.
- Milestones: `.milestone` with a `.diamond` — solid green `release` = shipped, outlined `release projected` (+ `projected` on the wrapper, `~` before the date) = projected, big red `renewal` = quarter boundary (or contract end on bounded engagements).

## 5. Writing rules

- **Full words, no shortforms** — "Sprint 5", never "S5"; "Investigation", never "Invest.".
- **No named individuals anywhere** — bars, labels, row-metas, tooltips. "Pending client sign-off", never "Awaiting Brad".
- Bar `title` attributes (tooltips) are client-visible: same rules as labels.
- One font family, tabular numerals — already in the CSS; don't add fonts or touch styles for content edits.

## 6. Information-exposure guardrails (CRITICAL)

The roadmap is a **client-facing artifact**. Internal talk must never reach it. **NEVER include:**

- Commentary about the client or their stakeholders as people — responsiveness, mood, competence, "goes quiet then complains", being a bottleneck.
- Team sentiment, venting, frustration, jokes, off-hand remarks.
- Internal economics: hours, budget, margin, rates, capacity, who's overloaded.
- Renewal odds, account risk, sales/upsell strategy, any internal read on the relationship.
- Blame, fault, or speculation about causes involving named people.
- Named individuals anywhere (restated from §5).
- Security/credential details, internal URLs, tokens, infra specifics.
- Anything a reasonable client would be unhappy to see written about their account.

**DO** describe only the work — shipped, slipped, re-scoped, added, deferred — in neutral, professional, outcome-focused language. A blocked item becomes a status ("Pending client sign-off"), never a story about a person.

**Fail closed:** if it's ambiguous whether something is client-safe, leave it out. Losing a minor update is always cheaper than leaking internal talk.

## 7. Update discipline

- Edits are **surgical**: touch only the bars/rows/metas the source (meeting, ticket) justifies. Never restructure the page, rename lanes, or change CSS during a content update.
- Publishing flow: edit on a **`draft` branch** → review the rendered diff → merge to `main` (the client-facing deploy). Nothing goes straight to `main`.
- Every update ends with a plain-English changelog grouped **Shipped / Slipped / Scope / New / Deferred** — that's what the reviewer reads at merge time.
- Version the artifact in the topbar meta and footer (`DRAFT v1.0` → bump on meaningful revisions).
- Once per quarter, do the **quarter roll** (SPEC §2) as its own reviewed commit — never mixed into a regular content update.
