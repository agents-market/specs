# Changelog

## v1.0.0 (2026-09-26)

Initial extraction from the `agents-market` main repo into vendor-neutral,
versioned specs.

**Pipeline (`pipeline/1.0.0/`):**

- `PIPELINE_SPEC.md` — portable format core (~750 lines) extracted from
  main-repo `PIPELINE.md` (v0.10 lineage). Keeps: frontmatter schema,
  inputs/output/quality/limits/cost/memory blocks, full stage model (LLM,
  `uses`, `uses_parallel`, `loop`, `shell`-as-extension, multimodal
  `messages`), template interpolation, env-var substitution, retry/fallback,
  `when` DSL, stage output refs, checkpoints, parallel + fan-out,
  composition (`uses: pipeline:`, depth 4), `output_schema` subset +
  `output_retry`, MCP fields, provenance shape, streaming events, tool-skill
  namespace. Strips: CLI flags, server endpoints, marketplace/payment
  mechanics, storage backends, cost formulas, per-sprint changelogs.
- `schema.json` — pipeline YAML JSON Schema (draft-07). All 5 bundled
  examples validate against it.
- `output_schema.json` — meta-schema for the `output_schema` draft-07
  subset (only supported keywords allowed).
- `examples/` — 5 representative pipelines: `code-review-security-audit`,
  `dependency-scanner`, `performance-review`, `style-review`,
  `tts-synthesizer`.
- `conformance-tests/` — runner conformance notes (placeholder).

**Skill (`skill/1.0.0/`):**

- SKILL.md v1.0 format (`SPEC.md`), examples, and reference converter
  tools moved from the repo root into versioned layout (history preserved
  via `git mv`).

**MCP (`mcp/1.0.0/`):**

- `MCP_TOOLS.md` — 22-tool index (8 skill, 10 pipeline, 4 wallet) with
  one-line contracts + conformance notes.
- `README.md` — surface overview and non-goals.

**Repo-level:**

- `README.md` rewritten as index of the three specs.
- License set to MIT (was CC0 on the pre-extraction repo).
- `CONTRIBUTING.md` — single-vendor governance (may evolve).
