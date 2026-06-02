---
name: knowledgizer
description: >
  Orchestrate building and maintaining a cited, cross-linked knowledge wiki from
  source documents. Use when the user wants to "add a source/paper/document",
  "ingest a batch", "rebuild the wiki", "synthesize a theme", "ask the knowledge
  base", or "reprocess after a template/ontology change" in a repo that has a
  knowledgizer.config.yaml. Dispatches the source-compiler, entity-page-writer,
  crosslinker, theme-synthesizer, wiki-querier, wiki-linter, and claims-extractor
  agents.
metadata:
  version: "0.1.0"
---

You are the orchestrator for a knowledgizer knowledge base. You own cross-cutting
state (the index and graph); the agents do bounded, isolated work and you fan
them out. Read `${CLAUDE_PLUGIN_ROOT}/reference/concepts.md` for the model.

## Always first

1. Read `knowledgizer.config.yaml` at the repo root. If it's missing, tell the
   user to run the **knowledgizer-setup** skill first — do not improvise a
   config.
2. Note `wiki_dir`, `raw_dir`, `source`, `entity_kinds`, `tooling`, and
   `orchestration` (model pinning, wave size, shard size). Substitute these into
   every command and dispatch below.
3. Dispatch agents with the model pinned per `orchestration.model_pinning`.

## Workflow: add one source

1. Ensure the raw document exists at `<raw_dir>/<source.folder>/<id>.md` (run the
   project's extract/ingest step if it has one).
2. Dispatch **source-compiler** with the id. It writes the source page and
   returns entity lists keyed by `entity_kinds[].id`.
3. For each entity kind with new/touched entries, dispatch **entity-page-writer**
   once per entity with its `{entity: [citing_ids]}` mapping.
4. Dispatch **crosslinker** over the new source page plus every entity page
   touched in step 3 (explicit path list).
5. Run the deterministic post-steps if configured: `<normalize_cmd>`, update any
   News/changelog the project keeps, then `<build_index_cmd>` and
   `<build_graph_cmd>`.
6. Run `<lint_cmd>` (or dispatch **wiki-linter**) and fix structural errors.
7. Commit.

## Workflow: add a batch of N sources

1. Run the ingest/extract step once for all new documents.
2. Dispatch **source-compiler** in parallel waves of
   `orchestration.batch_wave_size` ids. On transient overloads, retry failed
   ids sequentially — do not abandon the wave.
3. Collect every returned entity list and invert to
   `{kind: {entity: [ids...]}}` (a short Python one-liner).
4. Fan out **entity-page-writer** once per entity. For high-cardinality kinds,
   shard alphabetically into buckets of `orchestration.gene_shard_size` to keep
   prompts tight.
5. Single **crosslinker** pass over the new source pages + all touched entity
   pages.
6. `<normalize_cmd>` → update News → `<build_index_cmd>` → `<build_graph_cmd>`.
7. `<lint_cmd>`; commit per wave, not per source.

## Workflow: synthesize a theme

Dispatch **theme-synthesizer** with `theme` + `slug`. Afterward run
`<build_index_cmd>` if the project's index lists themes.

## Workflow: ask the knowledge base

Dispatch **wiki-querier** with the question. It files a cited answer under
`<wiki_dir>/explorations/`.

## Workflow: reprocess after ontology/template changes

Pick the cheapest tier that covers the change:
- **Frontmatter only** (new/renamed template fields): run the project's
  deterministic frontmatter reprocessor if it has one; otherwise a targeted
  entity-page-writer pass.
- **Delta extraction** (expanded ontology, e.g. a new entity kind): re-dispatch
  source-compiler only for the affected fields/sources.
- **Full recompile** (structural template overhaul): re-dispatch source-compiler
  per source (expensive — last resort).

## Workflow: verifiable claims (opt-in)

Only if `claims.enabled`. Dispatch **claims-extractor** per source to write the
sidecar, then run the project's verifier/test-emit steps. This is NOT part of
the standard per-source loop.

## Rules of thumb

- **No agent edits the index or graph.** You run the deterministic build
  commands once at the end. This prevents merge conflicts across parallel agents.
- **Idempotency:** source-compiler overwrites (safe rerun); entity-page-writer
  merges (de-dups by source id).
- **Ontology discipline:** never invent canonical ids; unverified terms go to
  the ontology `_observed` holding pen, not into canonical pages.
- **Provenance:** every page carries `processed_by`/`processed_at` and the
  footer. If a page lacks them after an agent runs, treat it as a defect.
- **Links:** if `<normalize_cmd>` exists, always run it before lint — it repairs
  any Obsidian-style links agents may have slipped in. Otherwise dispatch
  wiki-linter and fix `obsidian_links` findings by hand.
- **Missing tooling is fine.** If a `tooling.*` command is blank, skip that step
  and rely on the agents' Read/Edit fallbacks.
