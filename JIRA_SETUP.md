# Connecting your Jira board to the roadmap

The roadmap can be updated straight from your Jira project: Claude reads your tickets, compares them to the board, and proposes the changes (`/roadmap-update` — see README). For that to work, Claude needs to be able to tell three things about every ticket. Two of them Anatta's Jira already does for you. One of them you have to do.

## The three things, and who does them

| Question about a ticket | How it's answered | Who sets it up |
|---|---|---|
| What state is it in? (to do / in progress / done) | Your ticket **status** — Anatta's statuses are the same in every project (Dev To Do, Dev In Progress, Dev Done, …) | ✅ Already done org-wide |
| What kind of work is it? (build / test / bug / investigation) | Your **issue type** (Dev Story, Bug, Discovery Story, …) | ✅ Already done org-wide |
| Should the client see it, and in which lane? | A **label** + an **epic** (below) | 🫵 You, once, ~15 minutes |

## Your one-time setup (~15 minutes)

### 1. Label what the client should see

Add the label **`roadmap`** to every ticket that belongs on the client's board.

- No label = the ticket **never** appears on the roadmap. This is on purpose (fail-closed): internal tickets, design subtasks, monitoring chores, and half-written ideas stay invisible by default.
- When you create a new client-visible ticket later, add the label then. That's the whole ongoing habit.

### 2. Give every lane an epic

Each swimlane on your roadmap (e.g. "Loyalty Program", "PDP Redesign") should be one **epic** in Jira, and every `roadmap`-labeled ticket should have one of those epics as its parent.

- If your project already uses epics this way (like LN does), you're done.
- If your tickets are loose (no parents), spend the 15 minutes: create one epic per lane, drag your roadmap tickets under them. This is good Jira practice anyway.

### 3. Fill in the `jira` section of your config

In your client repo's `roadmap.config.json`:

```json
"jira": {
  "site": "anatta-io.atlassian.net",
  "project": "PROJ",
  "visible_marker": "label:roadmap",
  "lanes": {
    "cart":            "PROJ-53",
    "plp":             "PROJ-59",
    "experiments":     "PROJ-61",
    "implementations": "PROJ-101"
  }
}
```

`lanes` maps each lane color class (left side — the six fixed classes from SPEC.md) to the epic that feeds it (right side — your epic's ticket key). Lanes you don't use, leave out.

## What Claude does with it (the default mapping)

You don't configure any of this — it's the org-wide default, listed here so you know what to expect:

| In Jira | On the roadmap |
|---|---|
| Status: Dev Done, Design Done, Closed, Done | `done` bar (gray, "Shipped") |
| Status: Dev In Progress, QA In Progress, Ready for QA | solid in-flight bar |
| Status: Dev To Do, Open — with a sprint/dates known | solid bar in that sprint |
| Status: Dev To Do, Open — no schedule known | `proposed` bar (dashed) |
| Issue type: Bug | `BUG` pill |
| Issue type: Discovery Story, or `[Spike]` in the title | `SPIKE` pill |
| `[Test]` in the title, or parent epic is the experiments lane | `TEST` pill |
| Everything else (Dev Story, Task) | `IMPL` pill |
| Issue type: Design Story, Sub-task | skipped (internal by default; label it `roadmap` to force-include) |

Two things Jira alone can never answer, so Claude will ask you (or read your meeting notes):

- **Timing** — which sprint columns a bar should span, when Jira has no sprint/due date on the ticket.
- **Wording** — ticket titles are written for the team, not the client. Every title gets rewritten into client-safe language before it lands on the board (the guardrails in SPEC.md §6 apply to Jira imports exactly as they do to transcripts). Example: "[Bug] PDP Timeline Copy: Wrong Claims on 9 Products" becomes "PDP timeline copy fix" — same work, client-appropriate words.

## The habit, ongoing

1. Work your Jira board like you always do.
2. Keep the `roadmap` label on client-visible tickets and parent them to a lane epic.
3. When it's time to update the board: run `/roadmap-update`, review what Claude proposes on the `draft` preview, merge.
