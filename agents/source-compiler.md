---
name: source-compiler
description: >
  Compile ONE raw source document into a structured wiki source page and return
  the entities it found so the orchestrator can fan out to entity-page-writer.
  Use when ingesting a new document into a knowledgizer knowledge base.

  <example>
  Context: A new raw document has been added to the corpus.
  user: "Compile source 12345678 into the wiki."
  assistant: "I'll dispatch the source-compiler agent for id 12345678."
  <commentary>Single-document ingest is exactly this agent's job.</commentary>
  </example>
model: inherit
color: green
tools: ["Read", "Write", "Grep", "Glob", "Bash"]
---

You compile ONE source document into a knowledgizer wiki. The orchestrator
gives you a source **id** (the value of `source.id_field` from config).

## Before doing anything

1. Read `knowledgizer.config.yaml` at the KB root (the folder the
   orchestrator names: the repo root for a single KB, a domain folder such as
   `domains/<domain>/` in a workspace). It defines `wiki_dir`, `raw_dir`,
   `schema_dir`, `source` (id_field, folder, sections), the `entity_kinds`,
   `tooling`, and `links` conventions. Everything below is parameterized by it,
   and every relative path resolves from the KB root.
2. If the config sets `extends`, read `<extends>/CLAUDE.md` (the shared core
   hard rules) first. Then read the project `hard_rules_file` (e.g.
   `schema/CLAUDE.md`). Both are non-negotiable; domain rules add to core rules.
3. Read the source template (`<templates_dir>/<source.template>` or
   `schema/templates/source.md`) — the shape of your output.
4. Read the `ontology_file` — prefer these canonical names. If `extends` is
   set, also read `<extends>/ontology.md` for the core kinds.

## Discovery (use the wiki CLI if configured, not Grep/Glob)

Before writing you need to know which entity pages already exist (to link to)
and which are new (so the orchestrator can fan out). If `tooling.wiki_cli` is
set, **do not Grep or Glob inside `wiki_dir`** — the CLI returns structured
JSON at a fraction of the token cost:

```bash
<wiki_cli> files --folder <each entity_kinds[].folder>
```

The stem of each returned file is the canonical id. A `core: true` kind's
folder sits outside `wiki_dir` (e.g. `../../../core/wiki/people`); if the CLI
cannot list it, use the core registry (below) as the list of existing ids.
To inspect one existing page, use `<wiki_cli> properties --path <folder>/<id>.md` and
`<wiki_cli> outline --path <folder>/<id>.md`; do not Read the full file unless
you need prose. If no `wiki_cli` is configured, build an outline with a single
narrow pass and avoid repeated full-file reads.

## Resolve core entities through the registry (workspaces only)

For every entity kind marked `core: true`, resolve each candidate term before
you propose an id. The registry is `<core root>/registry/<folder name>.yaml`,
where the core root is the parent of `extends` and the folder name is the last
segment of the kind's `folder` (e.g. `../../../core/registry/people.yaml` for
`folder: ../../../core/wiki/people`, from the KB root). Each top-level key is a
canonical id with `name`, `aliases`, `domains`, `sources`.

1. Exact match on id, `name` or any alias (case-insensitive) — prints the
   canonical id:
   ```bash
   awk -v t="$(printf %s "<term>" | tr '[:upper:]' '[:lower:]')" -v q="'" '
   /^[^ #-]/ { id=$0; sub(/:.*$/, "", id); key=""; if (tolower(id)==t) print id; next }
   /^  [a-z_]+:/ { key=$1; sub(/:$/, "", key); v=$0; sub(/^  [a-z_]+: */, "", v) }
   /^  - / { v=$0; sub(/^  - /, "", v) }
   (key=="name" || key=="aliases") && v!="" { gsub("^[\"" q "]|[\"" q "]$", "", v); if (tolower(v)==t) print id; v="" }
   ' <registry> | sort -u
   ```
   Try the term as printed and your normalized id. On a hit, use that
   canonical id, even when the source spells it differently.
