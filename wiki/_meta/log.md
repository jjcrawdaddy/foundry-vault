---
tags:
  - type/meta
  - area/meta
date_created: 2026-06-09
---

Append-only chronological record of every Foundry operation. Never rewritten, only appended.

## [2026-06-09] meta | Vault reset
- Cleared example sources and concepts; fresh start with empty catalog
## [2026-06-09] sync | Linkding
- Fetched: 217 bookmarks (217 primary)
- Inbox stubs written: [[post-by-itsolelehmann-on-x.md]], [[tech-stock-singularity-by-john-h-cochrane.md]], [[how-social-media-shortens-your-life-by.md]], [[the-weekend-press-on-melania-and-chugging-milk.md]], [[the-weekend-press-is-staying-married-taboo.md]], [[proteussensorcom.md]], [[the-best-spiral-sliced-ham-americas-test-kitchen.md]], [[reorgs-happen-ben-balter.md]], [[ogretrollecr-mounts-and-nashbar-trailer-mountain-bike.md]], [[the-ai-water-issue-is-fake-andy.md]], ... and 207 more
- Tag-removal failures: 0

## [2026-06-09] meta | Sync correction
- The sync above was faulty: the command spec used `?tag=foundry`, a parameter the Linkding API ignores, so ALL 217 bookmarks were pulled regardless of tag (and all mis-marked primary)
- All 217 wrongly-created inbox stubs deleted; user's manual test.md preserved
- Linkding side verified: other tags intact; 5 bookmarks currently tagged foundry remain queued
- foundry-sync.md and foundry-lint.md fixed to use `?q=%23foundry` search syntax plus a per-bookmark tag_names verification gate

