# Contributing to agents-market/spec

Thanks for your interest in improving these specs.

## Governance

Single-vendor for now: spec direction is set by the `agents-market` org.
This may evolve toward shared governance (editors, RFC process) as adoption
grows — that change itself will be proposed here, in the open.

## Process

1. **Open an issue first** — discuss the change before writing spec text.
2. **PR against `main`** — keep changes atomic and well-justified.
3. **Maintain backward compatibility** — additive only within `1.x`.

## What we accept

- ✅ Bug fixes (typos, broken examples)
- ✅ Clarifications (reword for precision)
- ✅ Additive optional fields (new YAML keys / new tools)
- ✅ New examples (must validate against the schemas)
- ✅ Better error/edge-case descriptions
- ✅ Conformance test contributions

## What requires a major version bump

- ❌ Breaking changes to required fields
- ❌ Changes to section/stage semantics
- ❌ Removing fields or tools
- ❌ Renaming tools in `mcp/`

## Style

- Clear, precise language (this spec is implemented by machines).
- One sentence per line where practical (easier diffs).
- Examples must conform to the specs themselves — pipeline examples MUST
  validate against `pipeline/1.0.0/schema.json`.
- Justify non-obvious changes in the PR description.

## Release process

- MAJOR bump (`1.0` → `2.0`): breaking change.
- MINOR bump (`1.0` → `1.1`): additive change.
- PATCH bump (`1.0` → `1.0.1`): clarification/typo.

Implementations of v1.0 MUST accept v1.x files (forward compat for
additive fields). New spec versions ship as new directories
(`pipeline/1.1.0/`, …) — old versions are frozen, never edited in place.

## License

By contributing, you agree to release your contributions under the MIT
License (see `LICENSE`).
