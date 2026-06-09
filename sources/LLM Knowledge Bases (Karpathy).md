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
source_hash: sha256:pending
primary: false
---

**Summary**

Karpathy describes a workflow for building personal knowledge bases with LLMs: raw source documents are ingested into a `raw/` directory, then an LLM incrementally compiles them into a `.md` wiki of summaries, concept articles, and backlinks. Obsidian is the IDE for viewing raw data, compiled wiki, and derived visualisations. Once the wiki reaches sufficient size (~100 articles, ~400K words in his case), it becomes a rich substrate for Q&A and further research, with outputs filed back into the wiki so explorations compound. He argues the LLM should write and maintain the wiki — the human rarely edits it directly.

**Key points**

- Separate `raw/` (immutable source) from the compiled wiki (LLM-owned, regeneratable).
- Obsidian Web Clipper is the preferred ingest mechanism for web articles; a hotkey downloads related images to local.
- Auto-maintained index files and brief summaries replace fancy RAG at this scale — simple works fine.
- Outputs of queries (reports, Marp slides, matplotlib charts) are worth filing back into the wiki so queries "add up" over time.
- Periodic "health checks" surface inconsistencies, missing data, and candidate new articles.
- Custom CLIs (e.g. a naive search engine) become tools the LLM calls for larger queries.
- Long-term direction: synthetic data generation + finetuning so the LLM "knows" the data in its weights, not just context windows.
- Claim: this approach deserves a real product, not a hacky collection of scripts.

**Claude's notes**

This post is the seed text for the Foundry itself — the vault structure, the ingest and compile loops, and the LLM-writes-the-wiki contract all derive directly from it. Worth keeping this note close when reviewing the `CLAUDE.md` rules of engagement.

Open tensions worth compiling later:
- Karpathy says the LLM should write the wiki and the human rarely touches it. Human-curated commonplaces celebrate curation as an act of taste. Where's the line? Which layers does the human own, which does the LLM own, and what's the healthy feedback loop between them?
- At what scale does "simple works fine" break down? Karpathy runs at ~100 articles / ~400K words. What are the failure modes beyond that?
- Marp for slide output, custom search CLIs — both are candidate tools for the Foundry to add later.
