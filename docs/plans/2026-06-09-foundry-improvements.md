# Foundry Vault Improvements Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Upgrade foundry-vault with YAML frontmatter (Dataview-compatible), source provenance hashing, typed relations, enhanced lint, and a primary-source override — prioritised lowest-cost/highest-payoff first.

**Architecture:** All changes are to Markdown prompt files (`CLAUDE.md`, `.claude/commands/*.md`) and the two example files in `sources/` and `wiki/`. No runtime code; Claude is the runtime. Tasks are ordered by dependency: schema first, then commands that reference the schema, then lint that validates the schema.

**Tech Stack:** Markdown, Obsidian YAML frontmatter, Dataview plugin (consumer), SHA-256 hashing (computed by Claude on ingest using bash or Python one-liner), standard wikilink syntax.

---

## Background: what changes and why

| Item | Current state | Target state | Reason |
|------|--------------|--------------|--------|
| Frontmatter format | Plain key-value, no delimiters | YAML with `---` delimiters | Enables Dataview queries; required by Obsidian Properties UI |
| Source provenance | None | `source_hash: sha256:<hex>` field | Idempotency on re-ingest; staleness detection in lint |
| Relations | Bare wikilinks in `Related:` | Typed YAML list `related: [{page, rel}]` | Makes contradictions queryable; doesn't require collapsing disagreements |
| 2-source rule | Applies to all sources | Bypassed for `primary: true` sources | Authoritative singletons (plan docs, IRS pubs) shouldn't wait for a second source |
| Lint checks | Structure + orphans + keywords | + hash drift + typed-relation vocab + citation integrity | Lint becomes the vault's immune system for the new fields |

---

## Task 1: Define the new YAML frontmatter schema in CLAUDE.md

**Files:**
- Modify: `CLAUDE.md` (the `Front-matter` section, lines 46–116)

This task establishes the ground truth. Every other task references it.

### Step 1: Write the new source note frontmatter block in CLAUDE.md

Replace the `Front-matter` section's **Source notes** example with:

```yaml
---
tags:
  - type/source
  - area/craft/ai
  - keyword/knowledge-management
  - keyword/llms
date_created: 2026-04-13
source: https://example.com/article
source_hash: sha256:abc123def456...  # hex digest of source content at ingest time
primary: false                        # set true for authoritative singletons (IRS pubs, plan docs, spec sheets)
---
```

Notes to add inline in CLAUDE.md:
- `tags:` — all type/area/keyword tags go here as a YAML list, **without** the `#` prefix (Obsidian adds it in the UI)
- `date_created:` — ISO date string, no `[[` wikilink wrapping
- `source_hash:` — computed once on ingest, never changed after. Lint uses this to detect source-note drift.
- `primary: true/false` — `true` means this source bypasses the 2-source rule for concept promotion. Use for authoritative singletons only.

### Step 2: Write the new concept page frontmatter block in CLAUDE.md

Replace the **Wiki pages — concepts** example with:

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

Notes to add:
- `sources:` — YAML list of wikilinks to source notes
- `related:` — YAML list of objects. `rel` must be one of: `extends`, `contradicts`, `supersedes`, `see-also`. Lint will flag unknown values.
- Minimum 2 `related` entries, or note in body why the concept is an island.

### Step 3: Write the new query and person page frontmatter blocks

```yaml
# Query
---
tags:
  - type/query
  - area/craft/management
  - keyword/leadership
date_created: 2026-04-14
question: "the question asked"
---

# Person
---
tags:
  - type/person
  - area/craft/ai
  - keyword/llms
date_created: 2026-04-14
---
```

### Step 4: Update the Tag taxonomy section of CLAUDE.md

Replace the old tag-as-prefix prose with:

> Tags go in the `tags:` YAML list **without** the `#` prefix. Obsidian renders them with `#` in the UI. Dataview queries use: `WHERE contains(tags, "type/source")`. The hierarchy is the same; only the syntax changes.

### Step 5: Verify the schema makes sense by rereading the full CLAUDE.md front-matter section

Read the updated `CLAUDE.md` top-to-bottom. Check:
- [ ] Source note example has `source_hash` and `primary` fields
- [ ] Concept example has `related` as a YAML list with `{page, rel}` objects
- [ ] Query and person examples are consistent
- [ ] Tag taxonomy section explains the no-# rule
- [ ] No old-style plain key-value examples remain

### Step 6: Commit

```bash
git -C /Users/pops/foundry-vault add CLAUDE.md
git -C /Users/pops/foundry-vault commit -m "schema: migrate frontmatter to YAML with hash, typed relations, primary flag"
```

---

## Task 2: Migrate the two existing example files to YAML frontmatter

