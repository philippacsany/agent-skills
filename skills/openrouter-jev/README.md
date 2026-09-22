# openrouter-jev

A skill that lets your coding agent call [Jev](https://openrouter.ai/typesafe), TypeSafe's
decisions model, through [OpenRouter](https://openrouter.ai).

Jev doesn't write text. You give it some state and typed questions, and it returns
typed answers with probabilities:

- **Noul:** how likely is it that a condition holds? (`0.95`)
- **Choice:** which option from a set fits best? (`"payments"`)
- **Score:** where does this sit on a scale? (`1.9` out of `0`–`2`)

This skill is adapted from TypeSafe's official
[`typesafe-ai` skill](https://github.com/typesafe-ai/skills), with two changes:

- **It calls Jev through OpenRouter** instead of TypeSafe's own API, so you only need
  your `OPENROUTER_API_KEY`.
- **It only activates when you ask for Jev**, TypeSafe, or OpenRouter Decisions. It won't
  suggest Jev for tasks where you didn't ask for it.

▶️ Watch the setup in the video: [Jev and Claude Code](https://youtu.be/s4skNgV8nJM)

## Setup

You need an [OpenRouter](https://openrouter.ai) account with some credits, and Claude
Code or another agent that supports skills.

### 1. Create an OpenRouter API key

Go to [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys) and create a
key. Set a credit limit on it, so a runaway script can't drain your balance.

If you already use OpenRouter and `OPENROUTER_API_KEY` is set in your environment,
skip to [step 3](#3-test-the-connection).

### 2. Add the key to your environment

In the terminal, go to the folder where you'll start Claude Code later, then export
the key:

```bash
export OPENROUTER_API_KEY="sk-or-v1-..."
```

This only lasts for the current terminal session. To keep the key across sessions, add
the same line to your shell profile (`~/.zshrc` on macOS, `~/.bashrc` on most Linux
setups) and open a new terminal.

Start Claude Code from a terminal where the key is set. Claude Code passes its
environment on to the commands it runs, so that's how the skill's code gets the key.

### 3. Test the connection

Before involving your agent, check that the key works and Jev answers:

```bash
curl -s https://openrouter.ai/api/alpha/decisions \
  -H "Authorization: Bearer $OPENROUTER_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "~typesafe/jev-latest",
    "state": "Help! My payouts have been failing for 3 days.",
    "questions": {
      "is_urgent": {
        "type": "noul",
        "instructions": "Does this message convey urgency?"
      },
      "team": {
        "type": "choice",
        "instructions": "Which team should handle this message?",
        "criteria": {
          "billing": "Payments, payouts, invoices, or refunds.",
          "technical": "Bugs, errors, or outages in the product.",
          "account": "Login, permissions, or profile settings."
        }
      }
    }
  }'
```

You should get a JSON response within a second or so, with an `answers` object like
this:

```json
"answers": {
  "is_urgent": { "type": "noul", "noul": 0.97 },
  "team": { "type": "choice", "choice": "billing", "confidence": 0.9, "probabilities": { ... } }
}
```

The exact numbers vary slightly between calls. If you get an error instead:

- **`401`:** The key is missing or wrong. Run `echo $OPENROUTER_API_KEY` to check that
  it's set in this terminal.
- **`402`:** Your account is out of credits, or the key hit its credit limit.

### 4. Install the skill

Install the skill for your user, so it's available in every project:

```bash
npx skills add philippacsany/agent-skills --skill openrouter-jev -g -a claude-code
```

Leave out `-a claude-code` to pick a different agent, and leave out `-g` to install the
skill into the current project only.

Or install it by hand by copying this folder into your Claude Code skills directory:

```bash
git clone https://github.com/philippacsany/agent-skills.git
cp -r agent-skills/skills/openrouter-jev ~/.claude/skills/
```

### 5. Check that Claude Code sees the skill

Start Claude Code, or run `/reload-skills` if it's already running. Then run
`/skills` and look for **openrouter-jev** in the list.

If you installed TypeSafe's official plugin earlier, uninstall it through `/plugin` →
**Installed** → **typesafe** → **Uninstall**. Otherwise, the two skills compete for the
same requests.

## Use it

Mention Jev in your prompt, and Claude Code loads the skill. For example:

> Use Jev to route the support tickets in `tickets.json` to the billing, technical, or
> account team, and flag the urgent ones.

> Add a Jev check to `moderate.py` that scores how toxic a comment is before we publish
> it.

The skill tells your agent how to call Jev through OpenRouter, how to write good
questions, and how to use the probabilities it gets back. For question design, the
skill points your agent to [TypeSafe's docs](https://docs.typesafe.ai), so you can
also ask things like "What's the difference between a Choice and a Score in Jev?"

## What's in this folder

| File | Contents |
| --- | --- |
| [`SKILL.md`](SKILL.md) | The instructions your agent loads: when to use Jev, how to call it, and how to design questions |
| [`references/decisions-api.md`](references/decisions-api.md) | OpenRouter's Decisions API: request and response shapes, errors, and retries |
| [`references/sdk-clients.md`](references/sdk-clients.md) | Code for the OpenRouter SDKs (Python, TypeScript), plain HTTP, and the TypeSafe SDK |

Your agent reads the reference files only when it needs them.

## Costs

Jev costs $0.042 per million input tokens on OpenRouter, and output is free. The test
call above costs a fraction of a cent. Check the
[model page](https://openrouter.ai/typesafe) for current prices.

## License

[MIT](LICENSE), based on TypeSafe's MIT-licensed
[`typesafe-ai` skill](https://github.com/typesafe-ai/skills).
