# Gofox Outbound Webhooks

Push **real-time CRM events** from Gofox to your HTTPS endpoint. Configure a URL, verify signatures, process JSON.

> Sibling products: [REST API](https://github.com/gofoxcrm-ai/restapi) · [Tracking Code API](https://github.com/gofoxcrm-ai/trackingcodeapi) · [SSO](https://github.com/gofoxcrm-ai/sso)

**Phased docs:** [PHASES.md](./PHASES.md)

---

## Status overview

| Phase | Area | Status |
|-------|------|--------|
| 1 | Outbound CRM → your app | ✅ Live |
| 2 | Emit matrix & payload contract | ✅ Documented (matches code) |
| 3 | Inbound ESP/SMS webhooks (ops) | ✅ Live (Connected Apps / BYOK) |
| Media | Screenshots / video | 🚧 Placeholders |

> This product docs set is primarily **outbound** (Gofox → you). Phase 3 covers **inbound** provider callbacks for ESP/SMS configuration.

---

# Phase 1 — Outbound webhooks

## Setup

1. Plan feature **`webhooks`** (Prime+; aliased with `api_access`).
2. Gofox UI: **Account Settings → Webhooks**
3. Add HTTPS URL + select events
4. Copy the **signing secret** (shown once on create)
5. Use **Test** to send a sample payload

![Webhooks settings (placeholder)](docs/assets/webhooks-settings.png)

[Setup video (placeholder)](docs/assets/webhooks-setup.mp4)

## Events (allow-list)

| Event | Description |
|-------|-------------|
| `lead.created` | Lead created via **public ingest / form** pipelines |
| `contact.created` | Contact created |
| `contact.updated` | Contact updated |
| `deal.created` | Deal created |
| `deal.updated` | Deal updated |
| `ticket.created` | Support ticket created |
| `invoice.paid` | Invoice marked paid |
| `form.submitted` | Public form submission |

## Delivery request

```http
POST https://your-app.example.com/hooks/gofox
Content-Type: application/json
X-Gofox-Signature: <hex hmac-sha256 of raw body>
X-Gofox-Event: contact.created
```

```json
{
  "event": "contact.created",
  "organizationId": "org_…",
  "occurredAt": "2026-07-20T12:00:00.000Z",
  "data": {
    "id": "contact_…",
    "email": "ada@example.com",
    "firstName": "Ada"
  }
}
```

**Contract notes (as implemented):**
- Body fields: `event`, `organizationId`, `occurredAt`, `data`
- There is **no** `X-Gofox-Delivery-Id` header today
- Signature is HMAC-SHA256 hex of the **exact raw JSON body** using the subscription secret

## Verify signatures

### Node.js

```js
import crypto from 'node:crypto';

export function verifyGofoxSignature(rawBody, secret, header) {
  const expected = crypto.createHmac('sha256', secret).update(rawBody).digest('hex');
  const a = Buffer.from(expected, 'utf8');
  const b = Buffer.from(header || '', 'utf8');
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}
```

### Python

```python
import hmac, hashlib

def verify_gofox_signature(raw_body: bytes, secret: str, header: str) -> bool:
    expected = hmac.new(secret.encode(), raw_body, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, header or "")
```

Always read the **raw** body before JSON parsing.

## Retries & SSRF

- Non-2xx responses are retried with exponential backoff via **BullMQ** when `REDIS_URL` is set (5 attempts, starting ~10s). Without Redis, delivery is attempted synchronously.
- Request timeout: **10 seconds**
- Inspect delivery history in the Webhooks UI
- Target URL must be public **HTTPS** in production (`http` only when `NODE_ENV=development` or `ALLOW_HTTP_WEBHOOKS=true`)
- Private / link-local IPs are rejected (SSRF protection)

## Tenant management API

Authenticated with **session JWT** (Account Settings user), not a public API key:

```
GET    /api/v1/tenant/webhooks
POST   /api/v1/tenant/webhooks
PATCH  /api/v1/tenant/webhooks/:subscriptionId
DELETE /api/v1/tenant/webhooks/:subscriptionId
POST   /api/v1/tenant/webhooks/:subscriptionId/test
GET    /api/v1/tenant/webhooks/deliveries?subscriptionId=
```

`GET /` also returns `availableEvents`. Create response includes `secret` once.

Implementation: `gofox-server/src/modules/webhooks/` + `gofox-server/src/lib/outbound-webhooks.ts`.

---

# Phase 2 — Event emit matrix

What actually fires today (important for integrators):

| Event | Emitted from |
|-------|----------------|
| `lead.created` | Public lead **ingest** / form ingest services — **not** every UI/REST `createLead` path |
| `form.submitted` | Public form submission pipeline |
| `contact.created` / `contact.updated` | Contacts service |
| `deal.created` / `deal.updated` | Deals service |
| `ticket.created` | Tickets service |
| `invoice.paid` | Invoices service (paid status) |

If you need `lead.created` for every CRM create, prefer listening to ingest/forms or open a product request to emit from REST/UI creates as well.

### Adding events

1. Emit via `emitOutboundWebhook(...)` from the domain service
2. Add the name to `OUTBOUND_WEBHOOK_EVENTS` in `outbound-webhooks.ts`
3. Expose it in the Webhooks UI enum
4. Document the row here + sample `data` shape
5. Update [PHASES.md](./PHASES.md)

---

# Phase 3 — Inbound provider webhooks (ops)

These receive events **from** email/SMS providers into Gofox (Connected Apps / BYOK). They are **not** the outbound product, but ops teams need the URLs.

Base: `https://api.gofox.io/api/v1/public/messaging`

| Method | Path | Purpose |
|--------|------|---------|
| `POST` | `/webhooks/ses` | Amazon SES bounce/delivery/open/click |
| `POST` | `/webhooks/sendgrid/:organizationId` | SendGrid events |
| `POST` | `/webhooks/mailgun/:organizationId` | Mailgun events |
| `POST` | `/webhooks/mandrill/:organizationId` | Mandrill events |
| `POST` | `/webhooks/twilio/:organizationId` | Twilio SMS |
| `POST` | `/webhooks/exotel/:organizationId` | Exotel |
| `POST`/`GET` | `/webhooks/gupshup/:organizationId` | Gupshup |
| `GET` | `/track/open.gif`, `/track/open/:token` | Email open pixel |
| `GET` | `/track/click/:token` | Email click redirect |

Configure these in your ESP/SMS provider dashboard to match the Connected App for that organization. Signature validation is provider-specific (implemented server-side).

Env related to outbound: `REDIS_URL`, `ALLOW_HTTP_WEBHOOKS`, `WEBHOOK_DISPATCH_WORKER_CONCURRENCY`.

---

## Media placeholders

| File | Purpose |
|------|---------|
| `docs/assets/webhooks-settings.png` | Settings UI |
| `docs/assets/webhook-delivery-log.png` | Delivery log |
| `docs/assets/webhooks-setup.mp4` | End-to-end setup |

---

## License

Documentation © Gofox. Webhook usage subject to plan entitlements.
