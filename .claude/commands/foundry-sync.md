---
description: Pull foundry-tagged bookmarks from Linkding into the inbox
model: haiku
---

Pull queued bookmarks from Linkding into the Foundry inbox.

**Before doing anything**, read `CLAUDE.md` for the full rules of engagement, including the Linkding integration config.

### Pre-flight check

Verify `LINKDING_URL` and `LINKDING_TOKEN` are set:

```bash
echo "URL: ${LINKDING_URL:-NOT SET}"
echo "Token: ${LINKDING_TOKEN:+SET (hidden)}"
echo "Token: ${LINKDING_TOKEN:-NOT SET}"
```

If either is unset, stop and tell the user what to set. Do not attempt API calls.

### Fetch queued bookmarks

**The Linkding API has NO `tag=` parameter — it silently ignores unknown parameters and returns ALL bookmarks.** Tag filtering uses the `q` search parameter with a URL-encoded `#` prefix (`%23`):

```bash
curl -s \
  -H "Authorization: Token $LINKDING_TOKEN" \
  "$LINKDING_URL/api/bookmarks/?q=%23foundry&limit=100"
```

The response is a JSON object: `{"count": N, "results": [...], "next": <url or null>}`. Each result has:
- `id` — numeric bookmark ID
- `url` — the URL to ingest
- `title` — page title (may be empty)
- `description` — user's notes on the bookmark (may be empty)
- `tag_names` — array of all tags on this bookmark

If `next` is non-null, follow it for additional pages.

Also fetch bookmarks tagged `foundry-primary`:

```bash
curl -s \
  -H "Authorization: Token $LINKDING_TOKEN" \
  "$LINKDING_URL/api/bookmarks/?q=%23foundry-primary&limit=100"
```

Merge the two result lists, deduplicating by `id`.

**Sanity gate (mandatory).** Before writing any stubs:

1. Fetch the total bookmark count (`?limit=1`, read `count`). If the filtered result count equals the total count and the total is greater than ~20, the filter is almost certainly not working — **stop and report, write nothing**.
2. For every bookmark in the merged list, verify `tag_names` actually contains `foundry` or `foundry-primary`. **Discard any that don't** — they are false positives from search matching (e.g. the word "foundry" in a title). Never trust the query alone; the tag list on the bookmark itself is the source of truth.

### For each verified bookmark

1. **Determine `primary`**: `true` if the bookmark's own `tag_names` contains `foundry-primary`, `false` otherwise. (Use `tag_names`, not which query returned it.)

2. **Write an inbox stub** at `inbox/<slug>.md`. Slug the title (or URL hostname if title is empty) to kebab-case, max 8 words. File content:

```yaml
---
url: "<bookmark url>"
title: "<bookmark title or empty string>"
notes: "<bookmark description or empty string>"
linkding_id: <numeric id>
primary: <true|false>
---
```

   Do not write a body — `/foundry-ingest` will fetch the URL and generate the summary.

3. **Remove the `foundry` and `foundry-primary` tags** from the bookmark so it isn't re-synced. Compute the new tag list (all existing tags minus `foundry` and `foundry-primary`) and PATCH:

```bash
curl -s -X PATCH \
  -H "Authorization: Token $LINKDING_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"tag_names": ["<remaining tag 1>", "<remaining tag 2>"]}' \
  "$LINKDING_URL/api/bookmarks/<id>/"
```

   If the PATCH fails, log the error and skip (don't lose the inbox stub — the stub is already written, so the content won't be lost). A failed PATCH just means the bookmark will show up again next sync; the idempotency check in `/foundry-ingest` will catch the duplicate.

### Logging

Append to `wiki/_meta/log.md`:

```
## [YYYY-MM-DD] sync | Linkding
- Fetched: N bookmarks (N primary)
- Inbox stubs written: [[stub-1.md]], [[stub-2.md]], ...
- Tag-removal failures: N (if any, list IDs)
```

### Final summary to print

- Bookmarks fetched (count + titles)
- Primary sources (count + titles)
- Inbox stubs written (count)
- Any PATCH failures (IDs)
- If no bookmarks were queued, say so and stop

### Hard rules

- Never write anywhere except `inbox/` and `wiki/_meta/log.md`.
- Never ingest source content in this command — that's `/foundry-ingest`'s job.
- If `LINKDING_URL` or `LINKDING_TOKEN` are unset, stop immediately with a clear message.
- If the API returns a non-200 status, report the error and stop — don't write partial stubs.

### Arguments

`$ARGUMENTS` — ignored. This command always syncs all pending `foundry`-tagged bookmarks.
