# Roadmap Template

The Anatta client-facing roadmap gantt, packaged for reuse. One hand-authored static HTML file per client — no build step, no framework — deployed on Vercel with a `draft` branch as the review lane.

Reference implementation: the Llama Naturals roadmap (https://ln-roadmap-anatta.vercel.app/).

| File | What it is |
|---|---|
| `index.html` | **The live example** — fictional "Example Co." Q3 2026 retainer board (loyalty program, returns management, PDP redesign, experimentation…) showing every lane, state, pill, and milestone pattern; this is what Vercel serves when the repo is deployed |
| `template.html` | Working sanitized skeleton — every visual pattern, `SETUP:` comments where client specifics go; start from this file, not the example |
| `SPEC.md` | The locked encoding spec: geometry, bar/row/pill semantics, colors, writing rules, guardrails |
| `NEW_CLIENT.md` | ~30-minute checklist to stand up a new client roadmap |
| `roadmap.config.example.json` | Per-client config consumed by the Claude skills |
| `.claude/commands/roadmap-new.md` | Claude skill: instantiate a new roadmap interactively |
| `.claude/commands/roadmap-update.md` | Claude skill: update a roadmap from a meeting transcript, guardrails enforced |

Start with `NEW_CLIENT.md`. If you edit with Claude Code, the two skills do most of the work; either way, `SPEC.md` §6 (information-exposure guardrails) is non-negotiable — the roadmap is client-facing.
