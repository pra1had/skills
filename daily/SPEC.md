# `/daily` Skill — Design Spec

> Day-to-day task management for the PKs-Life vault, agent-maintained, living
> alongside (and linking one-way into) the LLM Wiki.
>
> Status: **spec approved 2026-06-21** via grilling session. Not yet built.
> Destination: this file moves into the Skills GitHub repo at `daily/` as the
> design reference; the runnable skill is `daily/SKILL.md`.

---

## 1. Philosophy

Mirror the Wiki contract: **you feed it, the agent files and links.** You capture
messily; the agent classifies, links, schedules, and reconciles. The point is to
not re-derive state each day — after the morning sweep, *today's note is the
answer to "what's open."*

The Wiki link is **one-way only**: tasks reference `[[Wiki pages]]`; task
processing **never writes into `Wiki/`**. Obsidian's backlinks pane is the
Wiki-side view. Consequence: the Wiki's CLAUDE.md contract is entirely out of
scope for this skill.

## 2. Architecture (portability-first)

The user is on **Obsidian Sync**, which does *not* carry hidden dot-folders
beyond `.obsidian`. So skill logic must not depend on Sync.

| Artifact | Lives in | Sync mechanism |
|---|---|---|
| Skill logic (`SKILL.md`, this spec) | **GitHub "Skills" repo** | git |
| Symlink `.claude/skills/daily` → repo | machine-local | re-created per machine |
| Daily notes (the task data) | **vault** `diary/YYYY-MM-DD.md` | Obsidian Sync |
| Daily-note template | **vault** `diary/_template.md` | Obsidian Sync |
| Personal config (areas/owners/horizons) | **vault** `diary/_tasks-config.md` | Obsidian Sync |
| Run state (`_state.md` watermark) | **vault** `diary/_state.md` | Obsidian Sync |

**Boundary rule:** repo = generic behavior; vault = config + data + template.
The skill carries **zero personal data** and is reusable across vaults. On first
run it reads `./diary/_tasks-config.md`; if absent, it scaffolds one.

**Disaster recovery** (machine loss): `git clone` the Skills repo · re-create the
symlink · vault returns via Obsidian Sync. Both halves recover independently.

## 3. The model

- **Single source of truth: today's note.** After the morning sweep,
  `diary/<today>.md` holds *all* open work. One file to look at.
- **Move-and-stamp sweep.** Open tasks are *physically relocated* into today's
  note and **deleted from their source note** — exactly one live copy exists.
  No tombstone left behind (keeps old notes clean).
- **Completed tasks complete in-place and never move.** A checked `[x]` stays
  under whatever horizon section it was in; the sweep skips it. Past notes become
  a done-journal / daily record, completions sitting beside their horizon. (There
  is no separate "Done today" section — revised 2026-06-21 post-approval.)
- **Staleness via created-date.** Every task carries `(created: YYYY-MM-DD)` so
  age and drift are computable without moving files or leaving trails.
- **Pure markdown, no plugins.** Plain `- [ ]`, plain `#tags`, human-readable
  `(created: ...)`. No Tasks/Dataview dependency (avoids re-introducing the
  portability gap; Model-1 makes cross-note aggregation largely redundant).
  Plain `- [ ]` is a strict subset of Tasks-plugin syntax, so adding plugins
  later is non-destructive.
- **Watermark-bounded sweep.** `diary/_state.md` holds `Last swept: <date>`. The
  sweep reads only notes dated ≥ the watermark (normally one), making scan cost
  O(days-since-last-run), independent of vault size. Missing/corrupt watermark →
  full-scan fallback that self-heals by writing a fresh watermark.

## 4. Task line format

```
- [ ] <text> (created: YYYY-MM-DD) #area/<area> #<horizon> @<owner>
```

- `@me` is the implicit default owner and may be omitted on own-tasks.
- Horizon tag is one of the ladder values (§5).
- Area is an inline tag, **not** a grouping key (grouping is by horizon).

Example:
```
- [ ] draft Q3 plan section 2 (created: 2026-06-19) #area/work #today
- [ ] follow up on [[Spaced Repetition]] — forgetting-curve detail (created: 2026-06-21) #area/wiki #this-week
- [ ] API spec for ingest endpoint (created: 2026-06-17) #area/work @bob
```

## 5. Dimensions

1. **Area** — `#area/...` bucket (e.g. `wiki`, `work`, `home`, `learning`).
   Roster defined in `_tasks-config.md`.
2. **Horizon (priority axis)** — ladder:
   `#today` → `#this-week` → `#this-sprint` → `#this-quarter` → `#longterm`.
   **Sticky + nag policy:** horizons never auto-change. The sweep *flags* drift
   (e.g. "`#today` for 4 days — keep / demote / drop?"); the user re-tags
   manually. The agent re-tags **only on explicit say-so** — never silent
   demotion.
3. **Created date** — `(created: YYYY-MM-DD)`, set at capture, preserved across
   sweeps. Powers staleness detection.
4. **Owner** — `@name`. Delegated tasks (owner ≠ `@me`) are tracked separately
   (§6). `_tasks-config.md` holds the known owner roster.

## 6. Delegation ("Waiting on")

Separate **"what I do"** from **"what I'm waiting on."**

- Delegated tasks live in the `## Waiting on` section, **grouped by `@owner`**,
  kept out of the actionable horizon lists.
- They **roll forward in the sweep** like own-tasks (you stay accountable) and
  carry `(created: ...)` so "how long has this been out?" is visible.
