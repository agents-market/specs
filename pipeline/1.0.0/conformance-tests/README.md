# Conformance tests (placeholder)

> How to verify a runner implements `pipeline/1.0.0` correctly.
> Part of [`agents-market/spec`](https://github.com/agents-market/spec).

## Status

Placeholder for v1.0.0. A machine-readable conformance suite (positive +
negative fixtures per spec section) ships in a follow-up release.

## Intended coverage (when implemented)

- `frontmatter/` — required fields, pricing defaults, `cost` ceiling shapes.
- `stages/` — one fixture per stage type (LLM, `uses`, `uses_parallel`,
  `loop`, `shell`-rejection, `messages`).
- `control-flow/` — `retry`, `run_if`, `when` (all operators), stage refs
  (incl. forward-ref failure), checkpoints resume, parallel merge,
  composition (nesting, depth-4 rejection, loose vs strict pins).
- `validation/` — `output_schema` subset (each keyword + each unsupported
  keyword rejection) × `output_retry` modes (`none`/`single`/`loop`).
- `negative/` — every MUST-reject in `PIPELINE_SPEC.md` with the expected
  error class.

## Until then

`examples/` (one level up) are schema-validated positives: any runner MUST
accept all 5. A runner MAY self-certify by running its own test suite
against `schema.json` + `output_schema.json`.
