# knowledgizer

Build and maintain a **cited, cross-linked knowledge wiki** from a corpus of
source documents — papers, cases, contracts, transcripts, internal docs. One
config file makes the same agents work for any domain.

Knowledgizer generalizes the [`cbio-kb`](https://github.com/jim-bo/cbio-kb)
cancer-genomics workflow into a domain-agnostic, installable plugin. It is both
a **plugin** and a **marketplace**, so a single git repo is your one place to
update everything.

## Install (per project)

```
/plugin marketplace add jim-bo/knowledgizer
/plugin install knowledgizer@knowledgizer
```

Then in any project run the setup skill once to create the domain config, and
use the orchestrator skill to ingest documents.

> **Note:** plugins cannot bundle executables. If your project uses a
> token-efficient wiki/index CLI (like cbio-kb's `cbio-kb`), install it
> separately and point to it in the config's `tooling` block. Without one, the
> agents fall back to narrow file reads.

## What's in it

**Skills**

| Skill | Use it to |
|---|---|
| `knowledgizer-setup` | Interactively build `knowledgizer.config.yaml` and scaffold the wiki/templates for a new domain, or add a domain to a workspace. The "start-up step." |
| `knowledgizer` | Orchestrate the loops: add a source, add a batch, synthesize a theme, query the KB, reprocess after changes. Takes an optional domain path in a workspace. |

**Agents** (dispatched by the orchestrator, run in isolated context for cheap
parallel fan-out)

| Agent | Job |
|---|---|
| `source-compiler` | Compile one raw document into a structured page; return its entities. |
| `entity-page-writer` | Create/merge one entity page from its citing sources (idempotent). |
| `crosslinker` | Rewrite bare entity mentions as markdown links to canonical pages. |
| `theme-synthesizer` | Synthesize a cross-source theme; record agreements and conflicts. |
| `wiki-querier` | Answer a question against the wiki; file a cited answer. |
| `wiki-linter` | Audit structure (read-only). |
| `claims-extractor` | (Opt-in) Emit machine-verifiable structured claims as sidecar JSON. |

**Reference**

- `reference/config-schema.md` — every field of `knowledgizer.config.yaml`.
- `reference/concepts.md` — the methodology and invariants.

**Example**

- `examples/cbio-kb/` — the cancer-genomics instantiation, mapping each original
  cbio-kb concept onto a config field.
- `examples/workspace-kb/` — a minimal workspace: a workspace file plus one
  domain config that extends a shared core layer.

## How it's configured

Each knowledge base has one `knowledgizer.config.yaml` at its KB root defining
the entity kinds, page templates, source sections, ontology rules, and optional
tooling. The generic agents read it first and carry no hardcoded domain
assumptions — add an entity kind to the config and the whole pipeline extends
with no code change. See `reference/config-schema.md`.

## Layout

A single knowledge base (the KB root is the repo root):

```
<repo>/
  knowledgizer.config.yaml
  schema/{CLAUDE.md,ontology.md,_observed.md,templates/<kind>.md}
  data/raw/<source folder>/          # input documents (agents never edit)
  wiki/<source folder>/<id>.md       # one compiled page per source
  wiki/<kind folder>/<id>.md         # one page per entity
  wiki/themes/  wiki/explorations/
```

A workspace (the KB root for a run is one domain folder):

```
<repo>/
  knowledgizer.workspace.yaml        # lists core + domains
  core/schema/{CLAUDE.md,ontology.md,templates/<kind>.md}
  core/registry/<kind folder>.yaml   # id -> {name, aliases, domains, sources}
  core/wiki/<kind folder>/<id>.md    # people, organizations, diseases, genes, ...
  domains/<domain>/
    knowledgizer.config.yaml         # extends ../../core/schema; core kinds point at core
    schema/  data/raw/
    wiki/<source folder>/  wiki/<local kinds>/  wiki/themes/  wiki/explorations/
```

## Workspaces and the core layer

When several corpora share entities (the same people, genes, methods), keep
them in one repo as a **workspace**: a shared `core/` layer plus one folder per
domain. Each domain is an ordinary knowledgizer KB whose config adds:

- `extends: ../../core/schema` — agents read the core `CLAUDE.md` before the
  domain hard rules, and the core ontology alongside the domain's.
- `core: true` on an entity kind, with a relative `folder`
  (`../../../core/wiki/people`) and `template`. Paths still resolve as
  `wiki_dir/folder`, so core pages land in `core/wiki/` with no special casing.

The agents then resolve every core-kind term through `core/registry/` before
minting an id, write core pages once (with a `domains:` list naming every
citing domain), add the registry entry with each new core page, and put themes
or explorations that span two or more domains under `core/wiki/`. Run the
orchestrator with a domain path (`domains/<name>`), or at the workspace root
and pick a domain. Single-KB configs are unaffected; everything above is
optional. See `reference/config-schema.md` and `examples/workspace-kb/`.

## Design notes

- **Agents, not just skills:** the batch loop fans out the compiler in parallel
  waves, each in an isolated context — that only works with dispatchable
  subagents. The skills orchestrate; the agents do bounded work.
- **Invariants that keep it trustworthy:** cite everything, never invent
  canonical ids, idempotent reruns, deterministic post-steps own shared state,
  single-bracket markdown links only, provenance on every page.

## Status

v0.2.0 — workspaces: a shared core layer (registry, `extends`, `core: true`
kinds) across domains. v0.1.0 — foundation. Expected to iterate, especially on how far the
domain-specific bits (claim schemas, ontology discipline) push into config
versus per-project hard-rules files.

## License

MIT
