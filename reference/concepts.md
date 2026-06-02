# Knowledgizer — the methodology

Knowledgizer turns a pile of source documents into a **cited, cross-linked
knowledge wiki** that an LLM (and a human) can query reliably. It is the
generalization of the `cbio-kb` cancer-genomics workflow into a domain-agnostic
pattern driven by one config file.

## Mental model

```
  raw sources                compiled pages              the graph
  (data/raw/…)   ──compile──▶  wiki/<source>/{id}.md  ──entities──▶  wiki/<kind>/{id}.md
                                      │                                     │
                                      └──────────── crosslink ◀────────────┘
                                                     │
                              synthesize ▶ wiki/themes/{slug}.md
                              query      ▶ wiki/explorations/{ts}-{slug}.md
                              lint       ▶ structural audit (read-only)
```

Two kinds of node:

- **Source pages** — one per input document, structured into fixed sections,
  every claim citing the source itself.
- **Entity pages** — one per graph node (the `entity_kinds` in config), each
  accreting claims from every source that mentions it. The entity page is the
  cross-document synthesis; the citation back to a source is the provenance.

## The roles (agents)

| Agent | Job | Reads | Writes |
|---|---|---|---|
| **source-compiler** | Compile one raw source into a structured page; return the entities it found | `raw_dir`, schema | `wiki/<source>/{id}.md` |
| **entity-page-writer** | Create/merge one entity page from its citing sources | source pages, template | `wiki/<kind>/{id}.md` |
| **crosslinker** | Rewrite bare entity mentions as markdown links to canonical pages | wiki | edits wiki pages |
| **theme-synthesizer** | Synthesize a cross-source theme; record agreements/conflicts | source pages | `wiki/themes/{slug}.md` |
| **wiki-querier** | Answer a question against the wiki, file a cited answer | wiki, index | `wiki/explorations/…` |
| **wiki-linter** | Audit structure (frontmatter, links, orphans, sections) | wiki | nothing (read-only) |
| **claims-extractor** | (opt-in) Emit verifiable structured claims as sidecar JSON | one source page | `{id}.claims.json` |

## Invariants that make it trustworthy

1. **Cite everything.** Every assertion ends with a link to the source that
   supports it. Blank beats guess.
2. **Never invent canonical ids.** Genes, codes, study ids — only use ones the
   ontology or source declares. Unverified terms go to an `_observed` holding
   pen, never into canonical pages.
3. **Idempotent.** Re-running the compiler overwrites a source page; the
   entity-writer merges (de-dup by source id). Reruns are safe.
4. **Deterministic post-steps own shared state.** No agent edits the index or
   graph manifest — the orchestrator runs the deterministic build commands
   once at the end.
5. **Markdown links only.** Never Obsidian `[[wiki]]` syntax — it breaks
   Quarto/pandoc publishing.
6. **Provenance on every page.** `processed_by` / `processed_at` frontmatter
   plus an italic footer naming the last agent to touch the page.

## Token discipline

The wiki can grow to thousands of pages, so reads dominate cost. When the
project config supplies a `wiki_cli`, agents use it for structure
(`outline`, `properties`, `search-context`, `backlinks`, `files`) and then
**narrow-Read** only the lines they need. Full-file Reads and Grep/Glob inside
the wiki are last resorts. This single habit is the difference between a loop
that scales to a large corpus and one that doesn't.

## Why agents, not just skills

The add-batch loop fans out the source-compiler in parallel waves (default 5 at
a time), each in its own isolated context. That parallelism and context
isolation is what keeps a large ingest affordable — it only works with
dispatchable subagents. The orchestrator skill drives the waves; the agents do
the bounded work.
