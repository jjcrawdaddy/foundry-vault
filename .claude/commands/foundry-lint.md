Health-check the Foundry vault.

**Before doing anything**, read `CLAUDE.md` at the vault root for the full rules of engagement.

### What this does

Scan every file in `sources/` and `wiki/` (including `wiki/_meta/`). Validate structure, cross-references, and consistency. Overwrite `wiki/_meta/health.md` with the results. This is the vault's immune system — it catches drift before it compounds.

### Checks to run

#### 1. Stats

Count and report:
- Source notes in `sources/`
- Concept articles in `wiki/` (tags contains `type/concept`)
- Query reports in `wiki/` (tags contains `type/query`)
- People pages in `wiki/` (tags contains `type/person`)
- Total keywords registered in the Keywords section of `wiki/_meta/index.md`
- Total wikilinks across the vault (rough gauge of connectivity)

#### 2. Front-matter validation

Every `.md` file in `sources/` and `wiki/` (except `_meta/`) must have valid YAML frontmatter with `---` delimiters, containing:
- `tags:` — YAML list including exactly one `type/...` entry (`type/source`, `type/concept`, `type/query`, or `type/person`)
- `tags:` — at least one `area/...` entry
- `tags:` — at least one `keyword/...` entry (empty is a warning, not an error)
- `date_created:` — ISO date string (e.g. `2026-04-13`; no `[[` wikilink wrapping)

Additional required fields by type:
- **source**: `source:` (URL or reference), `source_hash:` (any value — absence is an error), `primary:` (bool)
- **concept**: `sources:` (YAML list with at least one `[[...]]` entry), `related:` (YAML list — empty `[]` is flagged in Orphans)
- **query**: `question:` (string)
- **person**: body must contain `**Sources in the Foundry**` and `**Concepts they inform**` sections

Report any file missing required fields or with malformed YAML frontmatter.

#### 3. Orphans

**Concept articles under the 2-related rule**
- Concepts where `related:` is an empty list `[]` or has fewer than 2 entries, AND no island-justification note appears in the body. List each with its current related count and sources count for context.

**Uncited source notes**
- Source notes in `sources/` whose title does not appear in any concept's `sources:` YAML list. These are ingested but never compiled.

**Unreferenced people**
- People pages where both **Sources in the Foundry** and **Concepts they inform** sections are empty or contain only `_(none...)_` markers.

#### 4. Broken links

Scan all wikilinks (`[[...]]`) in `sources/` and `wiki/`. Flag any that point to a file that doesn't exist in the vault. Exclude `[[VaultName/...]]` links that reference a companion vault (those live outside the repo — don't validate them, just skip). Exclude `[[YYYY-MM-DD]]` date links.

#### 5. Index staleness

Cross-check `wiki/_meta/index.md` against reality:
- Files in `sources/` or `wiki/` not listed in the appropriate section of `index.md`
- Entries in `index.md` that reference files which no longer exist
- Keyword count mismatches: recount how many files use each keyword vs the count listed in `index.md`

#### 6. Candidates needing attention

Read the Candidates section of `wiki/_meta/index.md`. Surface:
- **Close to promote**: candidates explicitly marked as close, or where you can verify that a second source has landed since the candidate was logged
- **Stale**: candidates that have been sitting since the vault's earliest logs without progress
- **Ready to fold**: candidates marked as likely-a-section-not-a-concept, still sitting as open candidates

Don't duplicate the full candidates list — just flag the ones that need action.

#### 7. Keyword drift

- **Suspected duplicates**: keywords in the Keywords section of `index.md` that are near-synonyms or overlap semantically (e.g. `focus` / `attention`). Use judgment — flag pairs that a human should decide on, don't flag obvious distinctions.
- **Unregistered keywords**: `keyword/...` tags used in file frontmatter `tags:` list but not listed in the Keywords section of `index.md`.
- **Phantom keywords**: keywords listed in `index.md` but never used in any file's `tags:` list.

#### 8. Source hash drift

