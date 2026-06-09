# Foundry - An Agent-Run Obsidian Vault

This is an autonomous knowledge vault maintained by Claude. It follows the Karpathy wiki pattern: raw sources are ingested and compiled into a wiki of concepts, connections, and open questions. This file is the single source of truth for how Claude operates inside it.

---

## Companion vault (optional)

If you keep a personal vault (a commonplace, Zettelkasten, or notes folder), you can link it alongside this one. The contract:

- **Your vault** — read-only for Claude. Never write, edit, move, or delete anything there. Cross-link into it with `[[YourVault/Path/Note Title]]` style references when a Foundry note is informed by your writing.
- **Foundry** — Claude writes, maintains, reorganises. You read, query, and occasionally correct.

One-way read, cross-linkable. That's the contract. If you don't have a companion vault, Foundry works fine standalone.

---

## Optional integrations

### Linkding

Set these environment variables before running any Foundry command:

```
LINKDING_URL=https://your-linkding-instance.com   # no trailing slash
LINKDING_TOKEN=your-api-token
```

Tag bookmarks in Linkding to queue them for the Foundry:
- `foundry` — queue for ingest (cleared from the tag after sync)
- `foundry-primary` — queue for ingest AND set `primary: true` on the source note

`/foundry-sync` fetches all `foundry`-tagged bookmarks via the Linkding REST API, writes inbox stubs, and removes the `foundry` tag so each bookmark is only processed once. Full text is fetched during the subsequent `/foundry-ingest` run.

If `LINKDING_URL` or `LINKDING_TOKEN` are not set, `/foundry-sync` and the Linkding healthcheck in `/foundry-lint` will skip gracefully and report the omission.

### qmd

qmd provides semantic search over the vault. After every `/foundry-ingest` or `/foundry-compile` run, Foundry triggers `qmd update` to keep the index current. `/foundry-ask` queries qmd first before falling back to index scanning.

The vault must be registered as a qmd collection before first use:
```bash
qmd collection add foundry /path/to/this/vault
qmd embed
```

If `qmd` is not in PATH or its index is missing, `/foundry-ask` falls back to index scanning and `/foundry-lint` reports the gap.

---

## Directory structure

Three layers, following the Karpathy wiki pattern:

| Directory | Purpose | Who writes |
|---|---|---|
| `inbox/` | Staging for unprocessed drops (PDFs, URLs, screenshots, pasted text) | You drop, Claude clears |
| `sources/` | Immutable atomic source notes — one per article/paper/transcript | Claude on ingest; minor edits only after creation |
| `wiki/` | LLM-maintained pages: concepts, queries, people, index, log, health | Claude maintains |

Special files in `wiki/_meta/`:
- **`index.md`** — catalog of every page, keyword glossary, research threads, open questions, prompts, candidates. Claude reads this first when answering a query. Updated on every operation.
- **`log.md`** — append-only chronological record. Each entry: `## [YYYY-MM-DD] operation | Title`. Never rewritten, only appended.
- **`health.md`** — lint dashboard. Overwritten each `/foundry-lint` run.

---

## File naming

- **Source notes**: Title Case — `The Rosetta Stone of Design Engineering.md`
- **Concept articles**: Title Case, descriptive — `Design-engineering handoff.md`
- **People**: `FirstName LastName.md` with hyphens only inside compound names. If surname unknown, `FirstName.md` and note the uncertainty.
- **Queries**: `YYYY-MM-DD-slug.md`
- No emoji, no unicode hacks, no date prefixes in titles (dates go in front-matter)

---

## Front-matter

YAML frontmatter delimited by `---`. Blank line after closing `---` then body.

Note from @jameesy: This is very personal to the way I like to use Obsidian, and the way that I use tags across vaults. The format is now YAML — change the tag values or add fields to suit however you are currently working.

**Source notes (`sources/`):**
```yaml
---
tags:
  - type/source
  - area/craft/ai
  - keyword/knowledge-management
  - keyword/llms
date_created: 2026-04-13
source: https://example.com/article
source_hash: sha256:abc123def456...
primary: false
---
```

- `tags:` — all type/area/keyword tags as a YAML list. **No `#` prefix** — Obsidian adds it in the UI. Dataview queries use: `WHERE contains(tags, "type/source")`
- `date_created:` — ISO date string (e.g. `2026-04-13`). No `[[` wikilink wrapping.
- `source_hash:` — SHA-256 hex digest of the raw source content, computed once on ingest, never changed. Lint uses this to detect if the immutable source note body has drifted. Use `sha256:pending` for sources ingested before this field existed.
- `primary: false` — set `true` for authoritative singletons (government publications, official plan documents, canonical specs). `primary: true` sources bypass the 2-source rule for concept promotion.

Body:

**Summary**
3-5 sentences. Core claim.

**Key points**
- 5-10 tight bullets

**Claude's notes**
One paragraph: what's interesting, where it connects, what it contradicts.

**Wiki pages — concepts (`wiki/`):**
```yaml
---
tags:
  - type/concept
  - area/craft/ai
  - keyword/knowledge-management
date_created: 2026-04-13
sources:
  - "[[Source One]]"
  - "[[Source Two]]"
related:
  - page: "[[Concept A]]"
    rel: extends
  - page: "[[Concept B]]"
    rel: contradicts
---
```

- `sources:` — YAML list of wikilinks to the source notes the concept synthesises
- `related:` — YAML list of `{page, rel}` objects. Minimum 2 entries (or note in body why concept is an island). `rel` must be one of: `extends`, `contradicts`, `supersedes`, `see-also`. Use `contradicts` when sources genuinely disagree — do not collapse the tension.

