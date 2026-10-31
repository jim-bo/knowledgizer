# Example: a workspace with a shared core layer

A minimal workspace: `knowledgizer.workspace.yaml` at the repo root and one
domain config, `domains/trial-matching/knowledgizer.config.yaml`, that extends
the shared core schema. The `core/` tree and the second domain it lists are
not included; the full layout is in `reference/config-schema.md`
("Workspaces and the core layer").

What to notice in the domain config:

- `extends: ../../core/schema` — agents read `core/schema/CLAUDE.md` before
  the domain's own hard rules, and `core/schema/ontology.md` alongside the
  domain ontology.
- `core: true` kinds use relative paths: `folder` from `wiki_dir`
  (`wiki/../../../core/wiki/people` = `core/wiki/people`) and `template` from
  `templates_dir`. Their registry is `core/registry/<folder name>.yaml`.
- `criterion_type` is a local kind and lives in `wiki/criterion_types/`.
- The citation template is written for domain pages; from a core page the same
  source is `../../../domains/trial-matching/wiki/papers/<id>.md`.
