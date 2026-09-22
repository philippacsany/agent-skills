# OpenRouter Decisions API reference (alpha)

Official reference:
https://openrouter.ai/docs/api/api-reference/alphadecisions/submit-a-decisions-request.md

```http
POST https://openrouter.ai/api/alpha/decisions
Authorization: Bearer $OPENROUTER_API_KEY
Content-Type: application/json
HTTP-Referer: https://your-app.example        # optional, enables app attribution
X-OpenRouter-Title: Your App                  # optional, needs HTTP-Referer to matter
```

The path is `/api/alpha/decisions`. Both `/api/v1/alpha/decisions` and
`/api/v1/api/alpha/decisions` return 404. `X-Title` is the legacy name of the title
header and still works.

## Request body

Required: `model`, `state`, `questions`. Optional: `provider`, `session_id`, `trace`, `user`.

| Field | Type | Notes |
| --- | --- | --- |
| `model` | string | `typesafe/jev-1.13` or `~typesafe/jev-latest` |
| `state` | string, object, or array | The content to evaluate. Prefer an object with named fields. |
| `questions` | object keyed by question ID | Each has `type` of `noul`, `choice`, or `score` |
| `provider` | ProviderPreferences | Routing preferences. Inert today: Jev has one endpoint (TypeSafe). |
| `session_id` | string, max 256 | Groups related requests for observability. Never sent to the provider. |
| `trace` | object | `trace_id`, `trace_name`, `span_name`, `generation_name`, `parent_span_id`, plus custom keys |
| `user` | string, max 256 | End-user identifier |

### Question shapes

`instructions` and every criteria value accept a plain string, or a JSON object or
array of structured guidance.

| Type | Required | `criteria` shape |
| --- | --- | --- |
| `noul` | `type`, `instructions` | Optional. If present, an object with **both** `"true"` and `"false"`. A single key fails with a 400 `invalid_union`. |
| `choice` | `type`, `instructions`, `criteria` | Object mapping option name to description (or `null`). Up to 255 options. |
| `score` | `type`, `instructions`, `criteria` | Array of level descriptions, ordered low to high. Two to ten levels. |

### Full example

```json
{
  "model": "typesafe/jev-1.13",
  "state": {
    "customer_tier": "enterprise",
    "ticket": "My checkout page shows a blank screen after I click Pay."
  },
  "questions": {
    "is_bug": {
      "type": "noul",
      "instructions": "Is the customer reporting a software defect?",
      "criteria": {
        "true": "The customer describes broken or unexpected product behavior.",
        "false": "The customer is asking a question or requesting a feature."
      }
    },
    "team": {
      "type": "choice",
      "instructions": "Which team should own this ticket?",
      "criteria": {
        "account": "Login, permissions, or profile issues.",
        "frontend": "Rendering, layout, or browser compatibility issues.",
        "payments": "Checkout, billing, or payment processing issues."
      }
    },
    "urgency": {
      "type": "score",
      "instructions": "How urgent is this ticket?",
      "criteria": [
        "Can wait for the next release",
        "Should be fixed this week",
        "Blocking revenue right now"
      ]
    }
  }
}
```

## Response

```json
{
  "id": "gen-dec-1789738314-X5e5eKGQdvR9rblyX250",
  "model": "typesafe/jev-1.13-20260917",
  "provider": "TypeSafe",
  "answers": {
    "is_bug": { "type": "noul", "noul": 0.96 },
    "team": {
      "type": "choice", "choice": "payments", "confidence": 0.75,
      "probabilities": { "account": 0, "frontend": 0.16, "payments": 0.84 }
    },
    "urgency": {
      "type": "score", "score": 1.99, "confidence": 0.99,
      "legend": { "0": "Can wait for the next release", "1": "Should be fixed this week", "2": "Blocking revenue right now" },
      "probabilities": { "0": 0, "1": 0.01, "2": 0.99 }
    }
  },
  "usage": { "input_tokens": 476, "output_tokens": 70, "cost": 0.000019992 }
}
```

Schema guarantees per answer:

| Type | Guaranteed | Optional in the contract |
| --- | --- | --- |
| `noul` | `type`, `noul` (0 to 1) | nothing else; no `confidence` or `probabilities` |
| `choice` | `type`, `choice` | `confidence`, `probabilities` |
| `score` | `type`, `score` (float on the level-index scale) | `confidence`, `probabilities`, `legend` |

`usage.input_tokens` and `usage.output_tokens` are guaranteed; `usage.cost` (USD) is
optional in the schema but present in practice. Look a call up later with
`GET https://openrouter.ai/api/v1/generation?id=<id>`.

## Errors

Body shape: `{"error": {"code": N, "message": "..."}}`, sometimes with `metadata`.

| Code | Meaning | Action |
| --- | --- | --- |
| 400 | Validation failed (TypeSafe's native API uses 422 for the same thing) | Fix the request |
| 401 | Bad or missing key | Fix credentials |
| 402 | Payment issue. Branch on `error.metadata.limit_source`: `openrouter_in_flight_budget` (wait and retry; `Retry-After` is set), `openrouter_key_limit` (key credit limit hit), `openrouter_credits` (balance too low) | SDKs never retry 402 on their own |
| 403 | Forbidden or moderation | Don't retry |
| 413 | Request too large (32k context) | Trim state |
| 429 | Rate limited; `Retry-After` set | Backoff |
| 500 | OpenRouter error | Backoff |
| 502 | Provider down or invalid provider response | Backoff |
| 503 | No provider meets routing requirements; `Retry-After` set | Backoff |
| 524 | Infrastructure timeout | Backoff |
| 529 | Provider overloaded | Backoff |

Retry 429, 500, 502, 503, 524, and 529 with exponential backoff and honor
`Retry-After`. The OpenRouter SDKs already do this for everything except 402.

Rate limit tables in OpenRouter's docs apply to `:free` models only. Jev is paid, so
there are no published per-key caps beyond Cloudflare protection. TypeSafe's own limits
(1,200 requests/min, 250k tokens/sec) can change without notice.

## Quick manual test

```bash
curl -s https://openrouter.ai/api/alpha/decisions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" -H 'Content-Type: application/json' \
  -d '{"model":"typesafe/jev-1.13","state":"Help! My payouts have been failing for 3 days.",
       "questions":{"is_urgent":{"type":"noul","instructions":"Does this convey urgency?"}}}'
```

Check the model's endpoint metadata and price without a key:

```bash
curl -s https://openrouter.ai/api/v1/models/typesafe/jev-1.13/endpoints | jq '.data.endpoints[0] | {name, pricing, context_length, status}'
```
