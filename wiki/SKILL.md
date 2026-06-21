---
name: wiki
description: Operating contract for the LLM-maintained wiki in this Obsidian vault's Wiki/ folder — ingest raw sources into interlinked notes, query the wiki to answer questions without re-deriving, and lint for health. Use whenever the user wants to ingest/process a source into the wiki, asks "what does the wiki say…", adds a clipping to digest, or wants to check/repair wiki links, orphans, or contradictions. The wiki distills reading once into durable interlinked notes (Karpathy's "llm-wiki" pattern).
---

# wiki

An incrementally-built, interlinked wiki distilled from raw sources (Karpathy's
"llm-wiki" pattern). **The point is to do the reading/synthesis once, write it
down as durable interlinked notes, and never re-derive it on each query.**

Everything the wiki owns lives under `./Wiki/` in this vault. Raw sources live
elsewhere (e.g. `./Clippings/`) and are read-only inputs — the wiki points *at*
them, never alters them. The wiki is searchable via **qmd** (collection
`PKs-Life`) and the **Obsidian CLI** is available as `obsidian` on PATH.

## Non-negotiable rules

1. **Raw sources are IMMUTABLE.** Never create, edit, move, rename, or delete
   anything in source folders (e.g. `Clippings/`). Read them only. The wiki
   points *at* sources; it never alters them.
2. **The agent owns `Wiki/` entirely.** All wiki pages are agent-managed.
3. **Always update [[index]] and [[log]] on every change.** No silent edits.
   `index.md` is the catalog; `log.md` is the append-only history.
4. **Link heavily with `[[wikilinks]]`** so Obsidian graph view stays useful.
   Prefer linking an entity/concept over restating it.
5. **Sources stay factual. Interpretation goes in concepts/synthesis.** A
   source page says what the source said; opinions/analysis live elsewhere.
6. **Contradictions are surfaced, never overwritten.** When sources disagree,
   note it explicitly on the relevant concept page and in [[overview]] (Open
   questions & contradictions) — cite both sources.
7. **Dates are absolute** `YYYY-MM-DD`, never "today"/"last week".

## Architecture

```
Wiki/
├── index.md       # content catalog + "Unprocessed sources" list (type: index)
├── log.md         # append-only operations log (type: log)
├── overview.md    # synthesized big picture (type: overview)
├── sources/       # one factual summary per ingested source (type: source)
├── entities/      # people, tools, orgs, products (type: entity)
├── concepts/      # ideas, patterns, techniques — interpretation (type: concept)
└── synthesis/     # answers spanning multiple pages (type: synthesis)
```

## Page conventions

- **Filenames:** wiki *content* pages (entities, concepts, sources, synthesis)
  use **Title Case** to preserve Obsidian wikilink autocomplete (e.g.
  `Spaced Repetition.md`). The three meta files keep their conventional lowercase
  names: `index.md`, `log.md`, `overview.md`.
- **Frontmatter:** every page has `type:` and `date_updated:` plus `wiki/*` tags.
  - source: `type: source`, `source_path:` (path to the immutable raw file),
    `date_ingested:`, tags `wiki/source`.
  - entity: `type: entity`, `entity_kind:` (person|tool|org|product), `wiki/entity`.
  - concept: `type: concept`, `source_count:`, `confidence:` (low|medium|high),
    `wiki/concept`.
  - synthesis: `type: synthesis`, `question:`, `source_count:`, `wiki/synthesis`.
- **Body:** lead with a one-line definition, then details. Use `[[wikilinks]]`
  inline. Cross-link related pages at the bottom under "Related".

## Operations

### Ingest — turn a raw source into wiki pages

1. Read the raw source (read-only) and any pages it relates to.
2. Create/update a factual `sources/<Title>.md` page with `source_path:`.
3. Extract entities → create/update `entities/*`; extract ideas → `concepts/*`
   (this is where interpretation and contradiction-noting happen).
4. Wikilink everything bidirectionally.
5. Update [[index]] (move the item off "Unprocessed sources", bump stats) and
   append an [[log]] entry.
6. **Before re-indexing, run the link check** (`scripts/check-wiki-links.sh`)
   and fix anything it reports — it must exit 0. *Then* run `qmd update && qmd
   embed` if search matters. Never run `qmd update` on a failing link check.

### Query — answer a question from the wiki

1. Read [[index]]/[[overview]]; use `qmd query "..."` to locate relevant pages.
2. Answer from existing pages where possible (the whole point — don't re-derive).
3. If the answer is reusable, file it as a `synthesis/<Question>.md` page, link
   it, and update [[index]] + [[log]].

### Lint — maintain health

1. Run `scripts/check-wiki-links.sh` first for broken / newline-wrapped
   `[[wikilinks]]` that Obsidian silently fails to resolve.
2. Then find orphan pages (no inbound links), stale `date_updated`, unflagged
   contradictions, and missing cross-references. Fix or report.
3. Reconcile [[index]] stats with actual files. Append a [[log]] entry.

## qmd / link-check discipline

`scripts/check-wiki-links.sh` **must exit 0 before any `qmd update`** — it
catches broken and newline-wrapped `[[wikilinks]]` that Obsidian silently fails
to resolve. After content changes, run `qmd update && qmd embed` to refresh
search. Never run `qmd update` on a failing link check.

## Self-check (run before reporting done)

- [ ] [[index]] reflects the change (catalog entry + stats), and the source is
      off "Unprocessed sources" if it was ingested.
- [ ] An [[log]] entry was appended for this operation.
- [ ] Every new/edited page has correct frontmatter (`type:`, `date_updated:`,
      `wiki/*` tags) and absolute dates.
- [ ] New links are bidirectional and no orphan page was created.
- [ ] No raw source file (e.g. under `Clippings/`) was created, edited, moved,
      renamed, or deleted.
- [ ] `scripts/check-wiki-links.sh` exits 0 (run it before any `qmd update`).
- [ ] Any source disagreement is surfaced on the concept page and [[overview]],
      not overwritten.

## Status

Bootstrapped 2026-06-20. Structure + schema in place; **no sources ingested
yet**. Next step: calibration ingest of 2–3 rich sources for template review
before batching the rest.
