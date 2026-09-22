---
name: clemta
description: Index of every Clemta skill. Load this first when a task mentions Clemta and it is not clear which product or skill applies. Points to the right skill by product and task.
license: Proprietary
metadata:
  author: clemta
  version: "1.0"
  audience: all
---

# Clemta skills

Clemta forms US companies and runs the services around them. Skills are named
`clemta-<product>-<topic>`. Load the one that matches the task. Each skill is
self-contained and names the others it pairs with.

## Partner API

The API partners use to form companies for their own customers under their own
brand. Base URL `https://api.clemta.com/v1`. Docs: https://docs.clemta.com/partner/introduction.md

| Task | Skill |
|---|---|
| Build or debug an integration: auth, create companies, order services, requirements, events, sandbox | `clemta-partner-quickstart` |
| Write or debug a webhook handler: tunnel, simulate, verify signatures, reconcile by polling | `clemta-partner-webhooks` |

## Reading the docs

- Any docs page as Markdown: append `.md` to its URL, e.g. `https://docs.clemta.com/partner/webhooks.md`.
- Full index: https://docs.clemta.com/llms.txt
- Docs search MCP server: `https://docs.clemta.com/mcp`
- OpenAPI, no key needed: `https://api.clemta.com/v1/openapi.json`

## Rules that hold everywhere

- Keys starting `clmt_test_` are sandbox and touch nothing real. Default to one while developing.
- Never place a key in a prompt, a skill file, or a committed file.
- Send a UUID `Idempotency-Key` on every write. Branch on error `code`, never on `detail`.