**Files:**
- Modify: `sources/LLM Knowledge Bases (Karpathy).md`
- Modify: `wiki/Two-layer knowledge systems.md`
- Modify: `wiki/Andrej Karpathy.md`
- Modify: `wiki/_meta/health.md`
- Modify: `wiki/_meta/index.md`
- Modify: `wiki/_meta/log.md`

These are the only existing content files. Migrating them validates the schema is complete before updating the commands.

### Step 1: Migrate `sources/LLM Knowledge Bases (Karpathy).md`

Current header:
```
Type: #type/source
Area: #area/craft/ai
Keyword: #keyword/knowledge-management #keyword/llms #keyword/obsidian #keyword/second-brain
Date created: [[2026-04-13]]
Source: https://x.com/karpathy/status/2039805659525644595

---
```

Replace with:
```yaml
---
tags:
  - type/source
  - area/craft/ai
  - keyword/knowledge-management
  - keyword/llms
  - keyword/obsidian
  - keyword/second-brain
date_created: 2026-04-13
source: https://x.com/karpathy/status/2039805659525644595
source_hash: sha256:pending   # not retroactively computable; mark pending, lint will warn
primary: false
---
```

Note: For existing sources where hash wasn't computed on original ingest, use `sha256:pending`. Lint will flag these as "hash not yet computed" (a warning, not an error) until they are re-ingested or manually verified.

### Step 2: Migrate `wiki/Two-layer knowledge systems.md`

Current header:
```
Type: #type/concept
Area: #area/craft/ai
Keyword: #keyword/knowledge-management #keyword/llms #keyword/second-brain
Date created: [[2026-04-13]]
Sources: [[LLM Knowledge Bases (Karpathy)]], [[What Matters in the Age of AI Is Taste]]
Related:

---
```

Replace with:
```yaml
---
tags:
  - type/concept
  - area/craft/ai
  - keyword/knowledge-management
  - keyword/llms
  - keyword/second-brain
date_created: 2026-04-13
sources:
  - "[[LLM Knowledge Bases (Karpathy)]]"
  - "[[What Matters in the Age of AI Is Taste]]"
related: []
---
```

Note: `related:` is empty here — this concept has no related concepts yet, and that's legitimately an island. The lint already flags concepts with fewer than 2 related entries; this is a known orphan.

### Step 3: Migrate remaining `wiki/` files

Apply same pattern to `wiki/Andrej Karpathy.md` (type/person) and `wiki/_meta/*.md` (type/meta). For _meta files, the frontmatter is minimal — just tags, date_created.

### Step 4: Verify all files parse as valid YAML

```bash
cd /Users/pops/foundry-vault && python3 -c "
import os, yaml
for root, dirs, files in os.walk('.'):
    dirs[:] = [d for d in dirs if d not in ['.git', '.obsidian']]
    for f in files:
        if f.endswith('.md'):
            path = os.path.join(root, f)
            content = open(path).read()
            if content.startswith('---'):
                parts = content.split('---', 2)
                if len(parts) >= 3:
                    try:
                        yaml.safe_load(parts[1])
                    except yaml.YAMLError as e:
                        print(f'YAML ERROR in {path}: {e}')
print('Done')
"
```

Expected: `Done` with no errors.

### Step 5: Commit

```bash
git -C /Users/pops/foundry-vault add sources/ wiki/
git -C /Users/pops/foundry-vault commit -m "migrate: convert existing files to YAML frontmatter"
```

---

## Task 3: Update `foundry-ingest.md` for YAML + hash + primary flag

**Files:**
- Modify: `.claude/commands/foundry-ingest.md`

### Step 1: Update the frontmatter-filling instruction

Replace the "Fill front-matter per CLAUDE.md" step with:

> **Fill front-matter per CLAUDE.md.** Write valid YAML between `---` delimiters. Required fields: `tags` (list), `date_created` (ISO string), `source` (URL or ref), `source_hash` (see below), `primary` (bool, default false).

### Step 2: Add the hash computation instruction

Add a new numbered step after "Fill front-matter":

> **Compute `source_hash`.** Before writing the source note, compute a SHA-256 digest of the raw source content (the full text of the article, PDF, or clipping, before your summarisation). Run:
> ```
> echo -n "<source_text>" | sha256sum
> ```
> or equivalent. Store the result as `source_hash: sha256:<hexdigest>`. This fingerprint is immutable — never update it after creation. Its only job is to let lint detect if the note body drifts from the original ingest.

### Step 3: Add idempotency check instruction

Add a step before writing the source note:

> **Idempotency check.** Before creating a source note, compute the source hash and grep `sources/` for an existing note with that `source_hash`. If a match exists, skip and log `already ingested: <title>`. Don't create a duplicate.

### Step 4: Add primary-source handling instruction

Add a note to the "Create a people page" step:

