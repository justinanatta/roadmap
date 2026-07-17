# Roadmap v2 — Completion Overlay, Jira Refresh, Data-Quality Gate, Deep Audit & App Packaging

**Status:** DESIGN — not yet built. Build spec for a focused session (build → two independent adversarial reviews → one consolidated fix pass → ship).

**Origin:** a 2026 pilot engagement. The pilot board's completion numbers had drifted, and a read-only reconciliation across Jira, the latest meeting transcript, the client's own tracker, and Confluence showed the board was actually *understating* progress — the §3.2 failure classes are exactly what that reconciliation observed. v2 makes those failure modes hard to hit and the whole workflow easy for any PM to rinse and repeat.

---

## 0. Serving & visibility constraints (applies to everything below)

These repos deploy as **static sites: every committed file is publicly fetchable at the deploy URL** (verified: `.md` files at the repo root are served). The GitHub repo may also be publicly visible. Therefore:

- **Treat every committed file as visible to the repo's own client.** A client repo necessarily names its intended client — that's fine. What is forbidden everywhere (docs, configs, commit messages, this file) is: any *other* client's identity, any named individual, internal commentary or economics, internal paths, and anything SPEC §6 bans from the board. The shared template repo names no real client at all — placeholders only. Machine-local paths (a PM's `transcripts_dir`, `deep_sources.tracker_file`) are mild exposure in committed config — moving them to a gitignored local overlay (`roadmap.config.local.json`) is a phase-0/1 item.
- `.vercelignore` (added in this commit) excludes `*.md`, `.claude/`, `.github/`, config, and sync-state from deploys — verify on the first deploy after merge (curl a doc URL expecting 404); if the Git integration ignores it, switch to a `vercel.json` arrangement. It ships with the template so client repos inherit it. This is a mitigation, not a license — the GitHub side may still be visible, so rule 1 stands.
- Audit artifacts (Part C) are **never committed to these repos at all** — they live outside the repo (see Part C).

## 1. What already exists (do NOT rebuild) — with known gaps

