![Foundry](docs/header.png)

# Foundry

A personal knowledge vault maintained by Claude. You collect sources. Claude compiles them into a wiki of concepts, connections, and open questions.

Based on [Andrej Karpathy's LLM wiki pattern](https://x.com/karpathy/status/2039805659525644595): raw source documents go in, an LLM incrementally compiles them into concept articles with backlinks, and the wiki becomes a rich substrate for Q&A and further research.

## Quick start

1. **Clone this repo** and open it in [Obsidian](https://obsidian.md)
2. **Install Claude Code** — [claude.ai/code](https://claude.ai/code)
3. **Drop something into `inbox/`** — a URL, a web clipping (via [Obsidian Web Clipper](https://obsidian.md/clipper)), a PDF, or pasted text
4. **Run `/foundry-ingest`** — Claude processes everything in the inbox into clean source notes
5. **Run `/foundry-compile`** — Claude scans for un-compiled sources and builds concept articles when themes recur across 2+ sources
6. **Run `/foundry-ask`** followed by a question — Claude researches your question across the vault and writes a report

That's it. Drop, ingest, compile. The vault grows itself.

---

## How it works

### Three layers

| Directory | Purpose | Who writes |
|---|---|---|
| `inbox/` | Staging for unprocessed drops | You |
| `sources/` | One atomic note per article, paper, or transcript | Claude |
| `wiki/` | Concept articles, people pages, query reports, index, and log | Claude |

### The contract

You curate what goes in. Claude writes and maintains the derived layer. The separation matters — your taste decides what's worth keeping; Claude does the synthesis work that humans can't sustain at scale.

### The 2-source rule

Claude won't create a concept article from a single source. Themes are logged as **candidates** in `wiki/_meta/index.md` until a second independent source confirms the pattern. This prevents the wiki from filling up with speculative one-offs.

**Exception:** sources tagged `primary: true` (government publications, official plan documents, canonical specs) can seed a concept article alone. See [Authoritative sources](#authoritative-sources-primary-flag) below.

### What compounds

- **Sources** build the raw material
- **Concepts** crystallise when themes recur across 2+ sources
- **Queries** (`/foundry-ask`) produce research reports that get filed back into the wiki — questions add up over time
- **Research threads** emerge as clusters of 3+ concepts sharing keywords

---

## Commands

| Command | What it does |
|---|---|
| `/foundry-sync` | Pull queued bookmarks from Linkding into `inbox/` |
| `/foundry-ingest` | Process inbox into source notes |
| `/foundry-compile` | Build or extend concept articles from un-compiled sources |
| `/foundry-ask` | Research a question across the vault |
| `/foundry-lint` | Health-check: stats, orphans, keyword drift, Linkding + qmd status |

---

## Linkding integration (optional)

Linkding is a self-hosted bookmark manager. Foundry can pull bookmarks directly from its REST API into `inbox/` so you never need to manually copy URLs.

### Setup

1. **Find your Linkding URL** — the base URL of your Linkding instance, e.g. `http://192.168.1.10:9090`. No trailing slash.

2. **Generate an API token** — in Linkding: Settings → API Keys → Generate API token.

3. **Add both to `.claude/settings.json`** in the vault root (this file is gitignored):

   ```json
   {
     "env": {
       "LINKDING_URL": "http://192.168.1.10:9090",
       "LINKDING_TOKEN": "your-api-token-here"
     }
   }
   ```

   These env vars are picked up automatically by every Foundry command when you run Claude Code from the vault directory. You don't need to export them to your shell.

### Tagging workflow

Tag bookmarks in Linkding to queue them for the Foundry:

| Tag | Effect |
|-----|--------|
| `foundry` | Queue for ingest. Tag is removed after `/foundry-sync` so the bookmark is only processed once. |
| `foundry-primary` | Queue for ingest AND mark the source as authoritative (sets `primary: true`). See [Authoritative sources](#authoritative-sources-primary-flag). |

All other tags on the bookmark are preserved. Only `foundry` and `foundry-primary` are stripped on sync.

### Usage

```
/foundry-sync        # pull all foundry-tagged bookmarks into inbox/
/foundry-ingest      # process inbox stubs into source notes (fetches full text)
```

Sync writes a minimal stub to `inbox/` — just the URL, title, and your Linkding notes. Full text is fetched during ingest, which also computes the source hash.

---

## qmd integration (optional)

[qmd](https://github.com/tobilu/qmd) provides semantic (vector + BM25) search over the vault. `/foundry-ask` queries qmd first to surface relevant files before falling back to index scanning — this matters once your vault grows past ~50 source notes.

### Setup

1. **Install qmd** (requires Node.js):

   ```bash
   npm install -g @tobilu/qmd
   ```

   If you hit a native module error after a Node.js upgrade, rebuild:

   ```bash
   cd $(npm root -g)/@tobilu/qmd && npm rebuild
   ```

2. **Register the vault as a qmd collection** — run this once from anywhere:

   ```bash
   qmd collection add foundry /path/to/your/foundry-vault
   ```

3. **Build the initial index:**

   ```bash
   qmd embed
   ```

   This generates vector embeddings for all files in the collection. Run `qmd update` (not `qmd embed`) for incremental updates going forward — Foundry does this automatically after every `/foundry-ingest` and `/foundry-compile` run.

### Verifying the setup

```bash
qmd status           # shows index health and collection list
qmd collection list  # confirms the foundry vault is registered
```

Or run `/foundry-lint` — Check 12 reports qmd availability, collection registration, and index freshness.

---

## Authoritative sources (primary flag)

Some sources are authoritative singletons — an official plan document, an IRS publication, a canonical spec — where waiting for a "second independent source" doesn't make sense. Tag these `foundry-primary` in Linkding (or set `primary: true` manually in the inbox stub) and Claude will promote them to a concept article immediately rather than waiting.

The concept article will include a notice: `> Single primary source — watch for corroboration.`

---

## Companion vault (optional)

If you already keep personal notes (a commonplace, Zettelkasten, or notes folder), you can point Claude at it as a read-only companion. Claude will cross-reference your notes when compiling but never write into them. See `CLAUDE.md` for setup.

---

## Customisation

- **Areas**: Edit the `#area/` taxonomy in `CLAUDE.md` to match your interests
- **Keywords**: Managed in `wiki/_meta/index.md` — add new ones as your reading grows
- **Obsidian theme**: The `.obsidian/` config is included with a clean setup. Swap themes or plugins as you like
- **Voice**: Concept articles are written as sharp foundations for *your* writing, not finished essays. Adjust the voice instructions in `CLAUDE.md` if you want a different tone
- **Dataview**: Frontmatter uses standard YAML, so Dataview queries work out of the box. Example: `TABLE date_created FROM "sources" WHERE contains(tags, "type/source") SORT date_created DESC`

---

## What's included

This template ships with two example sources and one example concept to show the structure:

- `sources/LLM Knowledge Bases (Karpathy).md` — the post that inspired this vault pattern
- `sources/What Matters in the Age of AI Is Taste.md` — on taste as the human differentiator in AI-augmented work
- `wiki/Two-layer knowledge systems.md` — a concept article compiled from both sources
- `wiki/Andrej Karpathy.md` — an example people page

Delete these and clear `wiki/_meta/index.md` once you've seen how they work, or keep them as seeds for your own vault.

Dump anything — different subjects, different formats, no need to organise by topic first. Karpathy's original design is explicitly a single vault for everything you find interesting.

The vault handles the sorting: each source gets its own note in `sources/`, and concept articles in `wiki/` only crystallise when a theme recurs across two independent sources. So unrelated topics just sit as source notes until enough cross-source signal accumulates to warrant a page. You might have 10 sources on retirement planning and 3 on soil science — they stay separate, each building its own candidate pool and eventual concept graph.

The one thing that does require thought is the `#area/` tag on each source. That's how you segment your reading life — `area/finances`, `area/garden`, `area/craft/ai`, whatever fits. The taxonomy is yours to define in `CLAUDE.md`. The keyword layer beneath it is free-form and self-organising. Everything else is automatic.

---

## Schema

Everything Claude needs to know is in `CLAUDE.md`. That file is the single source of truth for how the vault operates — frontmatter schema, tag taxonomy, operation rules, and integration config. Read it to understand or modify the rules.

---

Built with [Claude Code](https://claude.ai/code) and [Obsidian](https://obsidian.md).
