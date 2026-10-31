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

## The core layer (workspaces)

One KB per corpus works until two corpora share entities: the same person,
gene or method then gets a page in each, and the copies drift. A **workspace**
fixes this with a shared core layer and one domain per corpus (see
`config-schema.md`, "Workspaces and the core layer").

```
  domains/A/wiki/papers/{id}.md ──┐                ┌── domains/B/wiki/papers/{id}.md
  domains/A/wiki/<local kinds>/   │                │   domains/B/wiki/<local kinds>/
                                  ▼                ▼
                    core/wiki/<kind>/{id}.md   (one page per shared entity)
                    core/registry/<kind>.yaml  (id -> name, aliases, domains, sources)
```

- **Shared entities exist once.** Kinds marked `core: true` in a domain config
  live in `core/wiki/<kind folder>/`. Every domain links to the same page;
  no domain creates a core-kind page under its own wiki.
- **Resolve before mint.** The registry maps each canonical id to its name and
  aliases. Before proposing a new core id, agents look the term up there and
  reuse the canonical id on a hit. A new id gets its page *and* its registry
  entry in the same change, so the registry and the core wiki stay in step.
- **`domains:` provenance.** A core page lists every domain whose sources cite
  it (`domains:`) besides the citing source ids (`cited_by:`), so a reader of a
  shared page sees where it matters.
- **Promotion.** A kind starts local to one domain. When a second domain starts
  citing it, promote it: move its pages to `core/wiki/<kind folder>/`, add a
  registry file, mark the kind `core: true` (relative folder and template) in
  both configs, and re-point links. Promotion is a deliberate,
  human-reviewed change, not something an agent does mid-ingest.
- **Cross-domain syntheses live in core.** A theme or exploration citing
  sources from two or more domains goes under `core/wiki/themes/` or
  `core/wiki/explorations/`; otherwise it stays in the domain.

Each run targets one domain: the domain folder is the KB root, and the core
schema's hard rules and ontology are read alongside the domain's (`extends`).

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
   pen, never into canonical pages. In a workspace, resolve core-kind terms
   through the registry first.
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
