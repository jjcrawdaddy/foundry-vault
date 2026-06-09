Compile source notes in the Foundry vault into concept articles.

**Before doing anything**, read `CLAUDE.md` at the vault root for the full rules of engagement.

### Scope

Scan `sources/` for source notes that haven't been fully compiled. A source is "un-compiled" if its title doesn't appear in any concept article's `sources:` YAML list across `wiki/`. To check: grep `wiki/` recursively for `"[[<source-title>]]"` inside a `sources:` block.

If `$ARGUMENTS` names a specific source note, concept, or theme, focus the compile run on that. Otherwise work through un-compiled sources oldest first.

### For each un-compiled source

1. **Read the source note** in full, including Claude's notes at the bottom.
2. **Scan existing concept articles** in `wiki/` for topical overlap. If a companion vault is configured, also check it for related first-person notes.
3. **Decide**: extend an existing concept, or spin out a new one.
   - **Extend** when the source adds a new example, counter-example, or nuance to an existing concept. Add the source to the `sources:` YAML list. Add a paragraph or bullet under **Evidence across sources** that cites the new source.
   - **Spin out** when a distinct idea recurs across 2+ sources that doesn't fit existing concepts. A new concept article needs YAML frontmatter per CLAUDE.md:
     - `tags:` — `type/concept`, appropriate `area/...`, and `keyword/...` tags (no `#` prefix)
     - `date_created:` — today as ISO string
     - `sources:` — YAML list of wikilinks to the source notes it synthesises
     - `related:` — YAML list of `{page, rel}` objects. Minimum 2 entries. `rel` must be one of: `extends`, `contradicts`, `supersedes`, `see-also`. Use `contradicts` when sources genuinely disagree — do not collapse the tension into one reading. Use `see-also` as the default when no stronger relationship applies.

     **Picking `rel` values:**
     - `extends` — this concept builds on or deepens the related one
     - `contradicts` — core claim is in genuine tension; name the disagreement explicitly in the body under a **Tensions** subheading
     - `supersedes` — new evidence renders the related concept obsolete; note it in the log
     - `see-also` — real relationship that doesn't fit the above (use as default)

     Body sections: "What it is" > "Why it matters" > "Key points" > "Evidence across sources" > "Open questions" > "Prompts". If zero related concepts exist, note why it's an island in the body (expect lint to flag it).

4. **If uncertain**: log the theme as a candidate in the Candidates section of `wiki/_meta/index.md` with a one-line rationale. Don't create speculative concepts from a single source — wait for cross-source signal.

   **Primary-source exception.** If a source note has `primary: true` in its frontmatter, you may spin out a new concept article from it without waiting for a second source. At the top of the concept body, add:
   > Single primary source — watch for corroboration.

   Use this only for authoritative documents (official specifications, government publications, primary legal texts) where a second independent source may never meaningfully exist.

### Cross-vault synthesis

If a companion vault is configured in CLAUDE.md, check it for notes on the same topic. If a relevant note exists:
- Add it to the concept's `sources:` YAML list as `"[[VaultName/Path/Note Title]]"`
- Attribute in the body: *"In a note on X, the author argues…"*
- Never copy verbatim — paraphrase.

### Maintenance passes

After compiling:

1. **Update the Research Threads section of `wiki/_meta/index.md`** — if a new thread has formed (a cluster of ≥3 concept articles sharing keywords), add a section for it with links. If a thread gained new articles, update its list.
2. **Append to the Prompts section of `wiki/_meta/index.md`** — roll any new essay prompts from the compiled concept articles into the vault-wide aggregate.
3. **Keywords** — register any new keywords used during compile in the Keywords section of `wiki/_meta/index.md`, and increment counts.
4. **Update `wiki/_meta/log.md`** — append today's entry under `## [YYYY-MM-DD] compile | <summary>`, listing new concepts and extended ones.

### Output

End with a short summary printed to the user:
- Sources compiled (count + titles)
- New concepts created (titles + one-line each)
- Existing concepts extended (titles)
- Candidates flagged (titles + reason)

### After compile is complete

Run `qmd update` to index newly created or extended concept articles:

```bash
qmd update 2>&1
```

If qmd errors or is unavailable, log `qmd update failed: <error>` in the compile log entry — do not treat it as fatal.

### Hard rules

- Never write into the companion vault (if configured).
- Never create a concept article from a single source. Default is to wait for 2+ sources.
- Always update backlinks when renaming or restructuring. Never leave broken links.
- If you delete or substantially rewrite an existing concept, note it in the log under a **Restructured** bullet.

### Arguments

`$ARGUMENTS` — optional. A specific source note title, concept name, or theme to focus the compile run on. If empty, compile all un-compiled sources.
