---
description: Health-check the vault — structure, links, hashes, integrations
argument-hint: [--dry-run]
model: haiku
---

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

Validate every `.md` file in `sources/` and `wiki/` (except `_meta/`) against the frontmatter schema defined in CLAUDE.md (the Front-matter section) — required fields per note type, valid YAML, correct field formats.

Severity notes:
- Missing `keyword/...` tag: warning, not error
- Missing `source_hash:` or `note_hash:` on a source: error (any value including `sha256:pending` is acceptable — only absence is flagged)
- Empty `related: []` on a concept: flagged in Orphans, not here
- Person pages: body must contain `**Sources in the Foundry**` and `**Concepts they inform**` sections

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

#### 8. Note hash drift

For every file in `sources/`, validate the **`note_hash:`** field (NOT `source_hash:` — that is the raw-content fingerprint used by ingest idempotency, and it will never match the note body. See CLAUDE.md for the two fields' distinct roles.):

1. Read `note_hash:` from YAML frontmatter.
2. If the field is **absent entirely**: flag as **note_hash absent** (error).
3. If the value is `sha256:pending`: flag as **note_hash not computed** (warning — note predates drift tracking, or ingest failed to finalise it).
4. If the value is `sha256:<hexdigest>`: re-compute SHA-256 of the note body (everything after the closing `---` of the frontmatter). Use:
   ```bash
   awk '/^---$/{if(++n==2){found=1;next}} found{print}' <file> | shasum -a 256
   ```
   If the recomputed digest does not match the stored `note_hash`, flag as **Note drift** (error) — the immutable source note body has been edited since ingest.

Also check `source_hash:` for absence or `sha256:pending` (same severities), but never compare it against the body.

Report format (one line per issue):
- `[[Source Title]]` — note_hash absent
- `[[Source Title]]` — note_hash not computed (pending)
- `[[Source Title]]` — Note drift: stored `sha256:abc123`, body now hashes to `sha256:def456`
- `[[Source Title]]` — source_hash absent / pending

Note drift is not automatically wrong (you may have corrected a transcription error), but it must be visible. Do not auto-fix — only report.

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

#### 11. Linkding connectivity

Check whether the Linkding integration is configured and healthy.

1. **Config check**: Verify `LINKDING_URL` and `LINKDING_TOKEN` are set in the environment.
   - If either is unset: flag as **Linkding not configured** (info, not an error — integration is optional).
   - If both are set, proceed.

2. **Connectivity check**: Ping the API with a minimal request:
   ```bash
   curl -s -o /dev/null -w "%{http_code}" \
     -H "Authorization: Token $LINKDING_TOKEN" \
     "$LINKDING_URL/api/bookmarks/?limit=1"
   ```
   - `200` → healthy
   - `401` → token invalid or expired
   - `000` or other → host unreachable or URL misconfigured

3. **Pending bookmarks**: If healthy, count bookmarks tagged `foundry` or `foundry-primary`. Note: the Linkding API has no `tag=` parameter — use the `q` search parameter with URL-encoded `#` (`%23`):
   ```bash
   curl -s \
     -H "Authorization: Token $LINKDING_TOKEN" \
     "$LINKDING_URL/api/bookmarks/?q=%23foundry&limit=1" | python3 -c "import sys,json; print(json.load(sys.stdin)['count'])"
   ```
   Report the count. Zero is fine. Non-zero means `/foundry-sync` has work waiting.

Report format:
- `Linkding not configured` — LINKDING_URL or LINKDING_TOKEN not set
- `Linkding: ✓ reachable — N bookmarks pending sync`
- `Linkding: ✗ auth failed (HTTP 401)`
- `Linkding: ✗ unreachable (HTTP 000)`

#### 12. qmd index status

Check whether qmd is available and its index covers the vault.

1. **Availability check**:
   ```bash
   qmd status 2>&1
   ```
   - If `command not found`: flag as **qmd not in PATH**.
   - If exits with a Node.js module error (e.g. `ERR_DLOPEN_FAILED`, `NODE_MODULE_VERSION`): flag as **qmd broken — native module mismatch**. Include the fix: `cd $(npm root -g)/@tobilu/qmd && npm rebuild`.
   - If exits cleanly: qmd is available.

2. **Collection check**: Verify the vault root is registered as a qmd collection:
   ```bash
   qmd collection list 2>&1
   ```
   If no collection covers the vault path, flag as **Vault not indexed by qmd** — suggest `qmd collection add foundry <vault-path> && qmd embed`.

3. **Index freshness**: Compare the `qmd status` last-updated timestamp to the newest file in `sources/` and `wiki/`. If any file is newer than the last index update, flag as **qmd index stale — run `qmd update`**.

Report format:
- `qmd not in PATH`
- `qmd broken — native module mismatch (run: cd $(npm root -g)/@tobilu/qmd && npm rebuild)`
- `qmd: ✓ available, vault indexed, index current`
- `qmd: ✓ available — index stale (N files newer than last update)`
- `qmd: ✓ available — vault not registered as a collection`

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

The report must include sections for all 12 checks. Sections 8–12 template:

```
#### 8. Note hash drift
- _(clean)_

#### 9. Typed relation vocabulary
- _(clean)_

#### 10. Citation integrity
- _(clean)_

#### 11. Linkding connectivity
- _(not configured)_

#### 12. qmd index status
- _(unavailable)_
```

### Logging

Append to `wiki/_meta/log.md`:

```
## [YYYY-MM-DD] lint | Health check
- Sources: N, Concepts: N, Queries: N, People: N
- Orphans: N, Broken links: N, Index issues: N
- Keyword drift flags: N
- Note hash drift: N (pending: N, drift: N)
- Relation vocab violations: N
- Citation integrity failures: N
- Linkding: <reachable / not configured / error>
- qmd: <available+current / stale / broken / unavailable>
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
