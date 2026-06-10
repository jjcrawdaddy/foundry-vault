Ingest raw material into the Foundry vault.

**Before doing anything**, read `CLAUDE.md` at the vault root for the full rules of engagement. The summary below is a prompt, not the source of truth.

### What to ingest

1. Check `inbox/` for anything unprocessed. Files can be:
   - Raw `.md` clippings from Obsidian Web Clipper
   - PDFs — extract text
   - Plain URLs in a `.md` file — fetch them
   - `obsidian://open?vault=JAC&file=<encoded-title>` links — resolve to a JAC file and read it (see below)
   - Pasted text — treat as an article

**Resolving `obsidian://` links.** If the inbox file contains a link of the form `obsidian://open?vault=JAC&file=<encoded-title>`, resolve it to the companion vault filesystem path per CLAUDE.md: URL-decode the `file=` parameter and read the file at `<JAC path>/<decoded title>.md`. Treat the content of that JAC note as the source material. The source note in `sources/` should cite the original with `source: "[[JAC/<decoded title>]]"` (not a web URL). The JAC file is never modified.

2. If `$ARGUMENTS` contains a URL or path, ingest that specifically instead of scanning the inbox.

### For each item

1. **Idempotency check.** Before creating a source note, compute the SHA-256 of the source content (full text before summarisation):
   ```bash
   printf '%s' "<source_text>" | shasum -a 256
   ```
   Grep `sources/` for any note already containing that hash value. If a match exists, skip and log `already ingested: <matched title>`. Do not create a duplicate.

2. **Normalise into a source note** at `sources/<Title>.md`. Title is Title Case of the source.

3. **Fill front-matter** per CLAUDE.md. Write valid YAML between `---` delimiters. Required fields:
   - `tags:` — YAML list. Include `type/source`, the appropriate `area/...`, and all relevant `keyword/...` tags (**no `#` prefix** — just `type/source`, `area/craft/ai`, etc.)
   - `date_created:` — today as ISO string (e.g. `2026-06-09`)
   - `source:` — URL or canonical reference
   - `source_hash:` — see next step
   - `primary:` — `false` by default; see step 6

   **Read the Keywords section of `wiki/_meta/index.md` first.** Reuse existing keywords. If adding a new one, register it there with a one-line definition.

4. **Compute `source_hash`.** Use the same content you hashed in step 1:
   ```bash
   printf '%s' "<source_text>" | shasum -a 256
   ```
   Store the result as `source_hash: sha256:<hexdigest>`. This fingerprint is written once on creation and never updated. Lint uses it to detect if the immutable source note body has been edited after ingest.

5. **Write a concise summary** — what's the source saying in 3-5 sentences? What's the core claim?

6. **Extract key points** as a bulleted list — 5-10 items maximum. Be tight.

7. **Add Claude's notes** — one short paragraph on what's interesting, where it connects, what it contradicts. This is where the compile loop will look later.

8. **Create a people page** if the source has an author and no page exists in `wiki/` yet. Keep it thin — a connector node, not an essay. See CLAUDE.md for the three-tier rule.

   **Set `primary: true`** if the source is an authoritative singleton — a government publication, official plan document, canonical spec — rather than commentary or a secondary account. Sources with `primary: true` bypass the 2-source rule in the compile loop and can seed a concept article alone.

9. **Cross-reference the companion vault** (if one is configured in CLAUDE.md): scan for notes on the same topic. If any match, mention them in Claude's notes using `[[VaultName/Path/Title]]` style links. Never write into the companion vault.

10. **Clear the inbox**: remove the original once the source note is written.

### Logging

Append to `wiki/_meta/log.md`:

```
## [YYYY-MM-DD] ingest | <one-line descriptor>
- Sources: [[Title]] — one-line descriptor
- Noticed: anything surprising — candidate themes, unusual keywords, contradictions with existing notes
```

Also update the relevant sections of `wiki/_meta/index.md`:
- Add the new source to the Sources catalog
- Increment keyword counts
- If a recurring theme appeared across 2+ sources, add it to Candidates for the compile loop to promote

### After all items are ingested

Run `qmd update` to keep the semantic index current:

```bash
qmd update 2>&1
```

If qmd errors or is unavailable, log the failure in `wiki/_meta/log.md` under the ingest entry as `qmd update failed: <error>` — do not block the ingest or treat it as fatal.

### Hard rules

- Never write into the companion vault (if configured). Read only.
- Don't create concept articles in this command — that's the compile loop's job. Just get the source notes in clean.
- If the inbox is empty and no argument is given, say so and stop. Don't invent work.

### Arguments

`$ARGUMENTS` — optional. A URL, file path, or keyword. If present, ingest that specific thing. If empty, scan the inbox.
