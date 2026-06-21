---
name: tasks
description: Day-to-day task management for an Obsidian vault — process the daily-note inbox, sweep open tasks forward into today's note, group by horizon, track delegation, and flag drift. Manual morning command; pure markdown, no plugins.
disable-model-invocation: true
---

# tasks

Day-to-day task management for this vault. **You capture messily; this skill
files, links, schedules, and reconciles.** After a morning sweep, **today's
daily note is the single source of truth for all open work** — one file answers
"what's open."

Data lives in the vault at `./diary/YYYY-MM-DD.md` (pure markdown). This skill
carries **zero personal data**: the per-vault taxonomy (areas, owners, horizons,
drift thresholds) lives in `./diary/_tasks-config.md`, which this skill reads on
every run and **scaffolds if absent** (see "First run" below).

## Non-negotiable rules

1. **Wiki link is one-way.** Tasks may reference `[[Wiki pages]]`. This skill
   **never creates, edits, moves, or deletes anything under `Wiki/`.** Obsidian's
   backlinks pane is the Wiki-side view.
2. **Pure markdown, no plugins.** Only plain `- [ ]` checkboxes, plain `#tags`,
   and a human-readable `(created: YYYY-MM-DD)`. Never depend on the Tasks or
   Dataview plugins.
3. **Exactly one live copy of every open task.** The sweep *moves* open tasks
   into today's note and **deletes them from the source note** — no tombstone.
4. **Completed tasks complete in-place and never move.** A checked `- [x]` stays
   under whatever horizon section it was in when completed; the sweep skips it and
   leaves it there. Past notes become the done-journal, completions sitting beside
   the horizon they were finished under.
5. **Horizons never auto-change.** Flag drift; the user re-tags manually. Re-tag
   **only on explicit instruction** — never silent demotion.
6. **Dates are absolute** `YYYY-MM-DD`, never relative.

## Task line format

```
- [ ] <text> (created: YYYY-MM-DD) #area/<area> #<horizon> @<owner>
```

- `@me` is the implicit default owner and may be omitted on your own tasks.
- `<horizon>` is one rung of the ladder (below). `<area>` is an inline tag, not a
  grouping key — grouping is by horizon.
- `(created: ...)` is set once at capture and **preserved across every sweep**.

## Dimensions

- **Area** — `#area/<name>` (e.g. `#area/wiki`, `#area/work`). Roster in config.
- **Horizon ladder** (priority axis):
  `#today` → `#this-week` → `#this-sprint` → `#this-quarter` → `#longterm`.
- **Created date** — `(created: YYYY-MM-DD)`; powers staleness/drift.
- **Owner** — `@name`. Owner ≠ `@me` ⇒ delegated (tracked under "Waiting on").

## `/tasks` — the morning flow (no args)

1. **Process inbox.** Read the `## Inbox` of today's note and all prior daily notes
   that still have inbox content. For each line:
   - classify **area** (`#area/...`) from config; ask only if genuinely ambiguous,
   - detect/assign **owner** (`@name`); default `@me`,
   - infer **horizon**; **ask if ambiguous** rather than guessing,
   - match any `[[Wiki page]]` references and link them (one-way; do not touch
     the Wiki page),
   - stamp `(created: <today>)` if missing,
   - dedupe against existing open tasks.
   Then **empty the inbox**.
2. **Move-and-stamp sweep.** For every *open* `- [ ]` task in prior daily notes,
   **move it into today's note and delete it from the source note**, preserving
   its `(created: ...)`. Leave completed `- [x]` tasks where they are.
3. **Regroup** by horizon ladder under the matching `##` sections. Place every
   delegated task (owner ≠ `@me`) under `## Waiting on`, **grouped by `@owner`**.
   An opt-in middle state exists only on request: "mark delivered, pending my
   review" parks the task under a `### Delivered, pending review` line.
4. **Drift nag.** Build `## ⚠️ Drifted` from the config thresholds:
   - a horizon task whose age exceeds its bucket's threshold,
   - a delegated task idle past the delegated threshold (reframe: "delegated to
     @bob N days ago — check in?").
   **Flag only — never re-tag.** Offer keep / demote / drop per item.
   Horizons and delegated tasks without a configured threshold in `_tasks-config.md` are intentionally never drift-flagged.
5. **Summary.** Print counts per horizon, the Waiting-on roster, and the drift
   list for the user to act on.

## Optional verbs

- `/tasks capture <text>` — append `<text>` to today's `## Inbox` and stop
  (no full sweep). Same inbox the morning flow reads.
- `/tasks review @bob` — read out all open tasks for owner `@bob`.
- `/tasks review #area/wiki` — read out all open tasks in that area.

## Capture paths

Two ways in, **one inbox**: type directly into the `## Inbox` of today's note
(offline/mobile-safe via Obsidian Sync), or say "add task …" in chat and this
skill routes it to that same `## Inbox`. The morning flow reconciles both.

## First run / missing files

If `./diary/_tasks-config.md` does not exist, scaffold it with this content,
then continue:

```markdown
# Tasks Config

## Areas
- wiki, work, home, learning

## Owners
- @me (default)

## Horizon ladder
- today, this-week, this-sprint, this-quarter, longterm

## Drift thresholds
- #today: nag after 1 day
- #this-week: nag after 7 days
- delegated: nag after 7 days idle
<!-- horizons without a threshold above are never drift-flagged -->
```

If today's daily note does not exist, create it from `./diary/_template.md`
(or, if that is also absent, from the section structure listed in "Self-check").

## Self-check (run before reporting done)

After any `/tasks` run, confirm:
- [ ] No open `- [ ]` task remains in any *past* daily note (all swept forward).
- [ ] Every open task in today's note has a `(created: YYYY-MM-DD)` stamp.
- [ ] No task appears twice (one live copy).
- [ ] Nothing under `Wiki/` was modified.
- [ ] Today's note sections appear in template order: Inbox, Today, This week,
      This sprint, Longer horizons, Waiting on, ⚠️ Drifted.
- [ ] No horizon tag was changed without explicit user instruction.

## Manual trial-run (verification procedure)

To verify the skill end-to-end on a **scratch copy** (never the live diary):
1. Make a temp dir with two notes: a `<yesterday>.md` holding one open
   `- [ ] old task (created: <8 days ago>) #area/work #today`, and a fresh
   today's note whose `## Inbox` has the lines `buy milk` and
   `follow up on [[Spaced Repetition]]`.
2. Run the morning flow against that dir.
3. Confirm: inbox emptied; both inbox items are now tagged + `(created:)`-stamped
   tasks under a horizon; `old task` moved out of `<yesterday>.md` into today's
   `## Today`; `<yesterday>.md` retains no open task; `old task` is listed under
   `## ⚠️ Drifted` (8 days > `#today` 1-day threshold); the `[[Spaced Repetition]]`
   link is present and the Wiki page is untouched.