- **Per-client config** (`roadmap.config.json`): client identity, `html_file`, `live_url`, `review_branch`, `transcripts_dir`, and a `jira` block (`site`, `project`, `visible_marker`, `lanes` = lane-class → feeding-epic map).
- **`/roadmap-update [transcript-path | jira]`** (repo-local; client identity comes from the repo's config, not an argument): transcript mode + a Jira mode that maps ticket *status* → bar state, rewrites titles client-safe, and stages net-new workstreams as draft lanes. **Known gap:** the command frontmatter allows only `Read, Edit, Grep, Glob` — Jira mode's connector access is not actually authorized by the command file. Phase 1 must fix the frontmatter/tooling before anything else assumes a working Jira path.
- **`/roadmap-publish`** gate (promote/drop/hold draft lanes) and the `no-draft-on-main` workflow. **Known gaps:** the workflow hardcodes `index.html` (config allows any `html_file`) and it is only fail-closed once set as a required branch-protection check. Phase 0/1 items.
- **SPEC.md**: the locked encoding — including the engagement-length geometry variation (same encoding, more columns; *not* a separate board type — do not build a second renderer).

v2 layers onto this. The base hand-authored board is untouched for PMs who don't opt in.

## 2. Part A — Completion & parity overlay (opt-in)

The base board shows *timeline + state*. The overlay adds *how complete each initiative is and who owns what* — the model proven on the pilot.

### 2.1 Data block

Embedded as `<script type="application/json" id="progress-data">…</script>`, placed immediately before the renderer script at the end of `<body>`. Must parse as strict JSON (no comments/trailing commas); escape `</` inside strings. Top level:

```json
{
  "asof": "YYYY-MM-DD",
  "groupOverrides": {},
  "initiatives": {
    "PROJ-123": {
      "anatta": { "pct": 77, "basis": "10 of 13 stories closed", "src": "jira" },
      "weight": 60,
      "items": [
        { "name": "Client QA pass", "ticket": "PROJ-123", "owner": "client",
          "state": "in_progress", "size": 2, "note": "since Jul 3" },
        { "name": "Content sign-off", "owner": "client", "state": "not_started", "size": 1 }
      ]
    },
    "PROJ-45": { "shipped": "foundation shipped April" }
  }
}
```

Field contract: `anatta.pct` 0–100 int; `anatta.basis` = human-readable receipt (required); `anatta.src` = `"jira"` (derived, Part B) or `"pm"` (human estimate — renderer must visibly label it "PM estimate"); optional `anatta.held: true` = the gate declined to refresh this value — set on ANY hold (underived, implausible delta); the renderer marks chip and meter unverified and appends "— held" to the basis, while the changelog carries the specific reason; `weight` = Anatta's share of the joint blend, int 0–100, default 60; `items[]` = the open-items receipts list (NOT the source of `anatta.pct`), each with `owner` ∈ `anatta|client`, `state` ∈ `not_started|in_progress|done` (the 3-state rubric — clients report coarse status), `size` = relative weight (positive int), optional `ticket`, `note`, `blocked: true`. `shipped` entries render as done with the given caption and no meters.

### 2.2 Calculation contract (normative — the build must not reinvent this)

- **Client %** = size-weighted mean over `items` with `owner:"client"`, mapping state `not_started→0`, `in_progress→50`, `done→100`. No client items → client meter omitted.
- **Joint %** = `round(weight/100 × anatta.pct + (100−weight)/100 × client%)`, half-up. Client % enters the blend unrounded; round once, at the end. No client items → joint = `anatta.pct`.
- **Chip** = `{joint}% · with {anatta|client|both}` — the "with" party = whoever is still open: the client side is open while any `owner:"client"` item is non-done; the Anatta side is open while `anatta.pct < 100` **or** any `owner:"anatta"` item is non-done (both open → "both"). Neither side open (everything done but not yet marked `shipped`) → the chip shows just `{joint}%`, no "with" phrase. `shipped` rows render a muted `100%`. Any item `blocked:true` → chip phrase renders in the signal treatment and the item shows its neutral `note` (SPEC §6 wording rules apply — status, never a story about a person).
- **Meters/basis**: Anatta meter shows `anatta.basis` verbatim; client meter's basis is auto-derived ("2 items: 1 in progress · 1 not started (client-reported)"); joint basis = "weighted blend {weight}/{100−weight}".
- **Lane rollup** = unweighted mean of member joint %s. A lane with zero overlay members gets **no rollup** — never fabricate. A PM may override per lane via `groupOverrides`, keyed by the **lane color class** (stable, unlike visible lane names), value `{ "pct": 0–100, "basis": "why" }` — both required; an override replaces the computed rollup and renders its basis. `shipped` entries are excluded from rollups — a lane of old wins shouldn't read 100% forever.
- **Client-state changes are propose-then-approve, never automatic.** Automation may *propose* an item-state flip only with an evidence quote (transcript/channel) shown to the PM; the PM approves; the verbatim quote stays in the internal changelog, and the `note` carries a neutral, client-safe paraphrase with the date ("client confirmed Jul 3"). This is a hard rule: the client meter is client-reported truth, not inferred.

### 2.3 Renderer & SPEC amendment (draft §8 — to be finalized when Part A ships)

- Renderer = one JS block directly after the JSON block. Progressive enhancement: **with JS disabled the base board renders unchanged** (chips/drawers/rollups simply absent).
- DOM surface: chips append inside `.row-label`; the drawer is a single overlay element; pinned on-bar % spans clamp to the visible bar portion. Rows opt in via a `data-initiative="KEY"` attribute — the stable hook joining HTML rows to JSON keys (no text matching).
- CSS additions live in one clearly-marked, namespaced block (`.pv-*`); the existing locked styles are untouched.
- Renderer security: every JSON string enters the DOM via `textContent` (never `innerHTML`); ticket keys are validated against `^[A-Z][A-Z0-9]*-\d+$` before any URL is constructed; drawer links carry `rel="noopener"`.
- Update commands treat the JSON block as data: surgical JSON edits are in-scope for `/roadmap-update`; the renderer and CSS are as locked as the rest of the page. `/roadmap-publish` becomes overlay-aware: mechanically it re-checks draft lanes, validates ticket-key/marker rules, and warns when `asof` is stale (default > 7 days; configurable down for daily-cadence clients) — and it prints every overlay text string (basis lines, item names, notes) for the human SPEC §6 review, which remains the actual enforcement of client-safe wording.
- Drawer ticket links are **off by default** (`overlay.drawer_links: "none" | "jira"`): clients often have no access to our Jira, and links must only ever point at `visible_marker`-labeled tickets. Item-level visibility follows the same fail-closed rule as rows.

**Config:** `overlay.enabled: true|false` (default false). Existing boards without the flag are untouched.

## 3. Part B — Jira % refresh + data-quality gate

Extends Jira mode to derive `anatta.pct` from child-story counts per initiative and refresh the JSON block — with a gate that **refuses to ship a number it can't trust.**

### 3.1 Identity model (replaces any looser mapping idea)

`jira.lanes` keeps its existing job (swimlane membership). Progress derivation gets its own table — one entry per overlay initiative:

```json
"jira": {
  "progress": [
    { "key": "PROJ-123", "level": "epic" },
    { "key": "PROJ-90",  "level": "initiative" }
  ]
}
```

`key` = the JSON `initiatives` key = the Jira rollup root. `level` disambiguates counting: `epic` counts child stories; `initiative` counts stories under child epics (never "N sub-epics closed" — the pilot showed those counts go wrong). Unmapped initiatives keep `src:"pm"` and are listed in every changelog as unmapped.

### 3.2 Gate semantics (normative — per failure class, per initiative)

The gate is **per-initiative**: one bad initiative never blocks the others. Only invalid JSON / failed sentinels abort the whole run. `/roadmap-update` never publishes — "fail closed" here means *the number is not written*, and the block is reported in the changelog.

| Condition detected | Action |
|---|---|
| Rollup root has **0 child stories** | No derived % — never fabricate. Existing entry: keep the old value and set `"held": true` (renderer shows the unverified marker; basis gains "— held"; changelog says why). New entry: require `src:"pm"` + basis. |
| All children done but root not transitioned | Derive from stories (stories win); flag the root for hygiene in the changelog. |
| `level` mismatch (children are epics, not stories) | Block that initiative's write; tell the PM to fix `level`. |
| Orphan stories (no parent) among `visible_marker`-eligible issues | Never counted; report the total in the changelog as a coverage warning. Project-wide orphan counts are internal diagnostics only — they don't belong in a client repo's changelog. |
| A story counted under two roots | Block both initiatives' writes; ask the PM. |
| **Implausible delta** — \|new − last\| > 30 pts | Hold that initiative: the new number is **not written**, its baseline does not advance, and the kept value gets `held: true`. The changelog shows old → proposed; only an explicit PM confirmation writes the value (and only then does `.roadmap/sync-state.json` advance). The rest of the run proceeds. |

Last-run state for the delta check persists in `.roadmap/sync-state.json` (per-initiative `{pct, done, total, ts}`), committed on the review branch and excluded from deploys via `.vercelignore`. It may contain **only** data already visible on the deployed board (initiative keys, counts, dates — exactly what the basis lines show), because the Git side of these repos can be public; nothing more sensitive ever goes in it. Baselines advance only when a value is actually written. First run (no prior state): all values write, and the changelog leads with "first sync — no baseline, verify manually."

A clean structural pass is **necessary but not sufficient** — the changelog must always show old → new per initiative so the PM sanity-checks the numbers before publishing.

**Prerequisite for any unattended run:** Jira access via a **dedicated service account with read-only permissions** (a raw API token is not inherently read-only — scope lives on the account). Config carries `jira.api_token_ref` = the *name* of an environment variable the runner resolves; the secret never appears in any repo file.

## 4. Part C — Deep multi-source audit

`/roadmap-audit` (repo-local, like every other command; optional `--sources` filter). For high-stakes moments. A first runnable version ships alongside this spec (`.claude/commands/roadmap-audit.md`) — the same procedure, executable interactively in a connector-equipped session today; the build session hardens it. Cross-checks the board against: Jira (3.1 model) + the newest transcript in `transcripts_dir` + the client's own tracker + Confluence/Slack, using per-source adapters configured in:

```json
"deep_sources": {
  "tracker_file": "path-or-none",
  "confluence_space": "KEY-or-none",
  "slack_channels": []
}
```

Sources run independently (parallel where the platform supports subagents); an unavailable source is **skipped with an explicit "source unavailable" line** — never silently. Precedence: the client's own tracker wins on client-side status; Jira wins on our side; the transcript wins on wording/why; Confluence is checked for *staleness against the board*, not treated as truth.

Output: a reconciliation report — per initiative: MATCHES / STALE-DRIFTED / UNDERSTATED / OVERSTATED / FALSE / CONFLICT / UNVERIFIABLE, ranked by client-facing impact, with the data-quality evidence — written into a dated subfolder of `audit_output_dir` (default `../<repo-name>-audits/`, i.e. **outside the repo** — the command canonicalizes the path, refuses any location at or under the repo root, and creates `<date>/` inside it). Audit output is never committed: it contains internal signal by construction, and anything committed here can end up on the public deploy (§0).

## 5. Part D — Client-safe vs. internal split

The audit report has two hard-separated sections: **client-safe corrections** (eligible for the board) and **internal signal** (risk flags, relationship/ownership observations, anything §6 forbids). Internal signal routes to the PM's internal reporting artifact (e.g. the exec summary — external to this repo; routing is manual until that automation exists). The split is supported by the report format and by the publish gate surfacing every overlay string for review — the SPEC §6 judgment itself remains human.

## 6. Part E — App packaging (rinse-and-repeat)

Direction from leadership: this becomes an internal **product**, not a repo each PM clones. Target shape, phased:

- **New-client generator** — one step from client name → scaffolded board + config + hosting (grows out of `/roadmap-new`).
- **Board registry / portfolio dashboard** — every client board in one place: owner, `asof` freshness, last update, publish state. This is also the leadership portfolio view.
- **Org hosting + access control** — org-owned hosting with real tiers (client sees only their board; leadership sees the portfolio; PMs edit). **This is the major engineering build of Part E — auth and tenancy are not a thin wrapper — and it is the first funded item.** Today's public-URL model is the interim.
- **Hosted refresh runner** — Part B headless on a schedule (service account, env-resolved secrets). **Scheduled runs write only to the review branch; a human always publishes.** The approval boundary survives automation.

Everything except hosting/auth is a thin layer over the existing repo model; the locked HTML encoding stays the render target.

## 7. Config additions (v2, all optional)

```json
"overlay": { "enabled": false, "drawer_links": "none" },
"jira":    { "progress": [], "api_token_ref": "ENV_VAR_NAME" },
"deep_sources": { "tracker_file": null, "confluence_space": null, "slack_channels": [] },
"audit_output_dir": "../<repo-name>-audits/"
```

No existing key changes meaning; boards without these keys behave exactly as today. (The example config ships `api_token_ref: null` — unset until Part B wires the runner; the command appends the dated subfolder to `audit_output_dir` itself.)

## 8. Build order

0. **Repo hygiene (shipped with this commit):** `.vercelignore`. **Remaining phase-0:** parametrize `no-draft-on-main` to the config's `html_file`; make sure branch protection is actually configured per NEW_CLIENT.md §5a (the docs exist — enforcement is the gap); fix `/roadmap-update` frontmatter so Jira mode is actually authorized.
1. **Data-quality gate + sync-state** (3.2) — highest safety value. Note: full value arrives with % derivation (phase 3); at phase 1 it already gates status-mapping (orphans, untransitioned roots).
2. **Completion overlay + SPEC §8** (2.1–2.3).
3. **Story-count % refresh** feeding the overlay (3.1).
4. **Deep audit** (Part C).
5. **Headless runner** (service account; review-branch-only writes).
6. **App packaging** (Part E) — registry/generator early groundwork is cheap; hosting/auth is the funded build.

## 9. Guardrails carried forward

SPEC §6 applies to every mode and every new text surface (overlay basis/items/notes, draft-lane contents, audit output, anything the app serves) — and, per §0, to every file committed to these repos. Fail closed on client exposure, always.

---

## Appendix A — Proposed SPEC §8 (drop-in draft, finalized when Part A ships)

> ### 8. Completion overlay (opt-in)
>
> A board MAY carry a completion overlay. Boards without it are unaffected; everything in this section is additive and reversible.
>
> **Data.** One `<script type="application/json" id="progress-data">` block, placed immediately before the renderer script, which is the last element in `<body>`. Strict JSON (no comments, no trailing commas; `</` escaped inside strings). Schema, calculation rules, and error behavior: currently DESIGN v2 §2.1–2.2 — **when §8 is finalized, that text physically moves into SPEC with its cross-references rewritten** (SPEC becomes the sole source of truth; the design doc then points here). Companion edits at finalization: the SPEC preamble's "the only JS…" sentence and the §1 anatomy row gain the renderer; the propose-then-approve workflow rule lands in SPEC §7, not §8.
>
> **Row hook.** A row participates by carrying `data-initiative="KEY"` on its `.row` element, where `KEY` is the row's key in `initiatives`. No text matching, ever. **Exactly one row per key:** duplicate hooks → renderer console warning, only the first hooked row renders a chip, and the initiative counts once in rollups. Unmatched JSON keys → renderer console warning; `/roadmap-update` flags them in its changelog. Rows without the attribute are untouched.
>
> **Renderer.** One JS block directly after the JSON block. Progressive enhancement: with JS disabled the base board renders exactly as §1–7 describe. The renderer may only: append one chip element inside a hooked row's `.row-label`, maintain one drawer overlay element, and add a pinned % span to the hooked row's single epic/initiative-level bar — a hooked row with zero or multiple eligible bars gets no on-bar % (console warning), never opportunistic decoration of every bar. A `held` value renders visibly unverified: marker on chip and meter, basis suffixed "— held", and "unverified" in the accessible label. All strings enter the DOM via `textContent`; `innerHTML` is forbidden; ticket keys must match `^[A-Z][A-Z0-9]*-\d+$` before URL construction; links carry `rel="noopener"`.
>
> **CSS.** Overlay styles live in one marked block (`/* pv: completion overlay */`), all selectors `.pv-*`. Locked base styles are untouched.
>
> **Editing.** Content updates edit the JSON block only — the renderer and CSS are locked. `/roadmap-update` may rewrite `initiatives` values surgically; `/roadmap-publish` re-checks draft lanes, validates ticket-key/marker rules, warns on stale `asof`, and prints every overlay string for the human §6 review.
>
> **Print.** The drawer never prints; chips and pinned %s print as rendered.

## Appendix B — Golden worked example (build target; values are normative)

Input (placeholder client):

```json
{
  "asof": "2026-07-17",
  "groupOverrides": { "compliance": { "pct": 70, "basis": "external counsel estimate" } },
  "initiatives": {
    "PROJ-201": { "anatta": { "pct": 80, "basis": "16 of 20 stories closed", "src": "jira" }, "weight": 60,
      "items": [
        { "name": "Template QA", "ticket": "PROJ-201", "owner": "client", "state": "in_progress", "size": 2 },
        { "name": "Content entry", "owner": "client", "state": "not_started", "size": 1 },
        { "name": "Banner build", "owner": "anatta", "state": "in_progress", "size": 1 } ] },
    "PROJ-202": { "anatta": { "pct": 45, "basis": "9 of 20 stories closed", "src": "jira", "held": true }, "weight": 60,
      "items": [ { "name": "API specs", "owner": "client", "state": "in_progress", "size": 2, "blocked": true, "note": "pending sign-off" } ] },
    "PROJ-203": { "anatta": { "pct": 70, "basis": "PM estimate — no epic mapping", "src": "pm" }, "weight": 50, "items": [] },
    "PROJ-204": { "anatta": { "pct": 100, "basis": "12 of 12 stories closed", "src": "jira" }, "weight": 60, "items": [] },
    "PROJ-205": { "shipped": "foundation shipped April" }
  }
}
```

Required outputs (computed per §2.2 — a conforming build MUST reproduce these exactly):

| Key | client % | joint % | Chip | Notes |
|---|---|---|---|---|
| PROJ-201 | (2×50+1×0)/3 = 33.33 (unrounded into blend) | 0.6×80 + 0.4×33.33 = 61.33 → **61** | `61% · with both` | client open (2 non-done) AND anatta open (pct<100 + open item). Client basis: "2 items: 1 in progress · 1 not started (client-reported)" |
| PROJ-202 | 50 | 0.6×45 + 0.4×50 = **47** | `47% · with both`, signal treatment | `held` → unverified marker on the Anatta meter, basis gains "— held"; blocked item shows its neutral note |
| PROJ-203 | — (no client items) | = anatta.pct = **70** | `70% · with anatta` | client meter omitted; basis visibly labeled "PM estimate" |
| PROJ-204 | — | **100** | `100%` (no "with") | neither side open |
| PROJ-205 | — | — | muted `100%` | caption "foundation shipped April"; **excluded from lane rollups** |

Lane rollups: a lane containing PROJ-201 + PROJ-205 rolls up to **61** (shipped excluded). The `compliance` lane renders **70** with basis "external counsel estimate" (override replaces computation). A lane whose only members are shipped entries shows **no rollup**.

## Appendix C — Acceptance criteria per phase (the build session's verify suite)

- **Phase 0:** `no-draft-on-main` reads `html_file` from config (fixture with a renamed board file still fails on a draft lane); `/roadmap-update` invoked in a connector-equipped session can actually reach Jira (frontmatter no longer blocks it). Branch protection is repo configuration, not testable by fixture — it goes on a manual deployment checklist, checked off per repo.
- **Phase 1 (gate):** six fixtures — one per §3.2 failure class — each produces exactly the specified action and changelog line; invalid JSON aborts before any write; every changed number appears as old → new in the changelog; first-run fixture writes all values with the "first sync" banner.
- **Phase 2 (overlay):** Appendix B reproduces exactly (values, chips, markers, rollups); additional calculation fixtures required beyond Appendix B — omitted `weight` (default 60), a true `.5` result proving half-up, client-all-done while Anatta open, Anatta-100 while client open, invalid/zero item sizes rejected, duplicate row hooks and unmatched keys handled per §8; JS disabled → the base board is DOM-equivalent to a no-overlay board (overlay-generated elements simply absent); publish gate prints all overlay strings and fires the stale-`asof` warning on a fixture dated 8 days back.
- **Phase 3 (refresh):** against a recorded Jira fixture, writes expected `pct`/`basis`, advances sync-state only for written values, holds a >30-pt delta.
- **Phase 4 (audit):** produces the two-section report outside the repo; refuses an `audit_output_dir` inside the repo (canonicalized); a missing source yields an explicit "source unavailable" line, not silence.
- **Phase 5 (runner):** scheduled run writes only the review branch (no path to `main` exists in its credentials/config); failure notifies rather than retries silently.
- **Phase 6 (app):** registry lists every board with `asof` freshness; a client-tier link cannot reach the portfolio view or another client's board.
