---
description: Resolve draft lanes and gate this client's roadmap for publishing (review branch → main). Usage: /roadmap-publish
allowed-tools: Read, Edit, Grep, Glob, Bash
---

You are the PM about to publish this roadmap to the client. Your job is to make the board client-clean: every `draft` lane (internal-only staging, SPEC §4) must be **promoted** or **dropped** before the board can merge the review branch → `main`. Nothing internal reaches the client-facing deploy.

## Inputs

- **Client config:** read `roadmap.config.json` in the repo root (`html_file`, `review_branch`, `live_url`). If it's missing, stop and say so — don't guess paths.

## What a draft lane is

A whole swimlane staged internally: `<details class="group draft" open>` (SPEC §4). It renders with a `DRAFT · INTERNAL` badge and shows on the review-branch preview only. It **must never reach `main`** — that leaks internal/unconfirmed work to the client. This command is the friendly pre-merge gate; the `no-draft-on-main` GitHub Action is the hard backstop.

## Procedure

1. **Confirm you're on the review branch** (`git rev-parse --abbrev-ref HEAD` vs config's `review_branch`). If not, stop — publishing edits happen there, not on `main`.
2. Read the roadmap HTML (config's `html_file`). Find every draft lane: `grep -nE '<details[^>]*class="[^"]*\bdraft\b[^"]*"' <html_file>`. Report the count.
3. **If zero draft lanes:** the board is already client-clean. Say so, note it's safe to merge review branch → `main`, and skip to Output.
4. **If any:** for each draft lane, show the lane name + a one-line summary of its contents, then decide with the human — **promote**, **drop**, or **hold**:
   - **Promote** (confirmed with the client): remove `draft` from the wrapper (`class="group draft"` → `class="group"`) and strip any internal wording from the header. Then **re-check the lane against SPEC §6 guardrails** — no named individuals, no internal talk, neutral outcome-focused wording — and confirm every bar carries a real state (in-flight / `proposed` / `done` / `deferred`), not a placeholder. Fail closed: fix or omit anything ambiguous before promoting.
   - **Drop** (not happening / not ready): delete the entire `<details class="group draft"> … </details>` block, rows included.
   - **Hold** (keep staging, not publishing this cycle): leave it. **Holding blocks the publish** — see step 5.
5. Re-scan for draft lanes.
   - **None remain:** the board is client-clean and safe to merge. Proceed to Output.
   - **Any remain (held):** **stop — the board cannot be published this cycle.** A `draft` lane on `main` is a leak and the `no-draft-on-main` Action will fail the merge. Tell the human plainly: publish is blocked until held lane(s) are promoted or dropped. (Publishing merges the whole review branch, so a held lane defers the entire publish. A lane meant to persist internally across many client publishes is a separate long-lived-branch pattern, not this flow.)
6. Do **not** merge, commit, or push. Merging to `main` publishes to the client — a hard-to-reverse, outward-facing action. Prepare and gate the board, then hand the merge back to the human.

## Output

End with a concise **changelog** grouped **Promoted / Dropped / Held**, one line each (lane name + why). Then a one-line verdict: either **"Client-clean — safe to merge → `main`."** or **"Publish blocked — N draft lane(s) still held."**