- Drift nag reframes as a follow-up: "delegated to @bob N days ago — check in?".
- **Delivered = done:** when the work arrives, the task is simply checked `[x]`.
  An opt-in middle state exists only on request ("mark delivered, pending my
  review" → parks under a `### Delivered, pending review` line).

## 7. Daily-note structure (`diary/_template.md`)

```markdown
# {{date:YYYY-MM-DD}}

## Inbox
<!-- dump anything here anytime; agent files it on next /daily run -->

---

## Today  #today

## This week  #this-week

## This sprint  #this-sprint

## Longer horizons

---

## Waiting on

---

## ⚠️ Drifted (agent flagged — you decide)
```

## 8. Operation

**Trigger: manual only.** The user opens Obsidian every morning and invokes the
skill. No scheduled/cloud agent (would risk piling up empty notes and can't
resolve ambiguity interactively).

`/daily` (no args) runs the full morning flow:

0. **Read the watermark.** Read `./diary/_state.md` and parse `Last swept: <date>`.
   This date marks the current *live note* (the one holding all open tasks) and
   bounds the scan: every subsequent read of prior notes considers **only notes
   dated ≥ this watermark** (normally just the live note, plus any newer notes you
   created since — e.g. inbox dumps on days `/daily` was not run).
   **If `_state.md` is missing or unparseable, fall back to scanning *all* prior
   daily notes** (legacy behavior), then write a fresh watermark in step 6.
1. **Process inbox** — read `## Inbox` of recent note(s); for each item:
   classify area, detect/assign owner, infer horizon (ask if ambiguous),
   match and link any `[[Wiki pages]]` (one-way), stamp `(created: ...)`,
   dedupe. Empty the inbox.
2. **Move-and-stamp sweep** — relocate every open task from notes selected by
   the watermark (step 0) into today's note; delete from source; preserve
   created-date.
3. **Regroup** by horizon ladder; place delegated tasks under `## Waiting on`
   by owner.
4. **Drift nag** — build `## ⚠️ Drifted` for: horizon tasks stale past their
   bucket, and delegated tasks idle past a threshold. Flag only; never re-tag.
5. **Summary** — print counts + the drift list for the user to act on.
6. **Advance the watermark.** Only after every selected source note is fully
   reconciled (all open tasks moved and sources updated), overwrite
   `./diary/_state.md` with `Last swept: <today>`. A crash before this point
   leaves the watermark on the prior date, so the next run safely re-processes.

**Optional verbs:**
- `/daily capture <text>` — quick add to today's `## Inbox` (no full run).
- `/daily review @bob` / `/daily review #area/wiki` — filtered read-out.

**Capture paths:** type directly into `## Inbox` (offline/mobile-safe), or say
"add task…" in chat → the skill routes it to the same inbox. One source of truth.

## 9. `diary/_tasks-config.md` (vault-local, scaffolded if absent)

Holds the per-vault personal taxonomy the generic skill reads:

```markdown
# Tasks Config

## Areas
- wiki, work, home, learning

## Owners
- @me (default), @bob, @priya

## Horizon ladder
- today, this-week, this-sprint, this-quarter, longterm

## Drift thresholds
- #today: nag after 1 day
- #this-week: nag after 7 days
- delegated: nag after 7 days idle
```

## 10. Build checklist (when implementation is approved)

- [ ] Create the GitHub **Skills** repo.
- [ ] Author `daily/SKILL.md` (frontmatter `name: daily` + the §8 procedure)
      and move this spec to `daily/SPEC.md`.
- [ ] Symlink `.claude/skills/daily` → `<skills-repo>/daily` (relative link,
      matching the existing `.claude/skills/*` convention).
- [ ] Add `diary/_template.md` (§7) and wire Obsidian Daily Notes to use it.
- [ ] Scaffold `diary/_tasks-config.md` (§9).
- [ ] **Cleanup:** move the stray root `2026-06-21.md` into `diary/` so the skill
      has one consistent daily-note location.
- [ ] Document the new-machine recovery steps in the Skills repo README.

## 11. Decision log (from grilling, 2026-06-21)

| # | Decision |
|---|---|
| 1 | Scope = lightweight capture/roll-forward **(a)** + knowledge-linked **(d)**; agent-maintained |
| 2 | Capture = daily-note inbox **and** conversational, reconciled to one inbox |
| 3 | Filing = tasks live in daily notes, enriched in place |
| 4 | Querying = pure agent-driven markdown (no Tasks/Dataview) |
| 5 | Canonical = today's note (Model 1); move-and-stamp; no tombstone; created-date |
| 6 | Wiki linkage = one-way only; Wiki never written by task processing |
| 7 | Dimensions = area, horizon, created, owner |
| 8 | Horizon decay = sticky + nag; user demotes manually |
| 9 | Delegation = `@owner` + `## Waiting on`; rolls forward; delivered = done |
| 10 | Trigger = manual `/daily` each morning |
| 11 | Governance = a skill (not a contract doc); single `/daily` skill |
| 12 | Portability = generic skill in GitHub Skills repo, symlinked; vault holds config + data |
| 13 | Syntax = pure markdown |
| 14 | Backward-scan scaling = **watermark** (`diary/_state.md`, `Last swept`), not a backlog file; missing→full-scan fallback |
| 15 | Self-check "no open task in past notes" is a trusted invariant; full audit deferred to a future `/daily fsck` |
