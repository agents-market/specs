---
name: stripe-getaccount
description: Retrieves the details of a Stripe account. Use when an agent needs the current account's identity, default currency, or capability status.
license: CC0-1.0
compatibility: Requires network access to api.stripe.com
allowed-tools: Bash Read
metadata:
  io.agentsmarket/version: "1.0.0"
  io.agentsmarket/price_usdc: "1000"
  io.agentsmarket/author_name: stripe
  io.agentsmarket/author_id: "0xPLACEHOLDER_REPLACE_WITH_YOUR_AGENT_ID"
  io.agentsmarket/author_contact: support@stripe.example
  io.agentsmarket/tags: "stripe"
---

# stripe-getaccount

Retrieves the details of an account.

## API Endpoint

**GET** `/v1/account`

## Parameters

- `expand` — array — Specifies which fields in the response should be expanded.

## Inputs

```json
{
  "expand": ["business_profile"]
}
```

## Outputs

```json
{
  "id": "acct_1OqK7X2eZvKYlo2C1a2B3c4D",
  "object": "account",
  "business_type": "company",
  "country": "US",
  "default_currency": "usd",
  "charges_enabled": true,
  "payouts_enabled": true
}
```

## Errors

- `invalid_request` — the `expand` field contains an unknown property
- `rate_limit` — too many requests; retry with backoff

## Pricing

$0.001 per call (1000 μUSDC). No free tier.

## Versioning

- 1.0.0 (2026-10-06): initial release, migrated to SKILL.md v1.1