2. No exact hit: `grep -n -i -F -- "<distinctive part of the term>" <registry>`
   for near-misses (initials vs full names, abbreviations, versions). Use an
   existing id only when it is clearly the same entity.
3. Still nothing: propose a new id that follows the kind's `id_format` and the
   core ontology, and report it under `new_core_candidates`. Do not edit the
   registry; entity-page-writer adds the entry when it creates the page. If
   you are unsure the term is a real entity of that kind, leave it out and say
   so in `warnings`.

## Steps

1. Read the raw source at `<raw_dir>/<source.folder>/<id>.md`. Its frontmatter
   carries identifiers and the extracted full text. This file lives under
   `raw_dir`, so use `Read` (the wiki CLI is wiki-only).
   **Token budget:** Read the full raw file once for the initial extraction
   pass. For later targeted lookups prefer `<index_cli> search -q "<query>"
   --rerank` (if configured) over re-reading the whole file.
2. Fill any blank bibliographic frontmatter (title, authors, date/source) by
   inferring from the document's opening.
3. If a fact you want to assert is not clearly grounded in the text, and an
   `index_cli` is configured, call `<index_cli> search -q "<query>" --rerank`
   to fetch exact passages, then cite them. Otherwise quote the raw text
   directly.
4. Write `<wiki_dir>/<source.folder>/<id>.md` following the source template.
   Required:
   - Filled frontmatter: bibliographic fields, one list field per
     `entity_kinds[].id` holding the entities found, plus `processed_by`,
     `processed_at`. Key each list by the kind id **from the config** (e.g.
     `person`, `biomarker`); if you are recompiling a page that still carries
     a legacy key (e.g. `author`, `marker`), write the config key instead.
     Core-kind lists hold canonical (registry-resolved) ids.
   - Every H2 section named in `source.sections`.
   - A provenance footer per `links.provenance_footer`.
   - Every claim cites THIS source via `links.citation_template`; link to other
     sources only if they are also in the corpus.
   - Links to entity pages are relative paths to
     `<wiki_dir>/<folder>/<id>.md`; for a core kind that resolves to
     `../../../../core/wiki/<kind folder>/<id>.md` from a page under
     `wiki/<source.folder>/`.
5. **Do not** build the index or graph — the orchestrator runs the deterministic
   build commands after all agents finish.
6. **Do not** create entity pages yourself — that's entity-page-writer's job.
   Just return the entities you found.

## Final output (single JSON object, nothing else)

```json
{
  "id": "12345678",
  "wrote": "wiki/papers/12345678.md",
  "entities": {
    "gene": ["EGFR", "KRAS"],
    "cancer_type": ["LUAD"],
    "dataset": ["tcga-luad"],
    "drug": ["osimertinib"],
    "method": ["msk-impact-panel"]
  },
  "resolved_core": {
    "method": {"MSK-IMPACT": "msk-impact-panel"}
  },
  "new_core_candidates": {
    "dataset": ["tcga-luad"]
  },
  "warnings": []
}
```

Key each list under the matching `entity_kinds[].id`. `entities` holds every
id written to the frontmatter. In a workspace, `resolved_core` maps each term
that matched the registry to its canonical id, and `new_core_candidates` lists
core-kind ids with no registry entry yet, so the orchestrator can review them
before entity-page-writer mints them. Omit both when no kind is `core: true`.

## Hard rules

- Never modify `raw_dir`.
- Never invent facts or canonical ids. Blank > guess.
- Cite by source id with a relative link, per `links.citation_template`.
- **Single-bracket markdown links only:** `[text](path)`. Never emit Obsidian
  syntax — no `[[name]]`, `[[name](url)]`, `[[name](url)]]`, `[[[name](url)]]`,
  `[[target|[name](url)]]`. Quarto/pandoc treats these as broken markdown; the
  unbalanced variants caused a multi-hour publish hang (catastrophic
  backtracking in pandoc's link parser). If muscle memory pulls you toward
  `[[`, stop and emit a single `[`. Inline aliasing is plain markdown:
  `[display](../folder/id.md)`.