> If the source is an authoritative singleton (a government publication, official plan document, a canonical spec — not a commentary or secondary account), set `primary: true`. These bypass the 2-source rule and can seed a concept article immediately in the compile loop.

### Step 5: Verify the command file is self-consistent

Read `.claude/commands/foundry-ingest.md` top to bottom. Confirm:
- [ ] Frontmatter format instruction points to CLAUDE.md (which now has YAML examples)
- [ ] Hash computation step is present and unambiguous
- [ ] Idempotency check step is present
- [ ] Primary flag guidance is present

### Step 6: Commit

```bash
git -C /Users/pops/foundry-vault add .claude/commands/foundry-ingest.md
git -C /Users/pops/foundry-vault commit -m "ingest: add hash computation, idempotency check, primary-source flag"
```

---

## Task 4: Update `foundry-compile.md` for YAML + typed relations + primary override

**Files:**
- Modify: `.claude/commands/foundry-compile.md`

### Step 1: Update un-compiled detection method

Current: "A source is un-compiled if its title doesn't appear in any concept article's `Sources:` field."

Replace with:

> A source is un-compiled if its title doesn't appear in any concept article's `sources:` YAML list. To check: grep `wiki/` for `"[[<source-title>]]"` in a `sources:` block.

### Step 2: Update concept frontmatter writing instruction

Replace the "A new concept article needs" list with:

> **A new concept article needs** YAML frontmatter per CLAUDE.md:
> - `tags:` — type/concept, area, and keyword tags
> - `date_created:` — today as ISO string
> - `sources:` — YAML list of wikilinks to the source notes it synthesises
> - `related:` — YAML list of `{page, rel}` objects. Minimum 2 entries. `rel` must be one of: `extends`, `contradicts`, `supersedes`, `see-also`. Use `contradicts` when sources genuinely disagree — do not collapse the tension into one reading.

### Step 3: Add typed relation guidance

Add a note after the concept frontmatter block:

> **Picking `rel` values.** Use `extends` when this concept builds on the related one. Use `contradicts` when this concept's core claim is in real tension with the related one — name the disagreement explicitly in the body under a **Tensions** subheading. Use `supersedes` sparingly, only when new evidence makes an older concept obsolete (note it in the log). Use `see-also` as the default when the relationship is real but doesn't fit the other types.

### Step 4: Add primary-source override to the 2-source rule

In the "Decide: extend or spin out" section, add:

> **Primary-source exception.** If the source note has `primary: true`, you may spin out a new concept from it alone without waiting for a second source. Mark the concept with a comment in the body: `> Single primary source — watch for corroboration.` This is reserved for authoritative documents (official specs, government publications) where a "second independent source" may never meaningfully exist.

### Step 5: Verify the command file is internally consistent

Read `.claude/commands/foundry-compile.md` top to bottom. Confirm:
- [ ] Un-compiled detection references `sources:` YAML list (not `Sources:` plain field)
- [ ] New concept frontmatter uses YAML with typed `related:` list
- [ ] `rel` vocabulary is stated (extends/contradicts/supersedes/see-also)
- [ ] Primary-source exception is present

### Step 6: Commit

```bash
git -C /Users/pops/foundry-vault add .claude/commands/foundry-compile.md
git -C /Users/pops/foundry-vault commit -m "compile: YAML sources list, typed relations, primary-source exception"
```

---

## Task 5: Overhaul `foundry-lint.md` with hash drift and typed-relation validation

**Files:**
- Modify: `.claude/commands/foundry-lint.md`

This is the most substantial change. The existing lint already covers front-matter fields, orphans, broken links, index staleness, and keyword drift. We add three new checks.

### Step 1: Add Check 8 — Source hash drift

After the existing "Check 7: Keyword drift" section, add:

> #### 8. Source hash drift
>
> For every file in `sources/`:
> 1. Read the `source_hash:` field from YAML frontmatter.
> 2. If the value is `sha256:pending`, flag as **Hash not computed** (warning, not error).
> 3. If the value is present and not `pending`, re-compute SHA-256 of the file body (everything after the closing `---` of the frontmatter). If the digest does not match the stored hash, flag as **Hash mismatch** (error) — the supposedly-immutable source note has been edited since ingest.
>
> Report format:
> - `[[Source Title]]` — Hash not computed (pending)
> - `[[Source Title]]` — Hash mismatch: stored `sha256:abc123`, current body hashes to `sha256:def456`
>
> Sources are supposed to be immutable after creation. A mismatch is not automatically wrong (you may have intentionally corrected a transcription error), but it must be visible.

### Step 2: Add Check 9 — Typed relation vocabulary

After Check 8, add:

> #### 9. Typed relation vocabulary
>
> For every concept file in `wiki/`, read the `related:` YAML list. For each entry:
> 1. Verify `page:` is a wikilink pointing to an existing file in `wiki/`.
> 2. Verify `rel:` is one of: `extends`, `contradicts`, `supersedes`, `see-also`.
>
> Flag:
> - `[[Concept]]` → `[[Target]]` — unknown rel value: `<value>` (valid: extends, contradicts, supersedes, see-also)
> - `[[Concept]]` → `[[Target]]` — broken page link (target file does not exist)
>
> Also flag concepts where `related:` is an empty list `[]` and no island-justification comment exists in the body.

### Step 3: Add Check 10 — Citation integrity (sources list)

After Check 9, add:

> #### 10. Citation integrity
>
> For every concept file in `wiki/`, read the `sources:` YAML list. For each wikilink entry:
> 1. Verify the linked file exists in `sources/`.
> 2. Verify the linked source note has `source_hash:` present (not missing entirely).
>
> Flag:
> - `[[Concept]]` cites `[[Source Title]]` — source file not found in `sources/`
> - `[[Concept]]` cites `[[Source Title]]` — source note missing `source_hash` field (ingest predates provenance tracking)
>
> Note: this check supersedes the existing "Uncited source notes" orphan check for concepts. Keep the orphan check for sources not yet cited by any concept.

### Step 4: Update the `health.md` template in the command

Add the three new sections to the health.md output template:

```
#### 8. Source hash drift
- _(clean)_ / list of hash issues

#### 9. Typed relation vocabulary
- _(clean)_ / list of vocab violations

#### 10. Citation integrity
- _(clean)_ / list of broken citations
```

### Step 5: Update the log entry format

Update the log append template to include the new check counts:

```
## [YYYY-MM-DD] lint | Health check
- Sources: N, Concepts: N, Queries: N, People: N
- Orphans: N, Broken links: N, Index issues: N
- Keyword drift flags: N
- Hash drift: N (pending: N, mismatch: N)
- Relation vocab violations: N
- Citation integrity failures: N
```

### Step 6: Operational validation

To verify the lint changes work, do a dry run immediately after committing:
1. Run `/foundry-lint --dry-run` on the vault
2. Confirm the output includes sections 8, 9, and 10
3. Confirm `[[LLM Knowledge Bases (Karpathy)]]` is flagged as "Hash not computed (pending)" under Check 8
4. Confirm `[[Two-layer knowledge systems]]` is flagged under Check 9 for empty `related: []`
5. Confirm no false positives in sections 1–7

### Step 7: Commit

```bash
git -C /Users/pops/foundry-vault add .claude/commands/foundry-lint.md
git -C /Users/pops/foundry-vault commit -m "lint: add hash-drift, typed-relation vocab, citation integrity checks"
```

---

## Task 6: Update `foundry-ask.md` for YAML source field names

**Files:**
- Modify: `.claude/commands/foundry-ask.md`

Minor: the query output template uses plain key-value frontmatter. Update it to YAML to stay consistent.

### Step 1: Update the query output frontmatter template

Replace:
```
Type: #type/query
Area: #area/...
Keyword: #keyword/...
Date created: [[YYYY-MM-DD]]
Question: <the user's exact question>
```

With:
```yaml
---
tags:
  - type/query
  - area/...
  - keyword/...
date_created: YYYY-MM-DD
question: "<the user's exact question>"
---
```

### Step 2: Update all references to `Sources:` in the feedback-into-the-graph section

The instruction "Any finding backed by ≥2 sources" is prose guidance, not a field reference — no change needed.

### Step 3: Commit

```bash
git -C /Users/pops/foundry-vault add .claude/commands/foundry-ask.md
git -C /Users/pops/foundry-vault commit -m "ask: update query output template to YAML frontmatter"
```

---

## Task 7: Push branch and open PR

### Step 1: Push to fork

```bash
git -C /Users/pops/foundry-vault push origin main
```

### Step 2: Verify on GitHub

```bash
gh -R jjcrawdaddy/foundry-vault repo view --web
```

Confirm:
- [ ] All 6 commits are present
- [ ] `CLAUDE.md` frontmatter section shows YAML examples
- [ ] `docs/plans/` directory is present

---

## What this does NOT include (deliberate scope cuts)

- **qmd/semantic search wiring** — `/foundry-ask` currently scans files directly. Wiring it to qmd requires knowing the user's qmd MCP configuration. Scoped out; implement once we confirm qmd is available and configured.
- **Inbox bridge from miniflux/linkding** — a cron that exports starred items to `inbox/` is a separate project. The ingest command already handles any `.md` file dropped into inbox; the bridge just automates the drop.
- **Prompt injection hardening** — treating source bodies as untrusted requires wrapping ingest in a sandboxed read step. This needs a design decision about how Claude reads vs. processes source content. Scoped out pending that decision.
