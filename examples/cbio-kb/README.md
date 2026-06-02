# Example: cbio-kb (cancer genomics)

This is the original [`cbio-kb`](https://github.com/jim-bo/cbio-kb) workflow
expressed as a knowledgizer domain config. It shows how the generic, agnostic
plugin instantiates to a real, opinionated knowledge base.

## How the original maps onto knowledgizer

| Original cbio-kb concept | knowledgizer field |
|---|---|
| `paper-compiler` agent | `source-compiler` (id_field = `pmid`) |
| genes / cancer_types / datasets / drugs / methods pages | `entity_kinds` |
| `wiki/papers/{pmid}.md` sections | `source.sections` |
| `uv run cbio-kb wiki …` token-efficient reads | `tooling.wiki_cli` |
| `cbio-kb lint` / `build-index` / `build-graph` / `normalize-brackets` | `tooling.*_cmd` |
| `[PMID:NNN](../papers/NNN.md)` citations | `links.citation_template` |
| `sample_count` / `mutation_rate` sidecars | `claims.kinds` (opt-in) |
| HUGO / OncoTree discipline | `entity_kinds[].id_format` + `ontology_file` |

## Using it

1. Install the `cbio-kb` CLI (it is **not** bundled — plugins can't ship
   executables):
   ```bash
   uv tool install cbio-kb        # or: uv sync within the cbio-kb repo
   ```
2. Copy `knowledgizer.config.yaml` to the root of your `cbio-kb` checkout.
3. In Cowork/Claude with the knowledgizer plugin installed, the **knowledgizer**
   orchestrator skill now drives the same loops the original `AGENTS.md`
   described:
   - "add paper 39506116" → source-compiler → entity-page-writer → crosslinker
     → normalize → build-index → build-graph → lint.
   - "add this batch of PMIDs" → parallel waves of 5, alphabetical gene shards.

## What stays cbio-kb-specific (and lives here, not in the generic plugin)

The denominator validity gate for `mutation_rate` claims (tier-mixed / subset /
superset cohorts, the PMID 21252315 worked example) is cancer-genomics domain
knowledge. The generic `claims-extractor` references the principle; the concrete
rules belong in this project's `hard_rules_file` (`schema/CLAUDE.md`) and the
`claims.kinds[].verify` notes above.
