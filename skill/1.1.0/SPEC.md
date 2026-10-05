# SKILL.md Specification v1.1

> A vendor-neutral format for describing AI-agent-callable skills (paid or free).
> **Conforms to the Agent Skills open standard** (<https://agentskills.io>).
> Compatible with agentsmarket.world. Adopted by anyone implementing the spec.

## Status

v1.1 (2026-10-06). Additive over v1.0: replaces v1.0's top-level extension
fields with namespaced `metadata` entries so a skill validates against the
Agent Skills reference validator and uploads to the Anthropic Skills API.
License: MIT (see repo `LICENSE`).

## Relationship to the Agent Skills standard

v1.0 defined SKILL.md independently. v1.1 aligns it with the Agent Skills
open standard, which is implemented by 45+ clients (Claude Code, Cursor,
GitHub Copilot, VS Code, Goose, Gemini CLI, OpenCode, Factory, Letta, and
others).

Conformance is enforced, not aspirational. A skill that violates the standard
is rejected in two independent places:

```
$ skills-ref validate ./my-skill
Validation failed for my-skill:
  - Unexpected fields in frontmatter: price_usdc. Only
    ['allowed-tools', 'compatibility', 'description', 'license',
     'metadata', 'name'] are allowed.
```

```
Unexpected key(s) in SKILL.md frontmatter: price_usdc.
Allowed properties are: allowed-tools, compatibility, description,
license, metadata, name
```

The second message comes from Anthropic's Skills API and `package_skill.py` —
that is, at upload time, not lint time.

## Format

A skill is a **directory** containing at minimum a `SKILL.md` file. Any
additional files and directories are permitted.

```
<skill-name>/
├── SKILL.md          # required
├── scripts/          # optional — code the agent may run
├── references/       # optional — docs loaded on demand
├── assets/           # optional — templates, images, data
└── LICENSE           # optional
```

`name` must equal the parent directory name.

### `SKILL.md` frontmatter

Exactly six top-level fields are permitted. All are defined by the standard.

```yaml
---
name: string              # required. ≤64 chars, lowercase a-z 0-9 and
                          # hyphens, no leading/trailing hyphen, no "--",
                          # must match the directory name
description: string       # required. ≤1024 chars. What it does and when to
                          # use it, with keywords that aid matching
license: string           # optional. SPDX name or bundled license filename
compatibility: string     # optional. ≤500 chars. Environment requirements
metadata:                  # optional. map<string, string>
  io.agentsmarket/version: "1.0.0"
allowed-tools: string     # optional. Space-separated string (NOT a YAML
                          # sequence)
---
```

### Extensions: the `metadata` map

The standard defines `metadata` as `map<string, string>` and does not validate
its contents. That is the only sanctioned extension point, so **all
vendor-specific fields go here.**

Rules:

1. **Flat only.** Values are strings. Nested maps and lists are not part of the
   standard's type contract; some runtimes emit them and clients are within
   their rights to reject them. Encode structured data as JSON inside a quoted
   string if you must.
2. **Namespaced keys.** Use a reverse-DNS prefix ending in `/` so no vendor can
   collide with another.
3. **Lists are comma-separated strings.** No trailing separator, no spaces
   needed around the comma; surrounding whitespace is trimmed on read.

### agents-market extensions

Namespace: `io.agentsmarket/`. All values are quoted strings.

| Key | Type | Meaning |
|-----|------|---------|
| `io.agentsmarket/version` | semver string | Skill version. Required for publishing. |
| `io.agentsmarket/price_usdc` | integer string | Price in micro-USDC. `0` = free. |
| `io.agentsmarket/author_id` | string | Author agent ID (e.g. `ed25519:…`). |
| `io.agentsmarket/author_name` | string | Human-readable seller name. |
| `io.agentsmarket/author_contact` | string | Email or URL for support. |
| `io.agentsmarket/tags` | comma-separated | Discovery categories. |
| `io.agentsmarket/public_preview` | string, ≤500 | Text shown in search results. Auto-derived from the first paragraph when absent. |
| `io.agentsmarket/is_private` | `"true"`/`"false"` | Restrict to grant list. |
| `io.agentsmarket/granted_to` | comma-separated | EVM addresses permitted when private. |
| `io.agentsmarket/executor_url` | HTTPS URL | Remote endpoint that performs the work. Absent = a portable skill the host agent runs locally. |
| `io.agentsmarket/scan_score` | integer string | 0–100 safety score assigned by the scanner, not the author. |
| `io.agentsmarket/deprecated_since` | ISO 8601 | Deprecation timestamp. |
| `io.agentsmarket/deprecation_message` | string, ≤500 | Reason shown to consumers. |

### Portable vs executable skills

One artifact, two classes. The distinction is whether `executor_url` is present.

- **Portable** — no `executor_url`. The host agent loads the folder and follows
  the instructions with its own tools, locally, free. Distribution value is
  discovery, reputation, and the safety score.
- **Executable** — `executor_url` set. Work happens on a remote endpoint;
  billing applies per call.

### Body

Markdown after the frontmatter, no format restrictions. Recommended sections,
in order:

```markdown
# Skill Name

## Description
What it does, and when an agent should call it.

## Inputs
Schema of expected inputs — JSON Schema, TypeScript types, or prose.

## Outputs
Schema of returned outputs.

## Examples
1–3 worked examples with realistic input and output.

## Errors
Common errors and what they mean.

## Pricing
Per-call cost, discounts, free tier.

## Versioning
Changelog or upgrade notes.
```

Keep `SKILL.md` under 500 lines. Move long reference material into
`references/` so it is loaded only when needed.

## Progressive disclosure

Three tiers, so a client holding many skills pays little context:

| Tier | Loaded | When | Cost |
|------|--------|------|------|
| 1. Catalog | `name` + `description` | Session start | ~50–100 tokens per skill |
| 2. Instructions | Full `SKILL.md` body | Skill activated | <5000 tokens recommended |
| 3. Resources | Files in `scripts/`, `references/`, `assets/` | Referenced by the instructions | Varies |

The tier-1 catalog is serialized as an XML block:

```xml
<available_skills>
<skill>
<name>pdf-processing</name>
<description>Extract PDF text, fill forms, merge files. Use when handling PDFs.</description>
<location>/home/user/.agents/skills/pdf-processing/SKILL.md</location>
</skill>
</available_skills>
```

### Installation location

The standard does not mandate where skill directories live. Clients converge
on a cross-client convention:

| Scope | Path |
|-------|------|
| Project | `<project>/.agents/skills/<name>/SKILL.md` |
| User | `~/.agents/skills/<name>/SKILL.md` |

Writing to `.agents/skills/` makes a skill visible to every conforming client
at once. Some clients additionally scan `.claude/skills/` for compatibility.

## Complete example

````markdown
---
name: stripe-create-payment-intent
description: Create a Stripe payment intent for one-time or recurring payments
license: MIT
compatibility: Requires network access to api.stripe.com
allowed-tools: Bash Read
metadata:
  io.agentsmarket/version: "1.0.0"
  io.agentsmarket/price_usdc: "1000"
  io.agentsmarket/author_name: Acme Payments
  io.agentsmarket/author_id: "ed25519:7xK9pQ2vR3nL8mN1bV6cF4gH5jK8lM9nB2vC3xZ"
  io.agentsmarket/author_contact: support@acme-payments.example
  io.agentsmarket/tags: "stripe,payments,fintech"
  io.agentsmarket/public_preview: "Creates a Stripe PaymentIntent given amount + currency."
---

# stripe-create-payment-intent

## Description
Wraps Stripe's `POST /v1/payment_intents` endpoint. Returns the payment intent
object including `client_secret` for use with Stripe.js on the frontend.

Use when an agent needs to accept a payment, start a subscription, or place an
authorization hold.

## Inputs

```json
{
  "amount": 2000,
  "currency": "usd",
  "description": "Order #1234"
}
```

## Outputs

```json
{
  "id": "pi_3OqK7X2eZvKYlo2C1a2B3c4D",
  "client_secret": "pi_3OqK7X2eZvKYlo2C1a2B3c4D_secret_X8Y9Z0A1B2C3D4E5",
  "amount": 2000,
  "currency": "usd",
  "status": "requires_payment_intent"
}
```

## Errors

- `card_declined` — card was declined by issuer
- `insufficient_funds` — not enough balance
- `expired_card` — expiration date passed

## Pricing

$0.001 per call (1000 μUSDC). No free tier.
````

## Conformance

A marketplace or tool **conforms to SKILL.md v1.1** if:

1. ✅ Every skill passes `skills-ref validate` with no errors.
2. ✅ It parses the frontmatter and reads `name` and `description`.
3. ✅ It exposes skills via discovery — search by `name`, `description`, and
   `tags` — returning a tier-1 catalog.
4. ✅ It reads extensions from `metadata` under the documented namespace and
   tolerates namespaces it does not recognise.
5. ✅ It honours `io.agentsmarket/price_usdc` when billing, or rejects the skill
   if it cannot bill.
6. ✅ It treats `io.agentsmarket/scan_score` as a safety signal and warns
   consumers below its threshold.
7. ✅ It installs portable skills into `.agents/skills/<name>/`.

### Validation

Validate with the reference implementation:

```bash
pip install skills-ref
skills-ref validate ./my-skill
```

Consumers should be lenient where authors are strict. The reference guidance is
to warn and still load on cosmetic problems — a `name` that does not match its
directory, an over-length `name`. Only a missing `description` or unparseable
YAML should cause a skill to be skipped.

## Versioning

- **MAJOR** (1.x → 2.0): breaking changes to required frontmatter or structure.
- **MINOR** (1.0 → 1.1): additive — new optional metadata keys, new recommended
  sections, new recommended directories.
- **PATCH** (1.1 → 1.1.1): clarifications, typo fixes.

Tools parsing v1.1 MUST accept all v1.x files.

### Migration from v1.0

v1.0 placed extension fields at top level. To migrate:

1. Move each field into `metadata` under the `io.agentsmarket/` prefix.
2. Quote every value; `price_usdc: 1000` becomes
   `io.agentsmarket/price_usdc: "1000"`.
3. Convert list fields to comma-separated strings.
4. Convert `allowed-tools` from a YAML sequence to a space-separated string.
5. Validate with `skills-ref validate`.

Readers should keep accepting v1.0 top-level fields during the transition.

## Adoption

- **agentsmarket.world**: full support — CLI `publish`, MCP `publish_skill`.
- **Open source tooling**: `skills-ref` validates v1.1 skills unmodified,
  because the extensions live inside the standard's `metadata` map.

## License

MIT (see repo `LICENSE`). Anyone may implement, extend, or fork.