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
| `knowledgizer-setup` | Interactively build `knowledgizer.config.yaml` and scaffold the wiki/templates for a new domain. The "start-up step." |
| `knowledgizer` | Orchestrate the loops: add a source, add a batch, synthesize a theme, query the KB, reprocess after changes. |

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

## How it's configured

Each knowledge base has one `knowledgizer.config.yaml` at its repo root defining
the entity kinds, page templates, source sections, ontology rules, and optional
tooling. The generic agents read it first and carry no hardcoded domain
assumptions — add an entity kind to the config and the whole pipeline extends
with no code change. See `reference/config-schema.md`.

## Design notes

- **Agents, not just skills:** the batch loop fans out the compiler in parallel
  waves, each in an isolated context — that only works with dispatchable
  subagents. The skills orchestrate; the agents do bounded work.
- **Invariants that keep it trustworthy:** cite everything, never invent
  canonical ids, idempotent reruns, deterministic post-steps own shared state,
  single-bracket markdown links only, provenance on every page.

## Status

v0.1.0 — foundation. Expected to iterate, especially on how far the
domain-specific bits (claim schemas, ontology discipline) push into config
versus per-project hard-rules files.

## License

MIT
