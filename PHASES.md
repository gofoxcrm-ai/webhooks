# Outbound Webhooks — documentation & delivery phases

| Phase | Theme | Status |
|-------|--------|--------|
| **1** | Subscribe, sign, deliver, retry, UI | ✅ Implemented — [README.md](./README.md#phase-1--outbound-webhooks) |
| **2** | Event emit matrix (what actually fires) | ✅ Documented — [README.md](./README.md#phase-2--event-emit-matrix) |
| **3** | Inbound ESP/SMS provider webhooks (ops) | ✅ Paths live; optional for integrators — [README.md](./README.md#phase-3--inbound-provider-webhooks-ops) |

## Phase checklist

### Phase 1
- [x] Tenant CRUD + test + delivery logs
- [x] HMAC-SHA256 `X-Gofox-Signature`
- [x] Event allow-list (8 events)
- [x] BullMQ retries when Redis is configured
- [x] SSRF / private IP blocking; HTTPS in production
- [ ] Screenshots / setup video

### Phase 2
- [x] Document which domain actions emit which events
- [x] Correct payload shape (`occurredAt`, no delivery-id header)
- [ ] Sample payloads per event in docs/assets

### Phase 3
- [x] SES / SendGrid / Mailgun / Mandrill / Twilio / Exotel / Gupshup inbound paths
- [x] Email open/click tracking URLs
- [ ] Customer-facing setup guides per ESP (Connected Apps)

## Source of truth

| Concern | Path |
|---------|------|
| Outbound emit / sign | `gofox-server/src/lib/outbound-webhooks.ts` |
| Tenant routes | `gofox-server/src/modules/webhooks/` |
| Queue / retries | `gofox-server/src/queues/outbound-webhook.queue.ts` |
| Inbound ESP | `gofox-server/src/modules/messaging-webhooks/` |
