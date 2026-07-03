# Anatta Client Roadmap

**What this is, in one sentence:** a way to give every Anatta client a beautiful webpage that shows what we shipped, what we're working on, and what's coming next — and it looks the same for every client, no matter which PM runs the account.

**See it before you read anything else:** open the live demo — **https://roadmap-alpha-three.vercel.app/** — a finished example for a made-up client called "Example Co." That page IS the product. Your job is to make one of those for your client.

---

## The big picture

Here is the whole workflow, start to finish:

```
1. COPY this repo               →  you get a starter roadmap file
2. FILL IN your client's info   →  names, dates, their actual work
3. PUT IT ON VERCEL             →  now it's a webpage with a link
4. SEND THE LINK to your client →  they can check it anytime
5. AFTER EVERY MEETING, update the page  →  it stays true forever
```

That's it. Everything else in this repo exists to make those five steps easy and to make sure your roadmap looks exactly like every other Anatta roadmap.

## Why we do it this way

- **Clients love it.** They get one link that always shows the truth: what shipped, what slipped, what's next. No more "can you send me an update?"
- **It's the same everywhere.** A client on any account sees the same style of page. A PM covering for another PM can read any roadmap instantly.
- **It's safe.** The format has strict rules about what never goes on the page (see "The rules" below), so nothing embarrassing ever reaches a client.
- **It can update itself later.** Because every roadmap has the exact same structure, we can plug in automation that turns meeting transcripts into roadmap updates for you. That only works if nobody invents their own format.

## Step-by-step: making your first roadmap

You need: a GitHub account, a Vercel account (free), and about 30 minutes. If you use Claude Code, it can do most of this for you — see "The easy way with Claude" below.

1. **Copy this repo.** Make a new repo for your client (call it something like `acme-roadmap`) and copy these files into it. `template.html` is your starting file — rename your copy to `index.html`.
2. **Open your `index.html` and search for the word `SETUP`.** Every place that word appears is a spot you need to change: your client's name, this quarter's dates, your sprint numbers, your Jira project. The comments tell you exactly what to put where.
3. **Replace the example work items with your client's real work.** The file contains one example of every kind of bar (finished work, in-progress work, tests, bugs, proposed ideas, parked ideas). Copy the kind you need, delete the ones you don't. Full instructions live in `NEW_CLIENT.md`.
4. **Make a branch called `draft`.** This matters: `main` is what the client sees, `draft` is where you make changes. You always edit `draft`, look at it, and only then merge into `main`. Think of `main` as "published" and `draft` as "my desk."
5. **Connect the repo to Vercel.** In Vercel: New Project → pick your repo → framework "Other" → no build settings → Deploy. Vercel gives you a link. The `main` branch is that link; the `draft` branch automatically gets its own preview link for checking your work.
6. **Send the `main` link to your client.** Done.

## What you do every week after that

After each client meeting (or whenever something changes):

1. Edit `index.html` **on the `draft` branch**: move a bar, mark something shipped, add a new row.
2. Look at the draft preview link. Does it look right? Is anything on there that shouldn't be? (see the rules)
3. Merge `draft` into `main`. The client's link updates by itself within a minute.

Once a quarter, you "roll" the board to the new quarter — new dates, new columns, unfinished work carried over. That takes about 30 minutes and the instructions are in `SPEC.md` (section 2).

## The easy way with Claude

If you use Claude Code, this repo includes two commands that do the heavy lifting:

- **`/roadmap-new`** — Claude interviews you (client name, quarter, sprint numbers, what work exists) and builds the whole starting file for you.
- **`/roadmap-update`** — you give Claude your meeting transcript or notes, and it updates the roadmap for you: moves the bars, updates the labels, and writes you a summary of what it changed. It knows all the format rules and all the safety rules.

Either way, YOU still review the draft before merging. Claude prepares; you approve.

## The rules (the part you actually have to memorize)

The roadmap is a **client-facing page**. The client reads it. Their boss reads it. So:

1. **No people's names. Ever.** Not yours, not theirs, not a developer's. "Pending client sign-off" — never "Waiting on Brad." This includes the little tooltip text on the bars.
2. **No internal talk.** Nothing about hours, budget, who's slow, renewal odds, team frustrations, or anything you wouldn't say in front of the client. If you're not sure whether something is safe to put on the page, **leave it off**. Losing a minor detail is always cheaper than an awkward client call.
3. **Don't touch the styling.** The colors, fonts, and layout are locked so every Anatta roadmap matches. You edit the content (bars, labels, dates) — never the CSS.
4. **Edit `draft`, publish `main`.** Never edit `main` directly. The review step is where mistakes get caught.

## What's in this repo

| File | What it is, plainly |
|---|---|
| `index.html` | The finished demo for fake client "Example Co." — look at this to understand what you're building. This is what Vercel shows when you deploy this repo. |
| `template.html` | Your starting file. Copy it, rename it `index.html` in your client's repo, and fill in the `SETUP` spots. |
| `NEW_CLIENT.md` | The step-by-step checklist for setting up a new client (the long version of the steps above). |
| `SPEC.md` | The rulebook: what every color, bar, and label means, and the do-not-cross safety rules. Read it once; look things up later. |
| `roadmap.config.example.json` | A small settings file the Claude commands read (client name, Jira project, etc.). Copy it, fill it in, commit it. |
| `.claude/commands/` | The two Claude commands (`/roadmap-new`, `/roadmap-update`) described above. They come along automatically when you clone the repo. |

## Stuck?

Read `NEW_CLIENT.md` first — it answers most setup questions. For "what does this bar/color/label mean," it's `SPEC.md`. For everything else, ask Justin.
