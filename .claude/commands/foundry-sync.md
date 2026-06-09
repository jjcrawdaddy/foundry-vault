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

Call the Linkding REST API to list all bookmarks tagged `foundry`:

```bash
curl -s \
  -H "Authorization: Token $LINKDING_TOKEN" \
  "$LINKDING_URL/api/bookmarks/?tag=foundry&limit=100"
```

The response is a JSON object: `{"count": N, "results": [...]}`. Each result has:
- `id` — numeric bookmark ID
- `url` — the URL to ingest
- `title` — page title (may be empty)
- `description` — user's notes on the bookmark (may be empty)
- `tag_names` — array of all tags on this bookmark

Also fetch bookmarks tagged `foundry-primary` (these will be marked `primary: true`):

```bash
curl -s \
  -H "Authorization: Token $LINKDING_TOKEN" \
  "$LINKDING_URL/api/bookmarks/?tag=foundry-primary&limit=100"
```

Merge the two result lists, deduplicating by `id`. Track which IDs came from `foundry-primary`.

### For each bookmark

1. **Determine `primary`**: `true` if this bookmark had the `foundry-primary` tag, `false` otherwise.

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
