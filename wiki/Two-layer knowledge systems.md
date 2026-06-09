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

**What it is**

A personal knowledge system built as two vertically stacked layers: a human-curated source layer (selected by taste, lightly organised, owned by the person) and a derived LLM-authored layer (summaries, concept articles, indexes, Q&A outputs — regeneratable, owned by the machine). The two layers are linked but editorially separate. The human chooses what to collect and what to believe; the LLM compiles, connects, and interrogates.

**Why it matters**

It resolves a tension most "AI for notes" tooling collapses. Pure human curation scales badly at retrieval and synthesis. Pure LLM authorship drifts from the person's actual taste and voice. The two-layer split preserves both: the source layer stays an authentic record of what this specific person found worth keeping; the derived layer does the compiling work that humans can't sustain at scale. It also makes the system regeneratable — the LLM layer can be rebuilt from the human layer, but not the other way around.

**Key points**

- Curation is a human act. Compilation is a machine act. Keep them structurally separate.
- The source layer is a curation of taste; it doesn't need to be structured, only preserved.
- The derived layer is disposable — it can be regenerated any time the source layer changes.
- Simple retrieval (auto-maintained indexes, brief summaries, backlinks) beats fancy RAG at personal scale.
- Outputs of queries compound when filed back into the derived layer, not the source.
- The editorial contract — who can write where — matters more than the tooling. Violate it and both layers degrade.
- The value of the system grows with the size and honesty of the source layer, not the sophistication of the derived layer.

**Evidence across sources**

- [[LLM Knowledge Bases (Karpathy)]] — Karpathy proposes a `raw/` directory of immutable source documents plus an LLM-compiled wiki of summaries, concept articles, backlinks, and indexes. Argues that at ~100 articles / ~400K words, "simple works fine" — auto-maintained index files and brief summaries replace fancy RAG. Outputs of Q&A should be filed back into the wiki so queries "add up" over time. His strong claim: *the LLM writes and maintains the wiki; the human rarely edits it directly.*
- [[What Matters in the Age of AI Is Taste]] — argues that AI output quality is proportional to the quality of context you provide. A curated personal knowledge base is the raw material that teaches AI to see through your eyes. The source layer (what you choose to collect) is an expression of taste; the derived layer (what AI produces from it) can only be as good as that curation. The "organise by context, not topic" principle maps to how the two-layer system works in practice — retrieval patterns matter more than taxonomies.

**Open questions**

- Where exactly does the authorship boundary sit? Karpathy says the LLM owns the wiki; the human owns the source layer. But both acknowledge blurred edges — the human occasionally edits the wiki; the AI's summaries may shape what the human saves next. A sharper rule set for *what each layer must never do* would help.
- What breaks the "simple works fine" threshold? Karpathy runs at ~100 articles / ~400K words. At 10x that scale, do auto-indexes still beat retrieval? Does the LLM-authored layer start hallucinating coherence that isn't there?
- Does the derived layer ever leak back into taste? If Claude's summaries start shaping what you choose to save, the source layer stops being an honest signal. Worth watching for.

**Prompts**

- *Where the authorship line should sit.* The tension between LLM-owns-the-wiki and human-curation-as-taste. A retrospective on where the boundary actually lands in practice — and what each layer must never do.
- *Two-layer as a writing discipline.* The same split (human-curated source, AI-derived output) applied to writing — your notes stay yours, an AI assistant produces derived work (outlines, structure checks, reference recalls) that you never paste in directly.
