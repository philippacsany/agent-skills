# Client code

Three ways to call Jev on OpenRouter. Match the user's existing stack; don't rewrite a
working client to switch routes.

| Client | Route | When |
| --- | --- | --- |
| OpenRouter SDK (Python `openrouter`, npm `@openrouter/sdk`) | `/api/alpha/decisions` | Default for new code. Includes retries. |
| Plain HTTP | `/api/alpha/decisions` | Scripts, edge runtimes, anything without an SDK |
| TypeSafe SDK with the base URL pointed at OpenRouter | `/api/v1/systemone` | Code that already uses `TypeSafeClient` / `choice()` / `noul()` / `score()` |

## OpenRouter SDK: Python

Package `openrouter` (Python 3.10+). In a project: `uv add openrouter`. For a standalone
script, declare the dependency inline (PEP 723) and run it with `uv run script.py`;
never `uv run python script.py`, which skips the inline metadata.

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["openrouter==1.2.17"]
# ///
import os
from openrouter import OpenRouter

ticket_text = "My checkout page shows a blank screen after I click Pay."

with OpenRouter(api_key=os.environ["OPENROUTER_API_KEY"]) as client:
    res = client.alpha.decisions.create(
        model="typesafe/jev-1.13",
        state={"ticket": ticket_text},
        questions={
            "is_bug": {
                "type": "noul",
                "instructions": "Is the customer reporting a software defect?",
            }
        },
        http_referer="https://your-app.example",   # optional attribution
        x_open_router_title="Your App",            # optional attribution
    )
    p_bug = res.answers["is_bug"].noul
    print(p_bug, res.usage.cost)
```

```bash
OPENROUTER_API_KEY=... uv run jev_check.py
```

Method: `client.alpha.decisions.create(...)` with flat keyword arguments. Async twin:
`create_async`. Optional kwargs: `provider`, `session_id`, `trace`, `user`, `retries`
(`openrouter.utils.RetryConfig`), `timeout_ms`, `server_url`. Errors are typed
(`errors.BadRequestResponseError` through `errors.ProviderOverloadedResponseError`,
fallback `errors.OpenRouterDefaultError`).

## OpenRouter SDK: TypeScript

`npm install @openrouter/sdk`. The body goes under `decisionsRequest`; `usage` comes back
in camelCase (`inputTokens`, `outputTokens`, `cost`).

```ts
import { OpenRouter } from "@openrouter/sdk";

const openRouter = new OpenRouter({
  apiKey: process.env.OPENROUTER_API_KEY ?? "",
  httpReferer: "https://your-app.example", // optional attribution
  appTitle: "Your App",                    // optional attribution
});

const res = await openRouter.alpha.decisions.create({
  decisionsRequest: {
    model: "typesafe/jev-1.13",
    state: { ticket: ticketText },
    questions: {
      is_bug: { type: "noul", instructions: "Is the customer reporting a software defect?" },
    },
  },
});
const pBug = res.answers["is_bug"].noul;
```

A tree-shakable form exists: `alphaDecisionsCreate(core, request)` from
`@openrouter/sdk/funcs/alphaDecisionsCreate.js`, returning `{ ok, value | error }`
instead of throwing.

### Base URL note for older SDK versions

Python `openrouter` before 1.2 and `@openrouter/sdk` before 1.3 built the Decisions URL
from the client's default base URL (`https://openrouter.ai/api/v1`), producing
`/api/v1/api/alpha/decisions` and a 404. Current versions hardcode
`https://openrouter.ai` for this operation, so client-level `server_url` / `serverURL`
is ignored.

- Preferred fix: upgrade the SDK.
- If the version is pinned: pass `server_url="https://openrouter.ai"` (Python) or
  `serverURL: "https://openrouter.ai"` (TypeScript) on a **separate** client used only
  for decision calls. Harmless on new versions.
- Never pass a per-call `server_url` / `options.serverURL` ending in `/api/v1`. That
  reproduces the 404 on every version.

If an installed version's method names differ, check its `alpha.decisions` reference
([Python](https://openrouter.ai/docs/client-sdks/python/sdks/decisions/README.md),
[TypeScript](https://openrouter.ai/docs/client-sdks/typescript/sdks/decisions/README.md))
instead of guessing.

## Plain HTTP

```ts
const res = await fetch("https://openrouter.ai/api/alpha/decisions", {
  method: "POST",
  headers: {
    Authorization: `Bearer ${process.env.OPENROUTER_API_KEY}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify({ model: "typesafe/jev-1.13", state, questions }),
});
if (!res.ok) throw new Error(`${res.status}: ${await res.text()}`);
const { answers, usage } = await res.json();
```

Validate the response shape (for example with Zod or Pydantic) and fail loudly if a
question key is missing from `answers`. Implement backoff for 429/5xx yourself.

## TypeSafe SDK pointed at OpenRouter

OpenRouter exposes `POST https://openrouter.ai/api/v1/systemone`, which speaks the
TypeSafe OpenAPI contract. Guide:
https://openrouter.ai/docs/guides/community/typesafe-sdk.md. TypeSafe documents the
same setup under "Configuring the base URL" in
https://docs.typesafe.ai/sdk/python/usage.md.

The Python package is `typesafe-sdk` (import `typesafe_sdk`). Not `typesafe`, which is
an unrelated package. In a project: `uv add typesafe-sdk`.

Set the base URL to `https://openrouter.ai/api` (the SDK appends `/v1/systemone`) and
use the OpenRouter key. Model IDs map automatically: `jev-1.13` routes as
`typesafe/jev-1.13`, `jev-latest` as `~typesafe/jev-latest`; IDs that already carry an
author prefix pass through.

```python
# /// script
# requires-python = ">=3.12"
# dependencies = ["typesafe-sdk==0.7.1"]
# ///
import os
from typesafe_sdk import Noul, TypeSafeClient

with TypeSafeClient(
    api_key=os.environ["OPENROUTER_API_KEY"],
    base_url="https://openrouter.ai/api",
    model="typesafe/jev-1.13",
) as client:
    result = client.system_one(
        "I was charged twice.",
        {"billing": Noul(instructions="Is this about billing?")},
    )
    print(result.nouls["billing"].noul)
```

```bash
OPENROUTER_API_KEY=... uv run jev_via_typesafe.py
```

Or set `TYPESAFE_BASE_URL=https://openrouter.ai/api` and
`TYPESAFE_API_KEY=$OPENROUTER_API_KEY` in the environment and leave existing code
unchanged. The SDK's default model is `jev-latest`, which OpenRouter maps to the alias.

Known differences on this route:

- OpenRouter adds `id`, `provider`, and `usage.cost` to responses. The TypeSafe SDKs
  pass them through without error.
- The TypeSafe SDK's model listing (`client.models.list()`) calls `GET /api/v1/models`,
  which returns OpenRouter's shape and fails. Don't call it on this route.
- Context is 32k on OpenRouter versus 64k on TypeSafe's native API.
- `POST /api/systemone` without `/v1` returns 404. The `/v1` prefix is correct here,
  unlike the Decisions route.
