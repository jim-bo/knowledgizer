---
name: knowledgizer-setup
description: >
  Stand up a new knowledgizer knowledge base by interactively building its
  domain config and scaffolding. Use when the user says "set up knowledgizer",
  "start a new knowledge base", "configure knowledgizer", "create the domain
  config", "add a domain", or is in a repo that has no knowledgizer.config.yaml
  yet. In a workspace (knowledgizer.workspace.yaml) it scaffolds a new domain
  that extends the shared core layer.
metadata:
  version: "0.2.0"
---

Guide the user through standing up a knowledgizer knowledge base in the current
repo. The deliverable is a valid `knowledgizer.config.yaml` plus the directory
scaffolding and starter templates the agents expect. The full config field
reference is in `${CLAUDE_PLUGIN_ROOT}/reference/config-schema.md`; read it
before generating the file. The methodology is in
`${CLAUDE_PLUGIN_ROOT}/reference/concepts.md`.

## First: detect state

Check, at the repo root, for `knowledgizer.workspace.yaml` and
`knowledgizer.config.yaml`.
- **Workspace file exists:** this repo shares a core layer across domains. Do
  not write a root config. Summarize the listed domains and offer to (a) **add
  a new domain** (see "Workspace: add a domain" below) or (b) edit one
  domain's config (edit mode, with that domain folder as the KB root).
- **Root config exists (no workspace):** switch to **edit mode**: summarize the
  current config and ask what to change, rather than overwriting.
- **Neither:** run the wizard below for a single knowledge base. Mention in one
  line that if they expect several corpora to share people, genes, methods and
  the like, a workspace with a shared core layer is an option
  (`reference/config-schema.md`, "Workspaces and the core layer"); proceed
  with the single-KB flow unless they ask for it.

## Wizard — ask in plain language, one topic at a time

Use AskUserQuestion. Do not dump schema jargon on the user; translate each
field into a plain question. Gather:

1. **What is this knowledge base about?** → `domain` + `description`. Offer a
   few example domains (research literature, legal cases, internal docs,
   product/competitor intel) to make the question concrete.
2. **What is one input document?** → `source.id_field` (its stable unique id),
   the `raw_dir` they'll drop documents into, and `source.folder`. Suggest
   `pmid` for papers, `case_id` for legal, `doc_id`/`slug` otherwise.
3. **What are the node types of the graph?** → `entity_kinds`. For each: a name,
   a folder, and how to recognize a valid id (`id_format`). Seed with sensible
   defaults for the stated domain, then let them add/remove.
4. **What sections should every source page have?** → `source.sections`. Offer a
   default set (Summary, Key findings, Details, Implications, Limitations) and
   let them tailor.