For every file in `sources/`:
1. Read `source_hash:` from YAML frontmatter.
2. If the field is **absent entirely**: flag as **Hash field absent** (error).
3. If the value is `sha256:pending`: flag as **Hash not computed** (warning — source predates provenance tracking).
4. If the value is `sha256:<hexdigest>`: re-compute SHA-256 of the note body (everything after the closing `---` of the frontmatter). Use:
   ```bash
   awk '/^---$/{if(++n==2){found=1;next}} found{print}' <file> | shasum -a 256
   ```
   If the recomputed digest does not match the stored hash, flag as **Hash mismatch** (error) — the immutable source note body has been edited since ingest.

Report format (one line per issue):
- `[[Source Title]]` — Hash field absent
- `[[Source Title]]` — Hash not computed (pending)
- `[[Source Title]]` — Hash mismatch: stored `sha256:abc123`, body now hashes to `sha256:def456`

A mismatch is not automatically wrong (you may have corrected a transcription error), but it must be visible. Do not auto-fix — only report.

#### 9. Typed relation vocabulary

For every concept file in `wiki/`, read the `related:` YAML list. For each entry:
1. Verify `page:` is a wikilink pointing to a file that exists in `wiki/`.
2. Verify `rel:` is one of: `extends`, `contradicts`, `supersedes`, `see-also`.

Flag:
- `[[Concept]]` → `[[Target]]` — unknown rel value: `<value>` (valid: extends, contradicts, supersedes, see-also)
- `[[Concept]]` → `[[Target]]` — broken page link (target file does not exist in `wiki/`)

Also flag concept pages where `related:` is an empty list `[]` and no island-justification note appears in the body. (This complements the Check 3 orphan count — it flags the vocab problem separately from the quantity problem.)

#### 10. Citation integrity

For every concept file in `wiki/`, read the `sources:` YAML list. For each wikilink entry:
1. Verify the linked file exists in `sources/`.
2. Verify the linked source note has a `source_hash:` field (even `sha256:pending` is acceptable — total absence is an error).

Flag:
- `[[Concept]]` cites `[[Source Title]]` — source file not found in `sources/`
- `[[Concept]]` cites `[[Source Title]]` — source note missing `source_hash` field

### Writing the report

Overwrite `wiki/_meta/health.md` entirely. Preserve the existing front-matter block:

```yaml
---
tags:
  - type/meta
  - area/craft/meta
date_created: YYYY-MM-DD
---

Single lint dashboard. Overwritten by each `/foundry-lint` run. Do not edit manually — edits get clobbered.
```

Then write each section. Keep the report scannable:
- Use bullet lists, not prose
- Lead each finding with the file name as a `[[wikilink]]`
- If a section has zero findings, write `- _(clean)_` — don't skip the section header

End the Stats section with `- **Last lint run:** YYYY-MM-DD` using today's date.

The report must include sections for all 10 checks. Sections 8–10 template:

```
#### 8. Source hash drift
- _(clean)_

#### 9. Typed relation vocabulary
- _(clean)_

#### 10. Citation integrity
- _(clean)_
```

### Logging

Append to `wiki/_meta/log.md`:

```
## [YYYY-MM-DD] lint | Health check
- Sources: N, Concepts: N, Queries: N, People: N
- Orphans: N, Broken links: N, Index issues: N
- Keyword drift flags: N
- Hash drift: N (pending: N, mismatch: N)
- Relation vocab violations: N
- Citation integrity failures: N
```

### Final summary to print

After writing `health.md` and appending to `log.md`, print a short summary:
- Vault size (sources / concepts / queries / people)
- Issues found (orphans, broken links, index staleness, keyword drift) — count per category
- Candidates needing attention (count + titles)
- If the vault is clean, say so

### Hard rules

- Never write into the companion vault (if configured).
- Never modify any file except `wiki/_meta/health.md` and `wiki/_meta/log.md`.
- Don't fix issues — only report them. Fixes are the job of other commands or the user.
- Don't invent problems. If a check passes, report it as clean.
- If `$ARGUMENTS` is `--dry-run`, print the summary but don't write `health.md` or `log.md`.

### Arguments

`$ARGUMENTS` — optional. `--dry-run` to preview without writing. Otherwise ignored.
