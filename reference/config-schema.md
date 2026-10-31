# knowledgizer.config.yaml — schema

Every knowledgizer knowledge base has one config file at its **KB root**:
`knowledgizer.config.yaml`. It is the single source of truth that makes the
generic agents domain-specific. Agents and skills **read this file first** and
never hardcode domain assumptions. The setup wizard (`knowledgizer-setup`
skill) generates it interactively.

The KB root is usually the repo root. In a **workspace** (several domains
sharing one core entity layer, see [below](#workspaces-and-the-core-layer-optional))
the KB root is a domain folder such as `domains/<domain>/`, and every relative
path in its config resolves from that folder.

If a field is absent, agents fall back to the documented default.

```yaml
# ─── Identity ────────────────────────────────────────────────────────────
domain: cancer-genomics            # human-readable label for this knowledge base
description: >                      # one paragraph: what this KB covers and for whom
  Structured wiki of cBioPortal cancer-genomics papers, linking genes,
  cancer types, datasets, drugs, and methods to their citing literature.

# ─── Layout ──────────────────────────────────────────────────────────────
# All paths are relative to the KB root (the folder holding this config).
wiki_dir: wiki                     # where compiled pages live (the "vault")
raw_dir: data/raw                  # source documents to compile FROM (read-only to agents)
schema_dir: schema                 # templates + ontology rules + hard-rules doc
templates_dir: schema/templates    # one template per entity kind + source + theme
hard_rules_file: schema/CLAUDE.md  # project-specific hard rules every agent reads first
ontology_file: schema/ontology.md  # canonical names / preferred terms

# ─── Source documents ────────────────────────────────────────────────────
# A "source" is one unit of input (a paper, a case, a contract, a transcript).
source:
  id_field: pmid                   # canonical id key (pmid | case_id | doc_id | slug)
  folder: papers                   # subfolder under wiki_dir for compiled source pages
  raw_glob: "papers/*.md"          # where raw source docs live under raw_dir
  sections:                        # required H2 sections in a compiled source page
    - TL;DR
    - Cohort & data
    - Key findings
    - Genes & alterations
    - Clinical implications
    - Limitations & open questions

# ─── Entity kinds ────────────────────────────────────────────────────────
# The node types of the knowledge graph. Each becomes a folder of wiki pages.
entity_kinds:
  - id: gene
    folder: genes
    id_format: "HUGO gene symbol (uppercase)"      # how to validate/normalize an id
    template: gene.md
  - id: cancer_type
    folder: cancer_types
    id_format: "OncoTree code"
    template: cancer_type.md
  - id: dataset
    folder: datasets
    id_format: "cBioPortal studyId or kebab slug"
    template: dataset.md
  - id: drug
    folder: drugs
    id_format: "lowercase generic drug name"
    template: drug.md
  - id: method
    folder: methods
    id_format: "kebab slug"
    template: method.md

# ─── Tooling (optional but recommended) ──────────────────────────────────
# A token-efficient, READ-ONLY CLI over the wiki. Agents prefer it over
# Read/Grep/Glob for structure. Leave blank if the project has none; agents
# then fall back to narrow Reads. Plugins cannot bundle executables — install
# this separately (see README).
tooling:
  wiki_cli: "uv run cbio-kb wiki"        # outline / properties / search-context / backlinks / files
  index_cli: "uv run cbio-kb index"      # passage-level semantic search over raw docs (optional)
  lint_cmd: "uv run cbio-kb lint --wiki-dir wiki"
  build_index_cmd: "uv run cbio-kb wiki build-index"
  build_graph_cmd: "uv run cbio-kb wiki build-graph"
  normalize_cmd: "uv run cbio-kb wiki normalize-brackets"
  append_citation_cmd: "uv run cbio-kb wiki append-citation"
  crosslink_cmd: "uv run cbio-kb crosslink"

# ─── Linking & citation conventions ──────────────────────────────────────
links:
  style: markdown                  # ALWAYS single-bracket markdown: [text](path). Never [[wiki]] links.
  citation_template: "[{ID_FIELD}:{id}]({rel_path})"   # how every claim cites a source
  provenance_footer: "*This page was processed by **{agent}** on **{date}**.*"

# ─── Claims (optional, advanced) ─────────────────────────────────────────
# Structured, machine-verifiable assertions the claims-extractor emits as
# sidecar JSON. Omit this whole block to disable claims extraction.
claims:
  enabled: false                   # opt-in; not part of the standard pipeline
  sidecar_suffix: ".claims.json"
  kinds:                           # each kind = a verifiable claim shape
    - id: sample_count
      fields: [study, value, denominator, quote, section]
      verify: "compare value to the data source's cohort size"
    - id: mutation_rate
      fields: [study, gene, modifier, value, denominator, quote, section]
      verify: "compare value to samples_with_feature / cohort_size"

# ─── Orchestration defaults ──────────────────────────────────────────────
orchestration:
  model_pinning:                   # cost/quality tradeoff per agent
    source-compiler: opus
    entity-page-writer: sonnet
    crosslinker: haiku
    claims-extractor: sonnet
    theme-synthesizer: opus
    wiki-querier: opus
    wiki-linter: haiku
  batch_wave_size: 5               # parallel source-compiler dispatches per wave
  gene_shard_size: 20              # alphabetical sharding for high-cardinality entity kinds
```

## Field reference (essentials)

- **`source.id_field`** — the primary key for an input document. Drives file
  names (`{id}.md`), citation links, and de-duplication. cbio-kb uses `pmid`;
  a legal KB might use `case_id`; an internal-docs KB `doc_id`.
- **`entity_kinds`** — the only entity types agents will create pages for.
  Adding a kind here (folder + template + id_format) extends the graph with no
  code change.
- **`tooling.wiki_cli`** — if present, agents call `<wiki_cli> outline/properties/
  search-context/backlinks/files` for cheap structured reads instead of full
  Reads. If absent, agents use narrow Reads against the outline they build
  themselves.
- **`links.style`** — must stay `markdown`. Obsidian `[[wiki]]` links break
  Quarto/pandoc rendering; the crosslinker and compiler agents enforce
  single-bracket links.
- **`claims`** — leave `enabled: false` unless the domain has API- or
  data-verifiable quantitative assertions worth round-tripping through a
  verifier.

## Validation checklist (what a well-formed config guarantees)

1. `wiki_dir`, `raw_dir`, `schema_dir` exist (or will be created by setup).
2. Every `entity_kinds[].template` exists under `templates_dir`.
3. `source.id_field` is a stable, unique key for each input document.
4. `hard_rules_file` exists and states the domain's non-negotiables
   (never invent canonical ids, blank > guess, cite everything).
5. If `tooling.*` commands are set, the underlying CLI is installed and on PATH.
6. If `extends` is set, `<extends>/CLAUDE.md` and `<extends>/ontology.md`
   exist, and every `core: true` kind's `folder` resolves (from `wiki_dir`) to
   an existing folder under the core wiki, with a matching
   `<core>/registry/<folder name>.yaml`.

## Workspaces and the core layer (optional)

Everything in this section is optional. A config without `extends` and without
`core: true` kinds is a plain single-KB config and behaves exactly as above.

A **workspace** is one repo holding a shared **core layer** (entities that
exist once, such as people, diseases, genes, methods) plus several **domains**,
each a normal knowledgizer KB in its own folder. The plugin runs against one
domain at a time: the domain folder is the KB root.

```
<repo>/
  knowledgizer.workspace.yaml        # lists core + domains
  core/
    schema/CLAUDE.md                 # shared hard rules (read before domain rules)
    schema/ontology.md               # confirmed ids for the core kinds
    schema/templates/<kind>.md       # one template per core kind
    registry/<kind folder>.yaml      # id -> {name, aliases, domains, sources}
    wiki/<kind folder>/<id>.md       # one page per core entity
    wiki/themes/  wiki/explorations/ # cross-domain syntheses (created on first use)
  domains/<domain>/
    knowledgizer.config.yaml         # a normal config plus `extends` and core kinds
    schema/  data/raw/
    wiki/<source folder>/  wiki/<local kinds>/  wiki/themes/  wiki/explorations/
```

### The workspace file

`knowledgizer.workspace.yaml` sits at the repo root. The orchestrator reads it
to pick a domain, and agents read it to name the current domain and to find
other domains' source folders when linking from core pages.

```yaml
core:
  path: core
  schema: core/schema
  registry: core/registry
  wiki: core/wiki
  kinds: [person, organization, disease, gene, biomarker, therapy, dataset, method, metric, task]

domains:
  - name: trial-matching            # the name used in core pages' `domains:` lists
    path: domains/trial-matching    # the KB root for runs in this domain
    source_folder: papers           # its source.folder, for cross-domain source links
    local_kinds: []                 # kinds that live only in this domain
  - name: immunoprofile
    path: domains/immunoprofile
    source_folder: papers
    local_kinds: [cell_type, platform]
```

The **domain name** is the workspace entry whose `path` is the KB root (fall
back to the folder name). It is not the config's free-text `domain:` label.

### `extends`

```yaml
extends: ../../core/schema          # relative to the KB root
```

Points at the core schema folder. When set:

- Agents read `<extends>/CLAUDE.md` **before** the domain `hard_rules_file`
  (domain rules add to core rules, never relax them), and read
  `<extends>/ontology.md` **in addition to** the domain `ontology_file`.
- The **core root** is the parent of `extends` (`../../core`). The registry is
  `<core root>/registry/`, the core wiki is `<core root>/wiki/`.

### `core: true` on an entity kind

```yaml
entity_kinds:
  - id: person                      # kind id, also the source-page frontmatter key
    core: true
    folder: ../../../core/wiki/people                       # relative to wiki_dir
    id_format: "kebab slug firstname-lastname as printed"
    template: ../../../../core/schema/templates/person.md   # relative to templates_dir
  - id: cell_type                   # a local kind: no `core`, plain folder
    folder: cell_types
    id_format: "kebab slug of the cell population"
    template: cell_type.md
```

The usual path rule still holds: a page lives at `<wiki_dir>/<folder>/<id>.md`
and its template at `<templates_dir>/<template>`. Because `folder` and
`template` are relative, `wiki/` + `../../../core/wiki/people` lands on
`core/wiki/people` and `schema/templates/` + `../../../../core/schema/templates/person.md`
lands on the core template. Agents keep joining paths this way; `core: true`
adds the rules that go with a shared page:

- **Registry:** `<core root>/registry/<last segment of folder>.yaml`
  (e.g. `people.yaml`), mapping each canonical id to
  `{name, aliases, domains, sources}` where `sources` is the number of citing
  source ids. Resolve every candidate id here before minting; a new core page
  gets its registry entry in the same change.
- **Frontmatter:** a core page carries `domains:` (every domain whose sources
  cite it) besides `cited_by:` (the source ids).
- **Links:** a core page links to a source as
  `../../../domains/<domain>/wiki/<source folder>/<id>.md`; a domain page under
  `wiki/<folder>/` links to a core page as
  `../../../../core/wiki/<kind folder>/<id>.md`. Both are just the relative
  path between the two files.
- **Placement:** a core-kind page never sits under the domain's `wiki_dir`.

Core kind ids in the reference layout are `person` (folder `people`),
`organization`, `disease`, `gene`, `biomarker` (folder `biomarkers`),
`therapy`, `dataset`, `method`, `metric` and `task`. A domain lists only the
core kinds it extracts.

### Tooling in a workspace

Tooling commands run from the KB root unless they say otherwise. A workspace
may ship repo-level checks (e.g. `tools/check_links.py`,
`tools/check_registry.py`) that run from the repo root; agents use them when
`tooling.lint_cmd` is unset.
