# Pipeline Specification v1.0.0

> A vendor-neutral format for multi-stage AI workflows that orchestrate
> skills, models, and tools to produce a final output.
> One file, one pipeline, any runner.
>
> Companion to [`../skill/1.0.0/SPEC.md`](../skill/1.0.0/SPEC.md) (SKILL.md format).

**Status:** `1.0.0` (2026-09-26). Extracted as the portable format core of the
pipeline format (v0.1 → v0.10 lineage). Runtime, marketplace, and payment
details are intentionally out of scope — see [Non-goals](#non-goals).

**Conformance keywords:** `MUST`, `MUST NOT`, `SHOULD`, `MAY` are normative.

---

## Table of contents

1. [What is this](#1-what-is-this)
2. [Design principles](#2-design-principles)
3. [Quick example](#3-quick-example)
4. [Frontmatter schema](#4-frontmatter-schema)
5. [Inputs block](#5-inputs-block)
6. [Output block](#6-output-block)
7. [Quality and limits blocks](#7-quality-and-limits-blocks)
8. [Cost ceiling block](#8-cost-ceiling-block)
9. [Memory block](#9-memory-block)
10. [Stages and the DAG](#10-stages-and-the-dag)
11. [Stage types](#11-stage-types)
12. [Template interpolation](#12-template-interpolation)
13. [Environment variable substitution](#13-environment-variable-substitution)
14. [Retry and fallback](#14-retry-and-fallback)
15. [Conditional branches: `when` DSL](#15-conditional-branches-when-dsl)
16. [Stage output refs](#16-stage-output-refs)
17. [Checkpoints](#17-checkpoints)
18. [Parallel stages](#18-parallel-stages)
19. [Composition](#19-composition)
20. [Stage output validation](#20-stage-output-validation)
21. [MCP forward-compatibility fields](#21-mcp-forward-compatibility-fields)
22. [Provenance](#22-provenance)
23. [Streaming events](#23-streaming-events)
24. [Tool skills](#24-tool-skills)
25. [Non-goals](#25-non-goals)
26. [Backward compatibility](#26-backward-compatibility)
27. [Version history](#27-version-history)

---

## 1. What is this

A pipeline file is a single YAML document describing a multi-step workflow.
Each step (*stage*) is one of: a direct LLM call, a SKILL invocation, a nested
pipeline reference, a loop, or a runner-specific extension (e.g. local shell).

The `stages` array forms a DAG. Edges are declared with `depends_on`; stages
without dependencies MAY run concurrently. A stage skipped by a condition
produces a `null` output but keeps its position in the graph.

---

## 2. Design principles

- **Declarative, not imperative.** No embedded code, no JS expressions, no
  eval. Branching uses the pure-comparison [`when` DSL](#15-conditional-branches-when-dsl);
  iteration uses [`loop` blocks](#113-loop-stage) with a tiny exit grammar.
- **Portable core, explicit extensions.** Fields marked runner-specific
  (`working_dir`, `shell`) MUST be ignored with a warning by runners that do
  not support them — never silently mis-executed.
- **Additive evolution.** New fields are optional. A v1.0.0 file MUST run on
  any v1.x runner (unknown fields ignored with a warning); older files run
  unchanged on newer runners.
- **Reproducibility.** Version pins (`uses: skill@version`,
  `uses: pipeline:name@version`) plus [provenance](#22-provenance) let any
  party verify exactly what ran.

---

## 3. Quick example

```yaml
name: bobs-code-refactor
version: 1.2.0
description: |
  Takes a code snippet + language + task, produces refactored code with
  unit tests and documentation.
author_id: 0xBobAddress
author_name: Bob
license: MIT

price_usdc: 50000               # one-time license, micro-USDC (50000 = $0.05)
call_price_usdc: 500            # per-invocation fee, micro-USDC (optional)
preferred_model: MiniMax-M3     # hint for stages that omit `model`

inputs:
  code: { type: string, required: true }
  language: { type: string, required: true, enum: [javascript, typescript, python, rust, go] }
  task: { type: string, required: true, enum: [refactor, add_tests, fix_bugs, document] }

output:
  format: files
  files: [refactored_code.py, tests.test.py, README.md]
  delivery: signed_url
  expires_in: 86400

quality:
  min_rating: 4.0
  refund_window: 24h

limits:
  total_tokens: 30000
  total_cost_usdc: 0.50
  timeout_seconds: 600

stages:
  - id: analyze
    provider: openai
    model: gpt-4o
    input: { code: "$input.code", language: "$input.language", task: "$input.task" }
    output_format: structured_json
    fields: [issues, suggestions, complexity]

  - id: refactor
    uses: code_rewriter@0.2.1
    model: claude-3.5-sonnet
    depends_on: [analyze]
    input:
      code: "$input.code"
      suggestions: "${stages.analyze.outputs.suggestions}"
    output_format: code

  - id: check_vulnerabilities
    uses: security_scanner            # no pin → latest, provenance MUST flag loose pin
    depends_on: [refactor]
    mcp_servers: [github]
    mcp_tools: [github_search, github_get_file]
    output_format: structured_json
    fields: [vulnerabilities, severity_counts]

  - id: test
    uses: [test_generator, doc_writer]  # sequential skill array (v0.1 syntax, still valid)
    depends_on: [check_vulnerabilities]
    model: claude-3.5-sonnet
```

---

## 4. Frontmatter schema

Top-level fields of the pipeline YAML document.

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Pipeline identifier. MUST match `[a-z0-9_-]+`. |
| `version` | semver | yes | Pipeline version (e.g. `1.2.0`). |
| `description` | string | no | Human-readable summary for listings. |
| `author_id` | string | yes | Author identifier (address, DID, or handle — format is registry-defined). |
| `author_name` | string | no | Display name. |
| `price_usdc` | integer (micro-USDC) | no | One-time license fee. `50000` = $0.05. Omitted or `0` = free to install. |
| `call_price_usdc` | integer (micro-USDC) | no | Per-invocation fee. Default: `price_usdc / 100` (rounded down). Both omitted/`0` = free to run. |
| `preferred_model` | string | no | Hint for stages that omit `model`. Hint only — runners MAY fall back. |
| `license` | string | no | SPDX identifier for the *output* (e.g. `MIT`). Default: `Proprietary`. |
| `tags` | string[] | no | Discovery tags. |
| `is_private` | boolean | no | If `true`, hidden from public listings; access via registry-defined grants. |
| `concurrency` | integer | no | Global cap on concurrently executing stages. Default: unlimited. |
| `inputs` | object | no | Input declaration — see [§5](#5-inputs-block). |
| `output` | object | no | Output declaration — see [§6](#6-output-block). |
| `quality` | object | no | Quality guarantees — see [§7](#7-quality-and-limits-blocks). |
| `limits` | object | no | Run limits — see [§7](#7-quality-and-limits-blocks). |
| `cost` | object | no | Cost ceiling — see [§8](#8-cost-ceiling-block). |
| `memory` | object | no | Memory partitions — see [§9](#9-memory-block). |
| `stages` | array | yes | The DAG — see [§10](#10-stages-and-the-dag). |

Pricing fields declare *intent*; settlement mechanics (escrow, splits,
payment protocols) are registry/runner concerns and out of scope.

---

## 5. Inputs block

`inputs` declares what the invoker provides. Each key maps to a descriptor:

```yaml
inputs:
  code: { type: string, required: true }
  language: { type: string, required: true, enum: [javascript, typescript, python] }
  ecosystem: { type: string, required: false, default: npm }
```

| Key | Type | Description |
|-----|------|-------------|
| `type` | string | `string` \| `number` \| `integer` \| `boolean` \| `array` \| `object`. |
| `required` | boolean | Default `false`. Missing required input MUST fail validation before any stage runs. |
| `default` | any | Used when the invoker omits an optional input. |
| `enum` | array | Allowed values; other values MUST fail validation. |
| `description` | string | Human-readable hint for form rendering. |
| `min_length` / `max_length` | integer | String length bounds. |
| `minimum` / `maximum` | number | Numeric bounds. |

---

## 6. Output block

The top-level `output` block declares what the pipeline produces.

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `format` | enum | no | `text` | One of `text` \| `json` \| `zip` \| `files`. `files` requires the `files` array. |
| `files` | string[] | conditional | — | Glob patterns for expected files. Required when `format: files`. |
| `delivery` | enum | no | `inline` | One of `inline` \| `signed_url`. `signed_url` requires `expires_in`. |
| `expires_in` | integer (seconds) | conditional | `86400` | URL lifetime. MUST be within `60`–`604800`. |

**Cross-field invariants** (runners MUST validate before execution):

- `format: files` requires a non-empty `files` array.
- `delivery: signed_url` requires `expires_in` within range.

### 6.1. Glob syntax (`output.files`)

- `*.py` — files in the working directory root.
- `out/*.py` — files in the `out/` subdirectory.
- `**/*.py` — files anywhere (recursive).
- `*` matches a single path segment (not `/`); `**` matches zero or more segments.

### 6.2. Stage-emitted files (`produced_files`)

Stages with `output_format: structured_json` MAY emit files inline via a
`produced_files` array in their JSON output. Each entry:

| Field | Required | Description |
|-------|----------|-------------|
| `name` | yes | Flat filename. MUST NOT contain `/`, `\`, or `..`. |
| `content_type` | no | MIME type. Default: `application/octet-stream`. |
| `content_b64` | yes | Base64 of the file bytes. Decoded length SHOULD NOT exceed 10 MB. |

Recommended runner bounds: ≤10 MB per file, ≤50 MB total per run, ≤100 files.
At least one produced file SHOULD match each glob in `output.files`;
unmatched globs SHOULD surface as warnings, not errors.

---

## 7. Quality and limits blocks

```yaml
quality:
  min_rating: 4.0        # minimum acceptable rating (1–5 scale)
  refund_window: 24h     # window in which quality claims are valid (e.g. 24h, 7d)

limits:
  total_tokens: 30000    # across all stages
  total_cost_usdc: 0.50  # across all stages
  timeout_seconds: 600   # wall-clock budget for the whole run
```

If a stage would push cumulative tokens/cost past `limits`, the runner MUST
abort the run and report which limit fired. `quality` declares intent;
enforcement (refunds, disputes) is a registry concern.

---

## 8. Cost ceiling block

```yaml
cost:
  max_per_run_usdc: 0.50   # hard cap per invocation, in USDC (not micro-USDC)
  on_exceed: fail          # 'fail' (default) or 'warn'
```

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `max_per_run_usdc` | number (≥ 0) | `1.0` | Cap on the estimated per-run cost. `0` is a sentinel meaning **unlimited**. Negative values MUST be rejected at publish/validate time. |
| `on_exceed` | enum | `'fail'` | `'fail'` = refuse to run before any charge; `'warn'` = log + proceed. |

The ceiling is evaluated against the pre-run cost *estimate*, before any
payment or execution. `on_exceed` MUST be `'fail'` or `'warn'`.

---

## 9. Memory block

Declares named key-value partitions the pipeline MAY use to accumulate state
across stages (and, where the runner supports it, across invocations).

```yaml
memory:
  partitions:
    - name: research
      scope: pipeline        # 'pipeline' (default) or 'stage'
      ttl_seconds: 86400     # default 86400 (24h); range 1s–30d
```

| Field | Description |
|-------|-------------|
| `partitions[].name` | Partition name (≤64 chars). |
| `partitions[].scope` | `pipeline` = visible to all stages; `stage` = isolated per stage. |
| `partitions[].ttl_seconds` | Entry lifetime. Runners SHOULD expire lazily on read. |

Companion tool skills `tool:memory_store` / `tool:memory_recall` (see [§24](#24-tool-skills))
read and write partitions during execution. Recommended value limit: ≤1 MB
serialized per key; keys ≤256 chars. Partition data MUST be isolated per
pipeline — no cross-pipeline leakage.

---

## 10. Stages and the DAG

### 10.1. Common fields (all stage types)

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `id` | string | yes | Unique within the pipeline. Used in `depends_on` and refs. |
| `depends_on` | string[] | no | Stage IDs that MUST complete first. Omitted = MAY run concurrently with other root stages. |
| `input` | object | no | Stage inputs. Values MAY use `$input.X`, `${stages.X.outputs.Y}`, `{{ ... }}` templates. |
| `system` | string | no | System prompt for the LLM call. NOT merged with any skill prompt. |
| `provider` | string | conditional | Routing target (e.g. `openai`, `anthropic`, `minimax`). Required on LLM stages. |
| `model` | string | conditional | Model name (e.g. `gpt-4o`). Falls back to `preferred_model` when omitted. |
| `output_format` | enum | no | `text` (default) \| `structured_json` \| `code` \| `markdown`. `structured_json` requires `fields`. |
| `fields` | string[] | conditional | Required keys asserted in the LLM output when `output_format: structured_json`. |
| `output_schema` | object | no | JSON Schema subset constraining stage output — see [§20](#20-stage-output-validation). |
| `output_retry` | enum | no | `none` (default) \| `single` \| `loop` — see [§20](#20-stage-output-validation). |
| `retry` | object | no | Transient-error retries — see [§14](#14-retry-and-fallback). |
| `run_if` | string | no | Compensation gate — see [§14](#14-retry-and-fallback). |
| `when` | object | no | Conditional execution — see [§15](#15-conditional-branches-when-dsl). |
| `checkpoint` | object | no | Crash-recovery marker — see [§17](#17-checkpoints). |
| `parallel` | object | no | Fan-out over a collection — see [§18](#18-parallel-stages). |
| `mcp_servers` / `mcp_tools` | string[] | no | Declared MCP surface — see [§21](#21-mcp-forward-compatibility-fields). |
| `working_dir` | string | no | Runner-specific (local execution). Non-supporting runners MUST ignore with a warning. |

### 10.2. DAG semantics

- Stages listed with no `depends_on` are roots and MAY execute concurrently.
- A stage with `depends_on: [a, b]` starts only after **all** of `a`, `b` complete.
- A skipped stage (`when` false) yields `null` output; topology stays static.
- One stage's failure does not abort already-running siblings; whether the
  run continues is decided by downstream stages' `depends_on`/`run_if` gates
  and the runner's failure policy.

---

## 11. Stage types

### 11.1. LLM-only stage (`provider` + `model` + `prompt`)

```yaml
- id: analyze
  provider: openai
  model: gpt-4o
  system: "You are a senior code reviewer."
  prompt: |
    Language: $input.language
    Task: $input.task
    Code: $input.code
    List 3-5 specific improvement opportunities.
  output_format: structured_json
  fields: [issues, suggestions]
```

`prompt` MAY interpolate `$input.X`, `${stages.X.outputs.Y}`, and `{{ ... }}`
(see [§12](#12-template-interpolation)). No skill is loaded for this type.
Alternatively, a stage MAY use `messages:` (multimodal array) instead of
`prompt:` — if both are present, `messages` wins. See [§11.6](#116-multimodal-messages-stage).

### 11.2. Skill stage (`uses:`)

```yaml
- id: refactor
  uses: code_rewriter                # latest version (loose pin — provenance MUST flag)
  uses: code_rewriter@0.2.1          # strict pin (recommended for reproducibility)
  uses: [skill_a, skill_b@1.0.0]     # sequential array; per-skill pins allowed
  model: claude-3.5-sonnet           # LLM used with the skill as its system prompt
```

Execution contract:

1. Parse `uses` into `[{id, version?}]` entries.
2. Resolve each skill (registry-defined fetch + access check).
3. If `version` is pinned, the runner MUST refuse to run on version mismatch.
4. If unpinned, the runner MUST record a loose-pin warning in provenance.
5. The skill content becomes the LLM system prompt; `input` + `output_format`
   are sent with it.
6. `output_format: structured_json` + `fields: [...]` triggers post-LLM
   conformance: parse as JSON, assert all `fields` keys exist, else fail
   the stage.

Unpinned `uses: skill_X` ≡ `uses: skill_X@*`.

### 11.3. Parallel skill fan-out (`uses_parallel:`)

```yaml
- id: parallel_research
  uses_parallel:
    - uses: web_search
      input: { query: "$input.topic" }
    - uses: github_search
      input: { query: "$input.topic language:python" }
```

Entries fan out concurrently with partial-failure semantics (per-entry status
in provenance). Runners without parallel support MAY execute sequentially.

### 11.4. Loop stage (`loop:`)

Self-correcting iteration: body stages repeat until `exit_when` is truthy or
`max_iterations` is reached.

```yaml
- id: refine_summary
  loop:
    max_iterations: 3
    exit_when: "outputs.score >= 0.9"
    body:
      - id: generate
        provider: minimax
        model: MiniMax-M3
        prompt: "Summarize: {{source_text}}"
      - id: score
        provider: minimax
        model: MiniMax-M3
        output_schema:
          type: object
          properties:
            score: { type: number, minimum: 0, maximum: 1 }
          required: ["score"]
```

**Exit expression grammar** (no eval, no function calls):
`outputs.KEY OP VALUE` where `OP` ∈ `{>=, <=, ==, !=, >, <}` and `VALUE` is a
number, boolean, or quoted string. Outputs of all body stages are flattened
for lookup (`outputs.score` searches every body stage output for `score`).

**Limits:** `max_iterations` default 3, hard cap 10. Empty `body` MUST be
rejected. Exhausted budget MUST fail the stage distinctly (budget-exceeded,
not stage-failed). `loop:` is mutually exclusive with `shell:`, `uses:`,
`model:`, and `prompt:` on the same stage.

### 11.5. Shell stage (`shell:`) — runner-specific, non-portable

```yaml
- id: lint
  shell: "eslint src/"              # shorthand: split on whitespace, no shell interpretation
- id: run-tests
  shell:                            # explicit form
    cmd: pytest
    args: ['-x', '-q', 'tests/']
    cwd: ./my-project
    timeout_ms: 120000              # default 30000, max 600000
```

| Field | Default | Notes |
|-------|---------|-------|
| `cmd` | (required) | Program to run; absolute path or resolvable on `PATH`. |
| `args` | `[]` | Passed verbatim as argv — no shell metachar expansion. |
| `cwd` | runner cwd | Working directory. |
| `timeout_ms` | `30000` | Kill after expiry; max `600000`. |

Stage output is a JSON object:
`{cmd, args, cwd, stdout, stderr, exit_code, duration_ms}` (streams SHOULD be
truncated, e.g. at 1 MB). `shell` is mutually exclusive with `uses`,
`model`, and `prompt`. **Server-side/sandboxed runners MUST reject pipelines
containing `shell` stages** with a clear error directing the user to a local
runner. `shell` MUST NOT be combined with `loop:` on the same stage.

### 11.6. Multimodal messages stage

A stage MAY use `messages:` (array) instead of a string `prompt:`:

```yaml
- id: classify
  provider: minimax
  model: MiniMax-M3
  messages:
    - role: user
      content:
        - type: text
          text: "Classify this image"
        - type: image_url
          image_url: { url: "{{image_url}}" }
```

Each message has `role` + `content[]`; content parts are
`{type: 'text' | 'image_url' | 'audio_url' | 'video_url', ...}`.
If `messages` is present, `prompt` is ignored. Runners MUST route multimodal
stages only to capable models and fail clearly otherwise.

---

## 12. Template interpolation

Three complementary syntaxes (no conflicts by construction):

| Pattern | Engine | Example |
|---------|--------|---------|
| `{{var}}` | stage-input interpolation | `prompt: "Hello {{name}}"` |
| `$input.X` | pipeline input literal | `prompt: "$input.user_query"` |
| `${stages.X.outputs.Y}` | stage-output chaining | `prompt: "Prev: ${stages.analyse.outputs}"` |

`{{ }}` references resolve against stage `input` + pipeline `inputs`;
`$input.X` against pipeline inputs only. Stage-output refs are covered in
[§16](#16-stage-output-refs).

---

## 13. Environment variable substitution

`${VAR}` / `${VAR:-default}` references resolve from the runner environment
**before** YAML parsing (text preprocessing), so they work in any string
position — scalars, array elements, nested maps, block scalars.

| Pattern | env present | env absent |
|---------|-------------|------------|
| `${VAR}` | env value | MUST fail with `MissingEnvVar`-class error |
| `${VAR:-default}` | env value (default ignored) | `default` value |
| `${VAR:-}` | env value (incl. empty) | empty string |

Var-name grammar: `[A-Z_][A-Z0-9_]*`. Substitution is single-pass
(non-recursive). Substitution MUST happen exactly once, at load time —
runners MUST NOT re-substitute stored specs at invoke time (environment may
have changed between authoring and invocation).

---

## 14. Retry and fallback

Two separate primitives:

- **`retry:`** — repeat with the same provider/model on transient errors.
- **`run_if:`** — run an alternative stage when a previous stage failed
  (compensation gate, GitHub-Actions-`if:-failure()` style).

```yaml
- id: call-primary
  provider: openai
  model: gpt-4o
  retry:
    attempts: 3
    backoff: exponential
    retryable_status: [429, 503]
  prompt: "..."

- id: call-fallback
  provider: openrouter
  model: openai/gpt-4o
  run_if: "{{ stages.call-primary.status == 'failed' }}"
  prompt: "..."
```

`retry` config is pure data (attempts, backoff strategy name, status-code
whitelist) — no expressions. Budget exhausted MUST surface distinctly
(retry-exhausted, not provider error). Fallback chains compose by stacking
`run_if` gates (`try-openai` → `try-anthropic` → `try-openrouter`).

---

## 15. Conditional branches: `when` DSL

Skip stages conditionally on prior outputs. Pure-comparison DSL — no code,
no I/O access.

```yaml
- id: handle-bug
  when:
    all:
      - stages.classify.outputs.category == 'bug'
      - stages.classify.outputs.confidence > 0.7
```

| Operator | Meaning |
|----------|---------|
| `==`, `!=` | equality |
| `<`, `>`, `<=`, `>=` | ordering |
| `in`, `not in` | membership |
| `all`, `any` | boolean composition |

Anything else MUST be rejected at validate time. `when: false` means the
stage is **skipped, not deleted**: output is `null`, topology is static,
decisions happen at execution time.

---

## 16. Stage output refs

Inline reference to an already-executed stage's output inside downstream
strings (`prompt`, `system`, `input`):
`${stages.STAGE_ID.outputs.KEY}`. Composes with env-var substitution —
env vars resolve at load time, stage refs resolve per-stage at execution time.

Identifier grammar: stage-id `[A-Za-z_][A-Za-z0-9_-]*`, output-key
`[A-Za-z_][A-Za-z0-9_-]*`.

| Case | Behavior |
|------|----------|
| Ref to a completed stage | Substituted with the value (`''` if that stage was `when`-skipped). |
| Ref to a stage NOT yet executed (forward ref) | MUST fail fast — abort the run. |
| Ref inside a larger string | Substituted inline, rest preserved. |
| Multiple refs in one string | All substituted independently. |

Guard idiom: `when: "stages.x.outputs.X != ''"` prevents a stage from
running on empty upstream input. A single bare ref is also a valid `when`
comparison operand.

---

## 17. Checkpoints

Persist run state mid-execution so long pipelines survive runner crashes
and wall-clock limits.

```yaml
- id: long-phase
  checkpoint:
    on: stage_complete
    storage:
      type: backend-name       # runner-defined (e.g. r2, s3, local, kv)
      path: checkpoints/{pipeline_id}/{exec_id}.json
    retention: 24h
```

What gets checkpointed (outputs are recomputable and SHOULD NOT be stored):

- Which stages completed (id + start timestamp).
- Resolved template variables (avoid re-running interpolation).
- Current DAG position (progress display).
- Operator/audit trail so far (resume preserves it).

On resume: deserialize → skip completed stages → continue from pending.

---

## 18. Parallel stages

### 18.1. Implicit parallelism

Stages without `depends_on` run concurrently; a stage with `depends_on`
waits for all listed stages:

```yaml
stages:
  - id: analyze-code
    provider: openai
    prompt: "Analyze: {{ inputs.code }}"
  - id: analyze-style
    provider: openai
    prompt: "Style: {{ inputs.code }}"
  - id: synthesize
    depends_on: [analyze-code, analyze-style]
    provider: openai
    prompt: "Synthesize..."
```

### 18.2. Fan-out over collections (`parallel.of`)

```yaml
- id: scan-endpoints
  parallel:
    of: "{{ inputs.endpoints }}"
    max_concurrent: 5
    body:
      provider: openai
      prompt: "Scan {{ item.url }}..."
```

Each element binds as `{{ item }}` inside `body`.

### 18.3. Concurrency controls

| Setting | Effect |
|---------|--------|
| Default | unlimited (`Promise.all` semantics) |
| Pipeline-level `concurrency: N` | global cap |
| Provider-level `rate_limit` | rate protection (registry-defined syntax) |
| `parallel.of.max_concurrent` | fan-out bound |

Stages with `streaming.enabled: true` emit to a shared event bus; consumers
subscribe by event tag. See [§23](#23-streaming-events).

---

## 19. Composition

A stage's `uses:` MAY reference a **published pipeline** instead of a skill.
The referenced pipeline's stages are inlined at invocation time with
id-prefixing, so they cannot collide with parent ids.

```yaml
stages:
  - id: refactor
    uses: code_rewriter@0.2.1          # skill ref (no prefix)
  - id: research
    uses: pipeline:web-research-helper@1.2.0   # pipeline ref (prefix + pin)
  - id: enrich
    uses: pipeline:enrichment-helper   # loose pin (latest)
```

**Identifier grammar:** literal prefix `pipeline:`, then pipeline id
`[A-Za-z_][A-Za-z0-9_-]*`, then optional `@<semver>` pin. Absent pin = latest.

**Expansion semantics:**

1. Parse `uses:` — strings starting with `pipeline:` are pipeline refs,
   anything else is a skill ref.
2. Resolve the referenced pipeline (registry-defined fetch).
3. Recursively expand (composition nests).
4. Prefix each inlined stage id with `<parent_id>.<child_id>`
   (e.g. `research` → `research.classify`, `research.extract`).
5. Rewrite non-dotted `depends_on` entries with the prefix; already-dotted
   paths are preserved verbatim. Downstream stages address expanded ids via
   the full dotted path (`stages.research.extract.outputs.entities`).

**Depth limit:** `MAX_COMPOSITION_DEPTH = 4`. Deeper nesting MUST fail with a
composition-depth error. This is also the cycle guard (`A → B → A` is blocked
by depth, no visited-set needed).

**Fetch timing:** at invoke time against the pipeline registry. Stored specs
keep the original `uses: pipeline:NAME@VERSION` syntax verbatim in
provenance for reproducibility. Pinned refs SHOULD resolve to the pinned
version; loose refs resolve to latest. Unresolvable refs MUST abort the run.

---

## 20. Stage output validation

Per-stage `output_schema` constrains the executor's string output against a
JSON Schema (draft-07 subset). Companion `output_retry` decides what happens
on failure. Validation runs AFTER the stage returns and BEFORE downstream
stages see the value.

```yaml
- id: aggregate
  model: MiniMax-M3
  prompt: |
    Emit JSON: { "summary": {...}, "recommendations": [...], "markdown_report": "..." }
  output_schema:
    type: object
    required: [summary, recommendations, markdown_report]
    additionalProperties: false
    properties:
      summary:
        type: object
        required: [total_findings, overall_risk_score]
        properties:
          total_findings: { type: integer }
          overall_risk_score: { type: number }
      recommendations:
        type: array
        maxItems: 5
        items: { type: string, maxLength: 200 }
      markdown_report:
        type: string
        maxLength: 8000
  output_retry: single   # none | single | loop (default: none)
```

### 20.1. Supported JSON Schema subset

| Keyword | Notes |
|---------|-------|
| `type` | `string` \| `number` \| `integer` \| `boolean` \| `array` \| `object` \| `null` (or array of these). `number` also accepts integers. |
| `properties` | `Record<string, Schema>` for object validation. |
| `required` | `string[]` of required keys. |
| `additionalProperties` | `boolean` \| Schema. `false` = strict. |
| `enum` | Exact-value match; short-circuits all other checks. |
| `items` | Schema for array elements. |
| `minItems` / `maxItems` | Array length bounds. |
| `minLength` / `maxLength` | String length bounds. |
| `pattern` | Regex matched against the whole string value. |
| `format` | Informational only; MUST NOT be validated. |

**Unsupported keywords** (schema MUST be rejected at parse time, before any
validation runs): `$ref`, `definitions`, `$defs`, `allOf`, `anyOf`, `oneOf`,
`not`, `dependencies`, `propertyNames`. Maximum nesting depth: 32.

### 20.2. Retry modes

| Mode | Total attempts | Behavior |
|------|----------------|----------|
| `none` (default) | 1 | Fail fast on first validation failure. |
| `single` | 2 | Re-run the stage once after first failure. |
| `loop` | 3 | Up to 3 attempts before failing. |

`output_retry` is independent of HTTP-status `retry:` (which handles
transient provider failures first). `output_retry` without `output_schema`
is ignored.

### 20.3. Validation error envelope

Failed validation MUST surface a structured envelope containing at minimum:

```json
{
  "error": "stage 'aggregate' failed output validation after 2 attempt(s)",
  "code": "OUTPUT_VALIDATION_FAILED",
  "stage_id": "aggregate",
  "attempt": 2,
  "last_error": {
    "path": "$.summary.total_findings",
    "message": "missing required property 'total_findings'",
    "schema_path": "$.properties.summary.required"
  },
  "violations": [
    {
      "path": "$.summary.total_findings",
      "message": "missing required property 'total_findings'",
      "schema_path": "$.properties.summary.required"
    }
  ],
  "run_id": "<uuid>"
}
```

`path` points into the value, `schema_path` into the schema. `violations`
carries the full list (runners MAY cap at 100).

---

## 21. MCP forward-compatibility fields

```yaml
- id: research
  uses: web_researcher
  mcp_servers: [playwright, github]
  mcp_tools: [playwright_navigate, github_search]
```

- Every name in `mcp_servers` MUST exist in the runner's MCP registry;
  every name in `mcp_tools` MUST be a known tool of one of the declared
  servers — otherwise fail with an MCP-config error.
- Runners without MCP execution MUST still parse + validate these fields
  and record the outcome in provenance, so a future runner can execute
  without re-validation.

---

## 22. Provenance

Provenance is **mandatory**. Every stage run records at minimum:

```json
{
  "stage_id": "refactor",
  "started_at": "2026-09-15T12:34:56.789Z",
  "completed_at": "2026-09-15T12:35:01.234Z",
  "duration_ms": 4445,
  "model_used": "claude-3.5-sonnet",
  "uses": [
    {
      "id": "code_rewriter",
      "version": "0.2.1",
      "version_pin": "strict",
      "content_hash": "keccak256:9a4f...",
      "fetched_at": "2026-09-15T12:34:57.000Z"
    }
  ],
  "mcp_servers_validated": [],
  "input_sha256": "f4c1...",
  "output_sha256": "a92d...",
  "output_format": "code",
  "output_format_validated": true,
  "cost_micro_usdc": 12650,
  "tokens": { "input": 1234, "output": 567 }
}
```

Pipeline-level provenance aggregates: `pipeline_name`, `pipeline_version`,
`spec_hash`, `inputs_hash`, `outputs_hash`, `ts`, `stages: [...]`.
Unpinned `uses:` MUST emit `"version_pin": "loose"` (with a warning).
Signature/attestation envelopes around this payload are registry-defined.

---

## 23. Streaming events

Runners that support streaming SHOULD emit these events in order:

| Event | Payload |
|-------|---------|
| `stage.start` | `{stage_id, attempt}` |
| `stage.output` | `{stage_id, partial, output, cost_so_far}` |
| `stage.complete` | `{stage_id, duration_ms, cost, cache_hit}` |
| `heartbeat` | `{ts}` — at a runner-defined interval |
| `run.complete` | `{run_id, total_cost, output, cache}` |
| `run.error` | `{error, code}` — terminal, connection closes |

Cached runs SHOULD return exactly one event (`run.complete`).

---

## 24. Tool skills

Runtime-native tools invokable from stages via the `tool:` namespace:

| Tool | Purpose |
|------|---------|
| `tool:image_describe` | Image → text description (multimodal model). |
| `tool:audio_transcribe` | Audio → text (speech-to-text). |
| `tool:tts_synthesize` | Text → speech audio. |
| `tool:image_generate` | Text prompt → image. |
| `tool:video_generate` | Text prompt → video (async). |
| `tool:memory_recall` | Read value / list keys by prefix from a memory partition. |
| `tool:memory_store` | Write value to a memory partition (with optional TTL). |

Availability is runner-defined (feature flags, provider keys). Unavailable
tools MUST either return a deterministic mock (clearly marked) or fail with
an explicit not-configured error — never silently succeed.

---

## 25. Non-goals

The following are **out of scope** for this spec (registry/runner concerns):

- Payment rails, escrow, fee splits, payout routing.
- Skill/pipeline marketplaces: search, publish, versioning policy, ratings.
- Server endpoints, CLI flags, wire protocols, auth schemes.
- Storage backends (object stores, KV, caches) and URL-signing schemes.
- Cost formulas, token pricing, compute overhead accounting.
- Sandbox execution, TEE attestation, operator key management.

Where the lineage format documented these, this spec keeps only the
*shape* a pipeline author writes (field names, types, defaults).

---

## 26. Backward compatibility

| Older feature | v1.0.0 status |
|---------------|---------------|
| `model:` + `prompt:` stage | ✅ unchanged |
| `uses: skill_id` (string, unpinned) | ✅ unchanged (≡ `skill_id@*`, loose pin) |
| `uses: [a, b]` (sequential array) | ✅ unchanged |
| `depends_on: [...]` | ✅ unchanged |
| `$input.X` interpolation | ✅ unchanged |
| `${stages.X.outputs.Y}` refs | ✅ unchanged |
| `output_format:` (omitted = text) | ✅ unchanged |
| `agent` attribute (legacy) | ⚠️ Deprecated. Use `model:` (LLM) or `uses:` (skill). |

Newer optional fields are ignored (with a warning) by older runners;
older files run unchanged on newer runners.

---

## 27. Version history

| Spec | Format lineage | Adds (format-relevant) |
|------|---------------|------------------------|
| **1.0.0** (2026-09-26) | v0.10 | Initial vendor-neutral extraction: full stage model, `when`/`retry`/`run_if`/refs/checkpoints/parallel/composition, `output_schema` + `output_retry`, `cost` ceiling shape, `memory` + `loop` blocks, `messages` multimodal, `shell` as explicit non-portable extension, `produced_files`, provenance shape, streaming events, tool-skill namespace. Strips all runtime/marketplace/payment specifics. |
| — | v0.9 | Multimodal `messages[]`, streaming events, run inspection, 5 media tool skills, run-inspection + composition tools. |
| — | v0.8 | `output_schema` (JSON Schema draft-07 subset) + `output_retry` (`none`/`single`/`loop`). |
| — | v0.7 | Cache re-signing note; `cost.max_per_run_usdc` + `cost.on_exceed`. |
| — | v0.6 | Invocation memoization (runner-defined). |
| — | v0.5 | `price_usdc` formalized, `call_price_usdc`, `preferred_model`, `output` block, `shell` stages, `produced_files`. |
| — | v0.3 | `provider` field, `${VAR}` env substitution, stage refs, retry/fallback, checkpoints, parallel DAG, `when` DSL, composition. |
| — | v0.2 | `uses@version` pins, `structured_json` + `fields`, `mcp_servers`/`mcp_tools`, `working_dir`. |
| — | v0.1 | Initial: `model`+`prompt`, `uses` stages, `depends_on`, `$input` interpolation. |
