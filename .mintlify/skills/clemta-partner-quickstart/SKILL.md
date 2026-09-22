---
name: clemta-partner-quickstart
description: Integrate the Clemta Partner API. Use when building or debugging code that creates companies, orders services, handles requirements, or consumes events from api.clemta.com. Covers auth, test vs live mode, idempotency, versioning, error handling, and the sandbox.
license: Proprietary
compatibility: Any HTTP client. No SDK required. Examples use curl.
metadata:
  author: clemta
  version: "1.0"
  product: partner-api
  audience: developers
  related: clemta-partner-webhooks, clemta
---

# Clemta Partner API

Base URL: `https://api.clemta.com/v1`. Full reference at https://docs.clemta.com, OpenAPI at
`https://api.clemta.com/v1/openapi.json` (no key needed). Any docs page is available as
Markdown by appending `.md` to its URL, e.g. `https://docs.clemta.com/partner/webhooks.md`.

## Rules that apply to every request

1. **Auth**: `Authorization: Bearer <key>`. Keys start with `clmt_test_` (sandbox, creates nothing real)
   or `clmt_live_` (real companies, real billing). Default to a test key while developing.
2. **Idempotency**: send a UUID in `Idempotency-Key` on every `POST`, `PUT`, `DELETE`. Generate it once
   per operation, persist it with the work, reuse it on every retry. Never generate it inside the retry loop.
   A replay returns the original response with `Idempotent-Replayed: true`. Keys live 24 hours.
3. **Version**: pin `Clemta-Version: 2026-08-13` on every request. The effective version is echoed
   back in the same header. Unknown dates return `invalid_api_version`.
4. **Errors**: every error is JSON with a stable `code`. Branch on `code`, never on `detail`. Validation
   failures (`invalid_request`) carry an `errors` array naming every bad field. Retry only on timeouts,
   connection errors, `5xx`, and `429` (honor `Retry-After`). Full list: https://docs.clemta.com/partner/errors.md
5. **Ids** are prefixed: `acct_`, `cmp_`, `so_`, `txf_`, `rqmt_`, `file_`, `evt_`. Every resource carries an
   `object` field naming its type. A `404` (`resource_missing`) is returned both for unknown ids and for
   resources that belong to someone else.
6. **Test and live never mix**: an id from one mode never resolves in the other, webhook endpoints are per
   mode, and `/sandbox/*` only accepts test keys (`test_mode_only` otherwise).

## Object model

Account (your customer) owns Companies. A Company has Service orders (products from your wholesale
catalog), Tax filings, Requirements (things needed from you or your customer), and Files (deliverables Clemta
publishes). Everything emits Events, delivered by webhook and readable on `GET /events`.

Design rule: the customer is the partner's. Clemta never contacts them. Anything that would reach the
customer reaches the partner as an event instead.

## Step 1: confirm the key

```bash
curl https://api.clemta.com/v1/me -H "Authorization: Bearer clmt_test_..."
# -> { "object": "me", "partner_id": "prt_...", "livemode": false, "api_version": "2026-08-13" }
```

## Step 2: create a company

One call creates the account, the company, and the services on it.

```bash
curl -X POST https://api.clemta.com/v1/companies \
  -H "Authorization: Bearer clmt_test_..." \
  -H "Idempotency-Key: $(uuidgen)" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "Acme", "ending": "llc", "state": "DE", "entity_type": "llc", "industry": "software",
    "account": {"first_name": "Ada", "last_name": "Lovelace", "email": "ada@acme.example"},
    "shareholders": [{"type": "individual", "first_name": "Ada", "last_name": "Lovelace",
                      "email": "ada@acme.example",
                      "relationship": {"title": "sole_owner", "percent_ownership": 100, "representative": true},
                      "tax_id_type": "ssn", "tax_id": "123-45-6789",
                      "address": {"country": "US", "line1": "1 Main St", "city": "Dover", "state": "DE", "postal_code": "19901"}}],
    "services": ["ein"]
  }'
```

The company comes back with `status: requires_information`: each owner still owes an identity document.
Either upload one via `POST /files` and attach it as `passport`, or create a hosted page with
`POST /companies/{id}/verification-sessions` and forward its `url` to the customer (no key needed on their side).
Use `external_id` on accounts and companies to map them to your own records. A reused `external_id`
returns `resource_already_exists`.

## Step 3: follow the lifecycle

Use webhooks where you can. Deliveries are signed per Standard Webhooks (`webhook-id`, `webhook-timestamp`,
`webhook-signature` headers). Verify against the raw body, dedupe on `webhook-id` (the event id), respond
`2xx` fast. Events arrive in order per endpoint. See the `clemta-partner-webhooks` skill for the local loop.

Polling is a full alternative and a safety net. Events never expire:

```bash
curl "https://api.clemta.com/v1/events?created_at[gte]=<last_sweep-5m>&sort=created_at&limit=100" \
  -H "Authorization: Bearer clmt_test_..."
# follow after=<end_cursor> until has_next_page is false. Process each event once, keyed by its id
```

Lists return current state only, so re-listing does not show what changed. The event stream is the only
change feed.

## Step 4: drive a test company

A test company never advances on its own. Simulate the change you want, and the same events fire as
in live mode:

```bash
curl -X POST https://api.clemta.com/v1/sandbox/companies/cmp_.../simulate \
  -H "Authorization: Bearer clmt_test_..." -H "Content-Type: application/json" \
  -d '{"event": "company.status.changed", "status": "active"}'   # fires company.incorporated too
```

Other simulable events: `company.verified`, `company.ein.assigned` (`ein`), `company.name.changed` (`name`),
`service_order.status.changed` (`order`, `step`, `form_required`, `completed`), `service_order.completed`
(`order`), `tax_filing.created` / `tax_filing.status.changed` (`tax_filing`), `file.created` (`file_name`),
`requirement.created` (`message`), `invoice.finalized`. Once-only rules apply exactly as in live.
The sandbox clock (`GET/POST /clock`, `POST /clock/advance`) moves time for renewals and deadlines.

## Common mistakes to avoid

- Branching on the `company.status.changed` event type instead of the `status` inside it.
- Re-serializing the webhook body before verifying the signature.
- Reusing one `Idempotency-Key` for two different operations (`idempotency_key_mismatch`).
- Calling `/sandbox/*` with a live key.
- Treating a `5xx` as settled: the idempotency key is released, so retry it.

## Read next (Markdown)

- https://docs.clemta.com/partner/concepts.md
- https://docs.clemta.com/partner/company-lifecycle.md
- https://docs.clemta.com/partner/requirements.md
- https://docs.clemta.com/partner/service-orders.md
- https://docs.clemta.com/partner/list-queries.md
- https://docs.clemta.com/partner/webhooks.md

## Related skills

- `clemta-partner-webhooks`: the webhook loop end to end. Tunnel, simulate, verify, reconcile by polling.
- `clemta`: the index of every Clemta skill.
