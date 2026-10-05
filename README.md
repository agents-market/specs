# agents-market/spec

> Vendor-neutral, public, versioned format specs for the AI-agent skill/pipeline ecosystem.
> One repo, three specs — any marketplace or runner can implement them.

## Specs

| Spec | Path | What |
|------|------|------|
| **SKILL.md** | [`skill/1.1.0/`](skill/1.1.0/SPEC.md) | One directory, one skill: instructions, scripts, pricing, safety. Conforms to the [Agent Skills](https://agentskills.io) open standard. |
| **Pipeline** | [`pipeline/1.0.0/`](pipeline/1.0.0/PIPELINE_SPEC.md) | Multi-stage workflows: DAG, skills, conditions, validation, composition. |
| **MCP tools** | [`mcp/1.0.0/`](mcp/1.0.0/MCP_TOOLS.md) | 22-tool surface for discover → invoke → manage (skills, pipelines, wallet). |

Each spec is versioned independently (`<spec>/<semver>/`). Each released version
is frozen: additive changes ship as v1.x, breaking changes as v2.0. Superseded
versions stay on disk for reference — read the version table below to pick the
right one.

## Version status

| Spec | Current | Superseded | Notes |
|------|---------|------------|-------|
| SKILL.md | **1.1.0** | 1.0.0 | v1.1 aligns with the Agent Skills standard. v1.0 placed extension fields at top level and does **not** validate — see the migration section. |
| Pipeline | 1.0.0 | — | |
| MCP tools | 1.0.0 | — | |

### SKILL.md v1.0 → v1.1

v1.1 is required to distribute skills. A v1.0 skill fails the standard's
reference validator and is rejected by the Anthropic Skills API at upload:

```
Unexpected fields in frontmatter: price_usdc. Only
['allowed-tools', 'compatibility', 'description', 'license', 'metadata', 'name']
are allowed.
```

To migrate, move each extension field into `metadata` under the
`io.agentsmarket/` prefix and quote the values. Readers should keep accepting
v1.0 top-level fields during the transition.

## Layout

```
spec/
├── README.md                  ← you are here
├── LICENSE (MIT)
├── CONTRIBUTING.md
├── CHANGELOG.md
├── skill/1.1.0/
│   ├── SPEC.md                ← SKILL.md format (Agent Skills standard + extensions)
│   └── examples/              ← conforming skills, one directory each
├── skill/1.0.0/
│   ├── SPEC.md                ← superseded, retained for reference
│   ├── examples/              ← flat .md examples (pre-directory format)
│   └── tools/                 ← reference converters (OpenAPI → SKILL.md)
├── pipeline/1.0.0/
│   ├── PIPELINE_SPEC.md       ← pipeline format core (~750 lines)
│   ├── schema.json            ← pipeline YAML JSON Schema (draft-07)
│   ├── output_schema.json     ← output_schema subset meta-schema (draft-07)
│   ├── examples/              ← 5 representative pipeline YAMLs (schema-validated)
│   └── conformance-tests/     ← runner conformance notes
└── mcp/1.0.0/
    ├── MCP_TOOLS.md           ← 22-tool index + conformance notes
    └── README.md
```

## Quick links

- SKILL.md format: [`skill/1.1.0/SPEC.md`](skill/1.1.0/SPEC.md)
- Agent Skills standard: <https://agentskills.io> — the format v1.1 conforms to
- Pipeline format: [`pipeline/1.0.0/PIPELINE_SPEC.md`](pipeline/1.0.0/PIPELINE_SPEC.md)
- Pipeline JSON Schema: [`pipeline/1.0.0/schema.json`](pipeline/1.0.0/schema.json)
- Output-schema subset: [`pipeline/1.0.0/output_schema.json`](pipeline/1.0.0/output_schema.json)
- MCP tool index: [`mcp/1.0.0/MCP_TOOLS.md`](mcp/1.0.0/MCP_TOOLS.md)

## Versioning

- `1.1.0` (2026-10-06) — SKILL.md aligned with the Agent Skills open standard:
  directory-as-unit, six allowed top-level fields, extensions namespaced under
  `io.agentsmarket/` inside `metadata`, progressive-disclosure catalog format,
  `.agents/skills/` install convention, updated conformance criteria.
- `1.0.0` (2026-09-26) — initial extraction: pipeline format core +
  schemas + examples, SKILL.md v1.0 placement, MCP 22-tool index.
- See [`CHANGELOG.md`](CHANGELOG.md) for details.

## Contributing

Single-vendor governance for now (see [`CONTRIBUTING.md`](CONTRIBUTING.md)):
issues and small PRs welcome; spec direction is set by the maintainer and
may evolve toward shared governance as adoption grows.

## License

MIT — see [`LICENSE`](LICENSE).
