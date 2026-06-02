---
name: knowledgizer-setup
description: >
  Stand up a new knowledgizer knowledge base by interactively building its
  domain config and scaffolding. Use when the user says "set up knowledgizer",
  "start a new knowledge base", "configure knowledgizer", "create the domain
  config", or is in a repo that has no knowledgizer.config.yaml yet.
metadata:
  version: "0.1.0"
---

Guide the user through standing up a knowledgizer knowledge base in the current
repo. The deliverable is a valid `knowledgizer.config.yaml` plus the directory
scaffolding and starter templates the agents expect. The full config field
reference is in `${CLAUDE_PLUGIN_ROOT}/reference/config-schema.md`; read it
before generating the file. The methodology is in
`${CLAUDE_PLUGIN_ROOT}/reference/concepts.md`.

## First: detect state

Check whether `knowledgizer.config.yaml` exists at the repo root.
- If it exists, switch to **edit mode**: summarize the current config and ask
  what to change, rather than overwriting.
- If not, run the wizard below.

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

## Finally: orient the user

Validate that every `entity_kinds[].template` you referenced now exists. Then
tell the user, in plain language:
- where to drop source documents,
- that they can now say "add this source" / "add these N sources" to trigger
  the **knowledgizer** orchestrator skill,
- and that editing `knowledgizer.config.yaml` (e.g. adding an entity kind)
  reshapes the whole pipeline with no code change.

Do not start ingesting documents in this skill — that's the orchestrator's job.
