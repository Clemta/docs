---
name: clemta-partner-webhooks
description: Receive, trigger, and verify Clemta Partner API webhooks against a local handler. Use when writing a webhook endpoint for api.clemta.com, debugging signature failures, replaying missed events, or testing handler paths without a real formation.
license: Proprietary
compatibility: Any language. Examples use Node and Python with the standardwebhooks library.
metadata:
  author: clemta
  version: "1.0"
  product: partner-api
  audience: developers
  related: clemta-partner-quickstart, clemta
---

# Debug Clemta webhooks locally

Everything here runs in test mode with a `clmt_test_` key. No live company is touched.

## The loop

1. **Run the handler locally**, then expose it with a tunnel (`ngrok http 4000`, `cloudflared tunnel`).
   Endpoints must be HTTPS. A free tunnel's URL changes on restart, so re-register when it does.
2. **Register the tunnel URL** as a test-mode endpoint on the Webhooks page of the partner dashboard
   (https://partner.clemta.com). Store the `whsec_...` secret, it is shown once. Endpoints are per mode:
   a test endpoint only ever receives test events.
3. **Create a test company** (fires `company.created` immediately):
   ```bash
   curl -X POST https://api.clemta.com/v1/companies \
     -H "Authorization: Bearer clmt_test_..." -H "Idempotency-Key: $(uuidgen)" \
     -H "Content-Type: application/json" \
     -d '{"external_id":"dev-1","name":"Dev Co","ending":"llc","state":"DE","entity_type":"llc","industry":"software",
          "account":{"external_id":"cust-1","email":"founder@dev.test"}}'
   ```
4. **Fire the events you want** with the sandbox. The change is applied exactly as a live one:
   ```bash
   # company.status.changed + company.incorporated
   curl -X POST https://api.clemta.com/v1/sandbox/companies/cmp_.../simulate \
     -H "Authorization: Bearer clmt_test_..." -H "Content-Type: application/json" \
     -d '{"event": "company.status.changed", "status": "active"}'
   # company.ein.assigned (fires once)
   curl -X POST https://api.clemta.com/v1/sandbox/companies/cmp_.../simulate \
     -H "Authorization: Bearer clmt_test_..." -H "Content-Type: application/json" \
     -d '{"event": "company.ein.assigned", "ein": "12-3456789"}'
   ```
5. **Verify each delivery** (below), dedupe on the event id, respond `2xx`.

## Verifying a delivery

Deliveries follow the Standard Webhooks spec. Three headers, each sent under two names:
`Clemta-Webhook-Id` / `webhook-id`, `Clemta-Webhook-Timestamp` / `webhook-timestamp`,
`Clemta-Webhook-Signature` / `webhook-signature`. The signature header is a space-delimited list:
`v1,<base64 HMAC-SHA256>` (symmetric, keyed with the secret base64-decoded after `whsec_`) and
`v1a,<base64 ed25519>` (asymmetric, verifiable with the endpoint's `whpk_` public key). Both sign
the string `<id>.<timestamp>.<raw body>`. Match any one entry of the scheme you verify. During a secret
rotation two `v1` entries are present and either verifies.

Shortest path: the standardwebhooks library.

```js
import { Webhook } from "standardwebhooks";
const wh = new Webhook(process.env.CLEMTA_WEBHOOK_SECRET); // whsec_...
app.post("/webhooks/clemta", express.raw({ type: "*/*" }), (req, res) => {
  let event;
  try {
    event = wh.verify(req.body, {           // req.body must be the RAW bytes
      "webhook-id": req.header("webhook-id"),
      "webhook-timestamp": req.header("webhook-timestamp"),
      "webhook-signature": req.header("webhook-signature"),
    });
  } catch { return res.sendStatus(400); }
  // dedupe on event.id, then switch on event.type and read event.data.object
  res.sendStatus(200);
});
```

```python
from standardwebhooks import Webhook
wh = Webhook(os.environ["CLEMTA_WEBHOOK_SECRET"])  # whsec_...

@app.post("/webhooks/clemta")
async def handle(request):
    body = await request.body()  # RAW bytes, before any JSON middleware
    try:
        event = wh.verify(body, dict(request.headers))
    except Exception:
        return Response(status_code=400)
    # dedupe on event["id"], then handle event["type"] / event["data"]["object"]
    return Response(status_code=200)
```

## Diagnosing failures

| Symptom | Likely cause | Fix |
|---|---|---|
| Every signature fails | Body was parsed and re-serialized before verification | Read the raw bytes. Mount the raw body parser before any JSON middleware on this route |
| Signature fails only after a secret roll | Handler holds the old secret | Switch to the new secret during the overlap window. Both verify until it ends |
| Timestamp rejected | Clock skew or a captured replay | Tolerance of a few minutes is normal; a fresh retry is re-signed with a new timestamp |
| Same event arrives twice | Retry or poll overlap | Dedupe on `webhook-id`, it is stable across retries |
| Later events never arrive | An earlier event is still being rejected | Deliveries are ordered per endpoint. Acknowledge or fix the stuck one and the rest follow |
| Nothing arrives at all | Tunnel URL rotated, endpoint registered in the wrong mode, or a `3xx` from the handler | Re-register the endpoint, check the mode, and return a 2xx, not a redirect |
| `test_mode_only` from `/sandbox/*` | Live key | Use a `clmt_test_` key |

Answering `410 Gone` disables the endpoint. An endpoint failing continuously for days is disabled
automatically and the workspace owner is emailed. No events are lost.

## Without a tunnel

Every webhook is fanned out from the same log `GET /events` reads, so simulate and then pull:

```bash
curl "https://api.clemta.com/v1/events?limit=10&sort=-created_at" -H "Authorization: Bearer clmt_test_..."
```

This is also the reconciliation recipe for production: sweep
`GET /events?created_at[gte]=<last_sweep - 5m>&sort=created_at`, follow `after=<end_cursor>` until
`has_next_page` is false, process idempotently by event id. Narrow with `?type=`, `?type[in]=a,b`,
or `?company_id=`.

## Payload shape

Each event carries `type` and `data.object`, the affected resource in its current API shape, rendered at
the workspace's webhook version (set in the dashboard, defaults to the account's pinned API version).
Switch on `type`, read `data.object.object` to know what you are holding. For
`company.status.changed` branch on the status inside the payload, not on the type.

## Read next (Markdown)

- https://docs.clemta.com/partner/local-development.md
- https://docs.clemta.com/partner/webhooks.md
- https://docs.clemta.com/partner/concepts.md

## Related skills

- `clemta-partner-quickstart`: the integration itself. Auth, idempotency, versioning, errors, creating companies, the sandbox.
- `clemta`: the index of every Clemta skill.