Body: `What it is` > `Why it matters` > `Key points` > `Evidence across sources` > `Open questions` > `Prompts`.

**Voice for concepts.** The Foundry gives you a *foundation* for your own writing — not a finished essay. Favour sharp one-liners, evidence citations, and open questions over flowing prose. Key points should be pithy — one line each, claim-shaped. When in doubt, write less.

**Prompts.** Essay-shaped prompts where the concept intersects your existing notes or thinking. Distinct from Open questions (research gaps): Prompts are *"you could write this now."* One or two sentences each, pointed and specific. Empty is fine.

**Wiki pages — queries:**
```yaml
---
tags:
  - type/query
  - area/craft/management
  - keyword/leadership
date_created: 2026-04-14
question: "the question asked"
---
```

**Wiki pages — people:**
```yaml
---
tags:
  - type/person
  - area/craft/ai
  - keyword/llms
date_created: 2026-04-14
---
```

Body:

One-line identifier. Topic description.

**Sources in the Foundry**
- [[Source Title]]

**Concepts they inform**
- [[Concept Title]]

---

## Tag taxonomy

> Tags go in the `tags:` YAML list **without** the `#` prefix. Obsidian renders them with `#` in the UI. Dataview queries use `WHERE contains(tags, "type/source")`. The hierarchy is unchanged.

Hierarchical sub-areas under Craft. Customise the top-level areas to match your life.

**`#area/`** — examples: Self, Craft, Work, Health, Finances, Meta (use whatever top-level areas fit your life)
**`#area/craft/`** — design, engineering, management, ai, product, writing

**`#type/`** — each note gets exactly one:
- `source` — an ingested external source (in `sources/`)
- `concept` — a synthesised wiki page built from 2+ sources
- `query` — a research report answering a question
- `person` — an entity page for a thinker/author
- `meta` — vault infrastructure (index, log, health)

**`#keyword/`** — free-form but curated via the Keywords section of `wiki/_meta/index.md`. Before creating a new keyword:
1. Check `wiki/_meta/index.md`
2. If a near-match exists, reuse it
3. If genuinely new, add it to the Keywords section with a one-line definition

---

## People — when to create a page

Three tiers:

| Tier | Trigger | Action |
|---|---|---|
| Author | Person authored a source in `sources/` | Always create a page in `wiki/` on ingest |
| Subject | Source is substantively about a person | Create page with a richer profile |
| Passing reference | Mentioned in passing | Use `[[Name]]` wikilink without creating a file. Create only on the **second** independent citation |

People pages stay thin — connector nodes, not essays.

---

## Citation & linking

- **Every claim in a concept must be traceable.** `Sources:` front-matter lists the source notes the claim rests on.
- **Backlink rule**: every new concept links at least 2 existing concepts in `Related:`, or notes why it's an island (flagged in `wiki/_meta/health.md` Orphans section).
- **Cross-vault links** (if using a companion vault) use `[[VaultName/Path/Note Title]]` form.
- **Never break a link.** If renaming, update all backlinks.

---

## Operations

### Ingest (`/foundry-ingest`)

Process anything in `inbox/` — web clippings, PDFs, URLs, pasted text — into clean atomic source notes in `sources/`.

Flow: read source > normalise into source note > fill front-matter > write summary + key points + Claude's notes > cross-reference companion vault (if any) > update `wiki/_meta/index.md` (Sources section, keyword counts) > append to `wiki/_meta/log.md` > clear inbox.

A single source might touch the source note, index, log, and occasionally a people page.

Won't do: write into your companion vault, create concept articles (that's compile's job), invent work if inbox is empty.

### Compile (`/foundry-compile`)

Scan `sources/` for un-compiled sources (not cited in any concept's `Sources:` field). Either extend an existing concept or spin out a new one — but only when the 2-source rule is met. Otherwise, log the theme to the Candidates section of `wiki/_meta/index.md` and wait.

After each run: update Research Threads, append new Prompts, update keyword counts — all in `wiki/_meta/index.md`. Append to `wiki/_meta/log.md`.

Won't do: write into your companion vault, spin out a concept from a single source, break links.

### Query (`/foundry-ask`)

Research a question across the Foundry and companion vault (read-only). Write report to `wiki/YYYY-MM-DD-slug.md`. Every claim cites its source. Good answers get filed as wiki pages — explorations compound.

Findings with 2+ sources feed back to Candidates. Open gaps feed to Open Questions. Essay prompts to Prompts — all in `wiki/_meta/index.md`.

Won't do: write into your companion vault, invent citations, pad thin answers.

### Sync (`/foundry-sync`)

Pull queued bookmarks from Linkding into `inbox/` as URL stubs. Requires `LINKDING_URL` and `LINKDING_TOKEN` env vars. No-ops cleanly if either is unset.

### Lint (`/foundry-lint`)

Health-check the whole vault. Overwrite `wiki/_meta/health.md` with: Stats, Orphans, Candidates needing attention, Keyword drift, Linkding connectivity, qmd index status.

---

## What NOT to do

- Don't write into the companion vault (if one exists).
- Don't create files outside `sources/`, `wiki/`, or `inbox/`.
- Don't speculatively create concepts from a single source. Wait for cross-source signal.
- Don't use emojis.
- Don't add TODO comments. If something's missing, log it in Candidates.
- Don't create helpers, templates, or meta-infrastructure. The schema (this file) + index + log is all the infrastructure the vault needs.
