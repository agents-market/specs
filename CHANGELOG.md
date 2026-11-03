# Changelog

## v1.1.0 (2026-10-06)

SKILL.md aligned with the [Agent Skills](https://agentskills.io) open standard.
This is a **breaking change for authors** of v1.0 skills, and a no-op for
readers.

**Why:** v1.0 defined the format independently. The Agent Skills standard,
implemented by 45+ clients, permits exactly six top-level frontmatter fields —
`name`, `description`, `license`, `compatibility`, `metadata`, `allowed-tools`.
v1.0 skills violate that in two ways, and the violation is enforced, not
advisory: `skills-ref validate` rejects them, and Anthropic's Skills API /
`package_skill.py` reject them at upload time with

```
Unexpected key(s) in SKILL.md frontmatter: price_usdc.
Allowed properties are: allowed-tools, compatibility, description, license,
metadata, name
```

Verified empirically: all 8 example skills in the `agents-market` main repo
failed validation before this change.

**Breaking:**

- A skill is now a **directory** containing `SKILL.md`, not a single file.
  `name` must match the directory name. Optional `scripts/`, `references/`,
  and `assets/` directories are defined.
- Extension fields move from top level into `metadata` under the
  `io.agentsmarket/` reverse-DNS prefix. All values are quoted strings; the
  standard defines `metadata` as `map<string, string>`, and nested maps are not
  part of that contract.
- List-valued fields become comma-separated strings — `tags`, `granted_to`.
- `allowed-tools` becomes a space-separated string, not a YAML sequence
  (clarified upstream in issue #520).
- `is_paid` / `is_free` are removed: `price_usdc` of `0` already means free.
- `description` limit raised from 200 to 1024 characters to match the standard.

**Added:**

- Progressive-disclosure tiers and the `<available_skills>` tier-1 catalog
  format.
- `.agents/skills/` as the cross-client install convention.
- The portable / executable split, keyed on the presence of `executor_url`.
- Migration section from v1.0, and guidance that consumers stay lenient
  (warn and load) where authors are strict.
- Two conforming examples, as directories.

**Unchanged:** `pipeline/1.0.0/` and `mcp/1.0.0/`. `skill/1.0.0/` is retained
on disk, frozen and unmodified, for reference and migration.

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
