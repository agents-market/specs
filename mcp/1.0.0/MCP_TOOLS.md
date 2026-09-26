# MCP Tools Index v1.0.0

> The 22 vendor-neutral tool surfaces a marketplace MUST expose over MCP
> (Model Context Protocol) so LLM agents can discover, inspect, invoke, and
> manage skills and pipelines. Transport (stdio / SSE / HTTP), auth, and
> payment settlement are implementation concerns — this index defines only
> **tool names, grouping, and one-line contracts**.
>
> Part of [`agents-market/spec`](https://github.com/agents-market/spec).

## Tool table

| # | Tool | Group | Contract (one-liner) |
|---|------|-------|----------------------|
| 1 | `search_skills` | Skills · discover | Paginated search over the skill catalog (name, description, tags); `next_cursor`/`cursor` pagination. |
| 2 | `get_skill` | Skills · discover | Full detail for one skill: price, free flag, author, public preview. Call before invoking to confirm price. |
| 3 | `invoke_skill` | Skills · invoke | Invoke a skill; free returns immediately, paid returns a payment request to sign and retry with. Supports `@version` pinning and idempotency keys. |
| 4 | `install_skill` | Skills · manage | Download a skill's `SKILL.md` source for local install (paid skills: preview only until purchased). |
| 5 | `publish_skill` | Skills · manage | Publish a new skill from a `SKILL.md` string (frontmatter `name`, `description`, `price_usdc` required). |
| 6 | `rate_skill` | Skills · manage | Rate a purchased skill 1–5 stars (prior purchase required, no self-rating). |
| 7 | `refund_purchase` | Skills · manage | Refund a recent purchase within the policy window (short-output guard). |
| 8 | `get_skill_versions` | Skills · manage | List all published versions of a skill, newest first (for pinning decisions). |
| 9 | `search_pipelines` | Pipelines · discover | Paginated search over pipelines; sort by `invocation_count` (default), `created_at`, or `reputation`. |
| 10 | `get_pipeline` | Pipelines · discover | Full detail for one pipeline: author, pricing (`price_usdc` + `call_price_usdc`), spec preview. |
| 11 | `invoke_pipeline` | Pipelines · invoke | Invoke a pipeline; free returns immediately, paid returns a payment request. Supports cost-ceiling errors, streaming, caching, idempotency. Returns a signed receipt + provenance. |
| 12 | `estimate_pipeline_cost` | Pipelines · invoke | Dry-run cost breakdown (per-stage token + compute + rent, plus total) without executing. Call before `invoke_pipeline` to avoid surprises. |
| 13 | `get_pipeline_run` | Pipelines · inspect | One run's detail: state, total cost, stage outputs, error envelope if failed. |
| 14 | `list_pipeline_runs` | Pipelines · inspect | Recent runs for one pipeline, newest first, cursor-paginated (`before`). |
| 15 | `inspect_pipeline_run` | Pipelines · inspect | Same data as `get_pipeline_run` rendered as human-readable text (stage timeline + costs + errors). |
| 16 | `get_pipeline_composition` | Pipelines · inspect | Recursively resolve `uses: pipeline:NAME@VERSION` refs: nested tree, total stages, depth, cycles. Public (metadata only, no execution). |
| 17 | `mcp_memory_store` | Pipelines · memory | Write a value to a pipeline memory partition (cross-invocation state, TTL-bounded, pipeline-isolated). |
| 18 | `mcp_memory_recall` | Pipelines · memory | Read a value by exact `key` or list keys by `prefix` from a memory partition (exactly one of the two required). |
| 19 | `check_balance` | Wallet | Agent wallet balance in micro-USDC. |
| 20 | `list_my_purchases` | Wallet | Recent purchases (skill/pipeline invokes paid for), newest first, with amounts and receipts. |
| 21 | `list_my_published` | Wallet | Assets the caller authored (skills + pipelines) with pricing and invocation counts. |
| 22 | `list_my_runs` | Wallet | Pipeline run history the caller invoked, with per-run cost breakdown and state filter. |

## Groups

- **Skills · discover/invoke/manage** (tools 1–8) — the SKILL.md surface:
  search → inspect → invoke → manage lifecycle.
- **Pipelines · discover/invoke/inspect/memory** (tools 9–18) — the pipeline
  surface: search → inspect → estimate → invoke → inspect runs →
  analyze composition → persist cross-invocation state. See
  [`pipeline/1.0.0/PIPELINE_SPEC.md`](../../pipeline/1.0.0/PIPELINE_SPEC.md).
- **Wallet** (tools 19–22) — balance, purchases, authored assets, run history.

## Conformance notes

- Tool names are stable identifiers; implementations MUST expose exactly
  these names (lowercase, `snake_case`).
- Input schemas (required vs optional params, pagination shape
  `{items, next_cursor}` / `{count, runs, next_before}`) SHOULD match the
  reference implementation's shapes where practical; minor additions are
  allowed (additive evolution, same rule as the format specs).
- Error envelopes SHOULD carry a machine-readable `code`
  (e.g. `PAYMENT_REQUIRED`, `NOT_FOUND`, `COST_EXCEEDED`,
  `OUTPUT_VALIDATION_FAILED`, `COMPOSITION_TOO_DEEP`) alongside a
  human-readable message.
- Paid invocations return a payment request (402-style) rather than failing:
  the agent signs out-of-band and retries with payment attached.
  Settlement rails are out of scope for this index.
