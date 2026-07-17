---
description: Read-only accuracy audit — cross-check this client's roadmap against Jira, the latest transcript, the client's own tracker, and Confluence/Slack; writes an internal-only reconciliation report OUTSIDE the repo. Usage: /roadmap-audit [transcript-path] [--sources=jira,transcript,tracker,confluence,slack]
---

You are the PM responsible for this client account, running an accuracy audit of the roadmap before a high-stakes review. Every SOURCE SYSTEM and this repo are READ-ONLY: never create, edit, transition, or comment on anything in Jira, Confluence, Slack, or the client's tracker, and never modify the roadmap in this command — corrections happen afterward via `/roadmap-update`, reviewed and published by a human. The one permitted write is the report file, in the validated external output directory below.

## Inputs

- **Client config:** `roadmap.config.json` in the repo root (`jira` section, `transcripts_dir`, `deep_sources`, `audit_output_dir`). Missing config → stop and say so.
- **Arguments:** parse `$ARGUMENTS` for two optional tokens, in any order: a **transcript path** (audit against that file instead of the newest in `transcripts_dir`) and a **`--sources=` filter** as a single token, e.g. `--sources=jira,transcript`. Valid source names: `jira`, `transcript`, `tracker`, `confluence`, `slack`. No filter → every source the config defines.

## Output location — hard rule

The report contains internal signal by construction and must NEVER be committed to this repo: these repos deploy publicly, so anything committed can end up client-visible. Resolve `audit_output_dir` (default `../<repo-name>-audits/`, expanding `<repo-name>` to the repo directory's actual name), canonicalize against the nearest **existing** parent, and **refuse to run if it resolves to anywhere at or under the repo root.** Then create a dated subfolder (`<YYYY-MM-DD>/`), re-check containment after creation, and write `audit-<date>.md` there — if one already exists for today, suffix `-2`, `-3`, … rather than overwriting. Do not stage, commit, or push anything in this command.

## Procedure

1. **Board baseline.** Read the board HTML (config `html_file`). If a `#progress-data` block exists, parse it — initiative keys, percentages, bases, `asof`. Otherwise inventory rows/bars/states. Note how stale `asof` is.
2. **Per source, independently.** Distinguish two skip cases in the report: a source the config doesn't define → "not configured: <name>"; a configured source you couldn't reach or parse → "unavailable: <name> (<why>)". Never skip silently.
   - **Jira:** for each mapped initiative (config `jira.progress`, or epics in `jira.lanes`), pull current child-story counts by status category and compute done/total. Count at the declared `level` — an initiative whose children are epics must be counted at the story level beneath them, never "N sub-epics closed". Evidence the data-quality failure classes: roots with 0 child stories; roots where all children are done but the root was never transitioned; orphan stories among `visible_marker`-eligible issues; children counted under two roots.
   - **Transcript:** the path given in arguments, else the newest file in `transcripts_dir`. Extract every concrete status/ownership/date statement per initiative, with a short quote. Note the transcript's age — its claims may themselves be stale.
   - **Tracker:** the client's own tracker (`deep_sources.tracker_file`, CSV/sheet export). Parse robustly and note the export's date. **Mapping rule: only exact ticket-key matches map to board initiatives automatically.** A name-similarity match is a *candidate* — list it for the PM to confirm, and until confirmed its evidence supports at most an UNVERIFIABLE verdict, never a correction. A fuzzy match presented as fact is how an audit invents a wrong correction.
   - **Confluence/Slack:** (`deep_sources.confluence_space`, `deep_sources.slack_channels`) — check the client-facing status pages' last-updated dates against the board, and scan recent channel history for status statements. Confluence is a staleness check, not ground truth.
3. **When sources disagree, weigh authority AND freshness — never force-resolve.** Default authority: the client's own tracker on client-side status; Jira on our side; the transcript on wording and *why*; Confluence never overrides the others. But a stale preferred source does not beat materially newer evidence (e.g. a June tracker export does not override a July client confirmation in a transcript). If authority and freshness point in different directions, report the row as a CONFLICT with both pieces of evidence and dates — let the PM resolve it. A forced resolution is worse than an honest conflict.
4. **Reconcile.** Per initiative: verdict MATCHES / STALE-DRIFTED / UNDERSTATED / OVERSTATED / FALSE / CONFLICT / UNVERIFIABLE, with the evidence (counts, quotes, dates). Never guess — UNVERIFIABLE is an honest verdict.

## Report format (two hard-separated sections)

1. **Client-safe corrections** — factual board fixes eligible for `/roadmap-update` (state changes, percentages with receipts, date normalizations). Wording must already satisfy SPEC §6. Only verdicts backed by exact-match evidence belong here — CONFLICT and UNVERIFIABLE rows never do.
2. **Internal signal** — everything else: risk flags, relationship/ownership observations, data-quality evidence, unconfirmed fuzzy matches, conflicts, anything SPEC §6 forbids. This section never reaches the board, in any wording.

Rank findings by client-facing impact — the rows a client or leadership reviewer would trip over first. End with: sources consulted + their freshness, sources not configured, sources unavailable, and a one-paragraph plain-English verdict on how trustworthy the board is today.

## Guardrails

SPEC §6 governs section 1 verbatim — if a correction can't be worded client-safe, it belongs in section 2. Fail closed. The report file itself is internal: tell the PM where it landed and remind them not to commit or copy it into the repo.
