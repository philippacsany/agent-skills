---
name: openrouter-jev
license: MIT
description: Calls TypeSafe's Jev decisions model through OpenRouter and returns typed Choice, Noul, and Score answers with probabilities. Use when the user names Jev, TypeSafe, or OpenRouter Decisions, or edits code that calls them.
---

# Jev through OpenRouter

Jev is TypeSafe's **System One model**. It takes application state and typed questions
and returns typed answers with probabilities. It does not generate text or explanations.
Code owns the workflow; Jev supplies narrow judgments where plain code needs semantic
understanding.

This skill calls Jev **through OpenRouter**, so one `OPENROUTER_API_KEY` covers both
the agent model and the judgments.

## Scope

- Use this skill when the user asks for Jev or TypeSafe, or asks you to edit code that
  already calls them. If a task could use Jev but the user hasn't mentioned it, solve it
  with the user's existing approach. Don't pitch Jev, brainstorm Jev use cases, or swap
  an existing LLM call for a Jev call unless asked.
- Keep the change to what the user asked for. No speculative questions, new pipelines,
  or architecture changes they didn't request.
- Jev is not a replacement for a coding agent's or chat app's LLM. It answers typed
  questions only.

## Transport

Two routes exist on OpenRouter. Pick by what the user's code already uses.

| Code uses | Route | Details |
| --- | --- | --- |
| Nothing yet, plain HTTP, or the OpenRouter SDKs | `POST https://openrouter.ai/api/alpha/decisions` (default) | [references/decisions-api.md](references/decisions-api.md) |
| The TypeSafe SDK (`TypeSafeClient`, `choice()`/`noul()`/`score()`) | `POST https://openrouter.ai/api/v1/systemone` via the SDK's base URL setting | [references/sdk-clients.md](references/sdk-clients.md) |

Facts that apply to both routes:

- Jev **cannot** be called through `/api/v1/chat/completions`. It returns a 400 with
  "is a decisions model and cannot be used with the chat/completions endpoint". Don't
  route it through OpenAI-compatible clients or LiteLLM chat calls.
- **Model IDs:** `typesafe/jev-1.13` (pinned; use in production, especially once
  thresholds are tuned) and `~typesafe/jev-latest` (alias; fine for prototypes). The
  response `model` field reports a dated snapshot such as `typesafe/jev-1.13-20260917`.
- **Limits and price:** 32k token context on OpenRouter (state plus questions). Input
  tokens are billed at $0.042 per million; output is free. `usage.cost` in the response
  is in USD.
- **Discovery:** Jev is absent from `GET /api/v1/models`. Check
  `GET https://openrouter.ai/api/v1/models/typesafe/jev-1.13/endpoints` for current
  pricing and status.
- **Keys:** keep `OPENROUTER_API_KEY` server-side, never in a browser bundle, and set a
  credit limit on the key.
- **Alpha endpoint.** If a call fails unexpectedly, re-read the
  [Decisions API reference](https://openrouter.ai/docs/api/api-reference/alphadecisions/submit-a-decisions-request.md)
  before changing code.

Minimal request (see the reference for the full schema, response shape, errors, and
SDK snippets):

```json
{
  "model": "typesafe/jev-1.13",
  "state": { "ticket": "My checkout page shows a blank screen after I click Pay." },
  "questions": {
    "is_bug": {
      "type": "noul",
      "instructions": "Is the customer reporting a software defect?",
      "criteria": {
        "true": "The customer describes broken or unexpected product behavior.",
        "false": "The customer is asking a question or requesting a feature."
      }
    }
  }
}
```

Answer: `{"answers": {"is_bug": {"type": "noul", "noul": 0.95}}, "usage": {...}}`.

## Design the judgments

TypeSafe's docs are the reference for *what to ask*. Read them for question design, not
for transport: [primitives](https://docs.typesafe.ai/primitives.md),
[state](https://docs.typesafe.ai/concepts/state.md),
[confidence](https://docs.typesafe.ai/confidence.md),
[advanced structure](https://docs.typesafe.ai/primitives/advanced.md),
[known weak spots](https://docs.typesafe.ai/model-jaggedness/jev-1.13.md). Browse the
[docs index](https://docs.typesafe.ai/llms.txt) for cookbooks only when the user asks
for patterns. Ignore pages about TypeSafe's own HTTP API and SDK transport.

| Need | Primitive | Notes |
| --- | --- | --- |
| One of a defined set | Choice | Picks one option; `probabilities` compare the options. Up to 255 options. Add a no-match option if nothing may fit. |
| Whether a condition holds | Noul | Probability of yes. No separate confidence. Use one Noul per label when several may apply. |
| Degree along one dimension | Score | Probability-weighted position on ordered levels. Two to ten levels; three is a good default. |

- Give each question enough **state** to answer: source text, identities,
  relationships, policies, and current facts. Use named JSON fields when state has
  several parts. Reference nested state in `instructions` with backticked paths such as
  `` `ticket.messages[0].text` ``. Jev reads text only, primarily English.
- Put the judgment in `instructions` and define the answers in `criteria`. Question IDs
  aren't sent to the model, so the question text must carry the full meaning. Choice
  option names and descriptions are both sent, so write descriptions that separate them.
- Score levels describe **situations, not degrees**. The model sees each level alone,
  without its number or neighbours, so "worse than the previous level" and numeric
  labels carry no meaning.
- Ask one narrow, atomic judgment per question. Questions in one request run in
  parallel over the same state and can't see each other's answers. Ask independent
  questions together; make a second request only when an earlier answer is needed to
  build the next state or options.
- Keep rules, calculations, counting, date comparison, exact lookups, and execution in
  code. Those are documented weak spots for the model.

## Use the answers

- Gate actions on probabilities and confidence, with thresholds chosen for the user's
  data and consequences. Start conservative and tune against real cases.
- Confidence measures how concentrated the distribution is. It isn't overall
  correctness or permission to act. Several acceptable alternatives can spread
  probability, so low confidence needn't invalidate a harmless preference choice.
- A Noul near 0.5 means yes and no are about equally likely, not "medium intensity."
- A Score is a float on the level index scale (for example 1.9 on a three-level scale).
  Don't read the fraction as an exact magnitude between levels.
- Identical inputs can move probabilities by several hundredths between calls. Leave
  margin around thresholds rather than setting them at an observed value.
- Read answers defensively: the contract guarantees `type` plus `choice`, `score`, or
  `noul`. Treat `probabilities`, `confidence`, and `legend` as optional.
- Typed output guarantees the interface, not the truth. Test on representative cases.
  When something fails, inspect the exact state, questions, answers, and consuming
  code. Separate model errors from code errors and OpenRouter or provider failures.
