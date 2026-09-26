# agents-market/spec

> Vendor-neutral, public, versioned format specs for the AI-agent skill/pipeline ecosystem.
> One repo, three specs — any marketplace or runner can implement them.

## Specs

| Spec | Path | What |
|------|------|------|
| **SKILL.md** | [`skill/1.0.0/`](skill/1.0.0/SPEC.md) | One file, one skill: inputs, outputs, pricing, author, behavior. |
| **Pipeline** | [`pipeline/1.0.0/`](pipeline/1.0.0/PIPELINE_SPEC.md) | Multi-stage workflows: DAG, skills, conditions, validation, composition. |
| **MCP tools** | [`mcp/1.0.0/`](mcp/1.0.0/MCP_TOOLS.md) | 22-tool surface for discover → invoke → manage (skills, pipelines, wallet). |

Each spec is versioned independently (`<spec>/<semver>/`). v1.0.0 of each is
frozen: additive changes ship as v1.x, breaking changes as v2.0.

## Layout

```
spec/
├── README.md                  ← you are here
├── LICENSE (MIT)
├── CONTRIBUTING.md
├── CHANGELOG.md
├── skill/1.0.0/
│   ├── SPEC.md                ← SKILL.md format definition
│   ├── examples/              ← conforming SKILL.md examples
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

- Pipeline format: [`pipeline/1.0.0/PIPELINE_SPEC.md`](pipeline/1.0.0/PIPELINE_SPEC.md)
- Pipeline JSON Schema: [`pipeline/1.0.0/schema.json`](pipeline/1.0.0/schema.json)
- Output-schema subset: [`pipeline/1.0.0/output_schema.json`](pipeline/1.0.0/output_schema.json)
- SKILL.md format: [`skill/1.0.0/SPEC.md`](skill/1.0.0/SPEC.md)
- MCP tool index: [`mcp/1.0.0/MCP_TOOLS.md`](mcp/1.0.0/MCP_TOOLS.md)

## Versioning

- `1.0.0` (2026-09-26) — initial extraction: pipeline format core +
  schemas + examples, SKILL.md v1.0 placement, MCP 22-tool index.
- See [`CHANGELOG.md`](CHANGELOG.md) for details.

## Contributing

Single-vendor governance for now (see [`CONTRIBUTING.md`](CONTRIBUTING.md)):
issues and small PRs welcome; spec direction is set by the maintainer and
may evolve toward shared governance as adoption grows.

## License

MIT — see [`LICENSE`](LICENSE).
