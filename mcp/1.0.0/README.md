# MCP Surface v1.0.0

> Which tools a marketplace exposes to LLM agents — and what each tool
> promises. Part of [`agents-market/spec`](https://github.com/agents-market/spec).

## What is this?

Agents need a stable tool surface to discover, invoke, and manage skills and
pipelines. This directory pins that surface at v1.0.0: **22 tools** in three
groups (Skills, Pipelines, Wallet).

- [`MCP_TOOLS.md`](MCP_TOOLS.md) — the full index: all 22 tools, grouping,
  one-line contracts, conformance notes.

## Non-goals

Transport (stdio vs SSE vs HTTP), authentication schemes, and payment
settlement are implementation concerns. This directory defines tool names
and contracts only — any transport that preserves them conforms.

## Version history

- **1.0.0** (2026-09-26) — initial extraction: 8 skill tools, 10 pipeline
  tools (incl. run inspection + composition analysis + cross-invocation
  memory), 4 wallet tools.
