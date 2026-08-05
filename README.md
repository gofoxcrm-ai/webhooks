# Gofox Outbound Webhooks

Push **real-time CRM events** from Gofox to your HTTPS endpoint. Pattern matches EngageBay-style [Webhooks](https://www.engagebay.com/api): configure a URL, verify signatures, process JSON.

> Sibling products: [REST API](https://github.com/gofoxcrm-ai/restapi) · [Tracking Code API](https://github.com/gofoxcrm-ai/trackingcodeapi) · [SSO](https://github.com/gofoxcrm-ai/sso)

---

## Status

| Area | Status |
|------|--------|
| Tenant webhook CRUD + test + delivery logs | ✅ Live |
| Signed deliveries (`X-Gofox-Signature`) | ✅ Live |
| Retries (BullMQ backoff) | ✅ Live |
| Screenshots / setup video | 🚧 Placeholders below |

> **Note:** Inbound provider webhooks (Razorpay, WhatsApp, telephony, etc.) are separate. This repo covers **outbound CRM → your app**.

---

## Setup

1. Plan feature **`webhooks`** / **`api_access`** required.
2. Gofox UI: **Account Settings → Webhooks**
3. Add HTTPS URL + select events
4. Copy the **signing secret**
5. Use **Test** to send a sample payload

<!-- SCREENSHOT: docs/assets/webhooks-settings.png
     Placeholder — Webhooks settings page with URL, events, Test button.
-->

![Webhooks settings (placeholder)](docs/assets/webhooks-settings.png)

<!-- VIDEO: docs/assets/webhooks-setup.mp4
     Placeholder — create endpoint, receive Test hit on webhook.site, verify signature.
-->

[Setup video (placeholder)](docs/assets/webhooks-setup.mp4)

---

## Events (working)

| Event | When |
|-------|------|
| `lead.created` | Lead created / ingested |
| `contact.created` | Contact created |
| `contact.updated` | Contact updated |
| `deal.created` | Deal created |
| `deal.updated` | Deal updated |
| `ticket.created` | Support ticket created |
| `invoice.paid` | Invoice marked paid |
| `form.submitted` | Public form submission |

Additional events (`company.*`, `task.*`, `deal.stage_changed`, …) can be added in `gofox-server/src/lib/outbound-webhooks.ts` — document them here when shipped.

---

## Delivery payload

```http
POST https://your-app.example.com/hooks/gofox
Content-Type: application/json
X-Gofox-Signature: <hex hmac-sha256>
X-Gofox-Event: contact.created
X-Gofox-Delivery-Id: del_…
```

```json
{
  "id": "evt_…",
  "event": "contact.created",
  "createdAt": "2026-07-20T12:00:00.000Z",
  "organizationId": "org_…",
  "data": {
    "id": "contact_…",
    "email": "ada@example.com",
    "firstName": "Ada"
  }
}
```

---

## Verify signatures

Compute HMAC-SHA256 of the **raw request body** with your webhook secret; compare to `X-Gofox-Signature` (hex).

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

---

## Retries & SSRF

- Non-2xx responses are retried with exponential backoff (BullMQ).
- Inspect delivery history in the Webhooks UI.
- Target URL must be public HTTPS (http allowed in local/dev). Private / link-local IPs are rejected.

---

## Tenant management API

Authenticated with session JWT (not public API key by default):

```
GET|POST   /api/v1/tenant/webhooks
PATCH|DELETE /api/v1/tenant/webhooks/:id
POST       /api/v1/tenant/webhooks/:id/test
GET        /api/v1/tenant/webhooks/:id/deliveries
```

Implementation: `gofox-server/src/modules/webhooks/`.

---

## Adding events

1. Emit from the domain service via `emitOutboundWebhook(...)` (or shared helper)
2. Register the event name in the allow-list / UI enum
3. Add a row to the Events table in this README
4. Include a sample `data` shape in docs when payloads diverge

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