5. **Is there a wiki CLI for this project?** → `tooling.*`. Most new projects
   have none — that's fine; leave the tooling commands blank and the agents
   fall back to narrow Reads. Only fill these if the user already has a
   token-efficient wiki/index CLI (like cbio-kb's). **Remind them plugins
   cannot bundle executables** — any such CLI is installed separately.
6. **Verifiable claims?** → `claims`. Default to `enabled: false`. Only enable
   if the domain has numbers checkable against an external source of truth.

Recommend defaults aggressively; if the user says "whatever you think," propose
a concrete config and confirm before writing.

## Then: write the config and scaffold

1. Read `${CLAUDE_PLUGIN_ROOT}/reference/config-schema.md` and write
   `knowledgizer.config.yaml` at the repo root with the gathered values. Include
   only fields that apply; omit empty optional blocks.
2. Create the scaffolding (use Bash `mkdir -p`):
   - `<wiki_dir>/<source.folder>/`, `<wiki_dir>/themes/`,
     `<wiki_dir>/explorations/`, and one folder per `entity_kinds[].folder`.
   - `<raw_dir>/<source.folder>/` for incoming documents.
   - `<templates_dir>/` and `<schema_dir>/`.
3. Generate starter files in `<schema_dir>`:
   - `hard_rules_file` (e.g. `CLAUDE.md`) stating the non-negotiables: cite
     everything; blank > guess; never invent canonical ids; single-bracket
     markdown links only; provenance on every page.
   - `ontology_file` (`ontology.md`) — a stub for canonical/preferred terms,
     plus an `_observed.md` holding pen for unverified terms.
4. Generate one starter template per entity kind and one
   `source.md` + `theme.md` template under `<templates_dir>`, each with
   frontmatter keys the agents fill (`processed_by`, `processed_at`, entity
   list fields) and the required section headings. Base the source template's
   H2s on `source.sections`.
5. Write a one-line `<wiki_dir>/index.md` placeholder (the project's
   deterministic build step, if any, regenerates it later).

## Workspace: add a domain

When `knowledgizer.workspace.yaml` exists, the new domain is a normal
knowledgizer KB in `domains/<name>/` whose config extends the core schema.
Read the workspace file, `<core.schema>/CLAUDE.md`, `<core.schema>/ontology.md`
and one existing domain config first, and match their conventions.

1. Ask (AskUserQuestion), in plain language: the domain name (kebab, becomes
   the folder and the name in core pages' `domains:` lists), what it covers,
   what one input document is (`source.id_field`, `source.folder`), the source
   sections, which **core kinds** it will cite (default: all of
   `core.kinds`), and any **local kinds** only this domain needs. If a
   proposed local kind is already a core kind, or a local kind in another
   domain, say so: the first should use core; the second is a promotion
   candidate (see `concepts.md`), not a new local kind.
2. Write `domains/<name>/knowledgizer.config.yaml`: the usual fields with
   paths relative to the domain folder, plus
   ```yaml
   extends: ../../core/schema
   entity_kinds:
     - id: person
       core: true
       folder: ../../../core/wiki/people                       # from wiki_dir
       id_format: "kebab slug firstname-lastname exactly as printed"
       template: ../../../../core/schema/templates/person.md   # from templates_dir
     # ... one entry per chosen core kind, folder = its core wiki folder
     - id: <local kind>
       folder: <local folder>
       id_format: "..."
       template: <local kind>.md
   ```
   Take each core kind's folder from `core.wiki` (e.g. `biomarker` →
   `biomarkers`, `person` → `people`) and its template from
   `<core.schema>/templates/<kind>.md`; check every resolved path exists.
3. Scaffold `domains/<name>/`: `wiki/<source.folder>/`, `wiki/themes/`,
   `wiki/explorations/`, one `wiki/<folder>/` per **local** kind only (never
   for a core kind), `data/raw/<source.folder>/`, `schema/templates/`.
4. Starter schema files in `domains/<name>/schema/`: a `CLAUDE.md` that says
   to read the core `CLAUDE.md` first and adds only domain rules (citation
   form from a domain page, link form to core pages
   `../../../../core/wiki/<kind folder>/<id>.md`), an `ontology.md` for local
   kinds, an `_observed.md` holding pen, and templates for the source, theme
   and each local kind. Core kinds use the core templates; do not copy them.
5. Add the domain to `knowledgizer.workspace.yaml` under `domains:` (`name`,
   `path`, `source_folder`, `local_kinds`). Leave the core layer and the other
   domains untouched.
6. Write a one-line `domains/<name>/wiki/index.md` placeholder.

Then orient the user as below, adding that runs in this domain take its folder
as the KB root (`knowledgizer domains/<name>`, or pick it when asked).

## Finally: orient the user

Validate that every `entity_kinds[].template` you referenced now exists. Then
tell the user, in plain language:
- where to drop source documents,
- that they can now say "add this source" / "add these N sources" to trigger
  the **knowledgizer** orchestrator skill,
- and that editing `knowledgizer.config.yaml` (e.g. adding an entity kind)
  reshapes the whole pipeline with no code change.

Do not start ingesting documents in this skill — that's the orchestrator's job.
