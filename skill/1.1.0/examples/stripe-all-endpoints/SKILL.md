---
name: stripe-all-endpoints
description: Router for 3 Stripe API endpoints. Provide a natural-language task and this skill returns the matching child skill name to invoke.
license: CC0-1.0
compatibility: Requires network access to api.stripe.com
metadata:
  io.agentsmarket/version: "1.0.0"
  io.agentsmarket/price_usdc: "0"
  io.agentsmarket/author_name: stripe
  io.agentsmarket/author_id: "0xPLACEHOLDER_REPLACE_WITH_YOUR_AGENT_ID"
  io.agentsmarket/author_contact: support@stripe.example
  io.agentsmarket/tags: "stripe,router,meta"
---

# stripe-all-endpoints

Router for 3 Stripe API endpoints.

## Usage

Provide a natural-language task. This skill returns the matching child
`skill_name` and endpoint. Invoke the child skill separately with the
required parameters.

## Inputs

```json
{
  "task": "create a payment intent for $20"
}
```

## Outputs

```json
{
  "skill_name": "stripe-create-payment-intent",
  "method": "POST",
  "endpoint": "/v1/payment_intents",
  "confidence": 0.94
}
```

## Errors

- `no_match` — no endpoint matched the task; suggest a reformulation
- `ambiguous` — several endpoints matched; the caller must disambiguate

## Pricing

Free. The child skill carries its own price.

## Versioning

- 1.0.0 (2026-10-06): initial release, migrated to SKILL.md v1.1
