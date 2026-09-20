---
name: algovoi-hosted-checkout
description: As an authenticated AlgoVoi tenant, create a hosted USDC checkout link, poll it until paid, cancel or abandon it while still active, and verify the signed payment.confirmed webhook.
api: openapi/algovoi-co-uk-gateway-openapi.yml
operations: [create_dynamic_payment_link_v1_payment_links_post, checkout_status_checkout__token__status_get, verify_checkout_checkout__token__verify_post, cancel_checkout_checkout__token__cancel_post, abandon_checkout_checkout__token__abandon_post, health_health_get]
mcp_tools: [create_payment_link, verify_payment, verify_webhook, list_networks]
generated: '2026-09-19'
method: generated
source: Grounded in openapi/algovoi-co-uk-gateway-openapi.yml, https://docs.algovoi.co.uk/quickstart, /api-reference/introduction, /api-reference/errors, /integrations/outbound-webhooks and /concepts/kyc-and-mainnet; every operationId exists verbatim in the spec.
---

# Hosted checkout on the AlgoVoi gateway

Use only when the operator has given you a tenant key. Base URL `https://api.algovoi.co.uk`
(`cloud.algovoi.co.uk` is a non-canonical proxy). Every call sends `Authorization: Bearer algv_…` and
`X-Tenant-Id: <uuid>`, `Content-Type: application/json`. Test-mode keys reach testnet only; mainnet needs KYC
(`422 kyb_required_for_mainnet` otherwise). Stay under 300 requests/minute per tenant; on 429 wait `Retry-After`.

## Create the link

1. `health_health_get` — GET `/health` → `{"status":"ok"}` before a batch.
2. `create_dynamic_payment_link_v1_payment_links_post` — POST `/v1/payment-links` with
   `{"amount": 9.99, "currency": "USD", "label": "Order #1234", "preferred_network": "algorand_testnet"}`
   and an `Idempotency-Key` header (a UUID) so a retry within 24 h returns the same link instead of a duplicate.
   Keep `checkout_url`, `token`, `expires_at` and the merchant-only `cancel_secret` — never expose the secret.
   (MCP: `create_payment_link`.)

## Wait for payment

3. `checkout_status_checkout__token__status_get` — GET `/checkout/{token}/status` → `active | paid | expired |
   cancelled` (+ `redirect_url`). Poll no faster than the 60/min per-checkout limit; it "does not mutate state".
   (MCP: `verify_payment`.)
4. If the payer hands you a `tx_id`, `verify_checkout_checkout__token__verify_post` — POST
   `/checkout/{token}/verify` — asks the facilitator to confirm the exact amount to the receiver on the right chain.
   `409 payment_link_already_paid` means success already happened; `422 payment_not_verified` or
   `payment_link_expired` are terminal for that link.

## Reverse while you still can

5. Before payment: `cancel_checkout_checkout__token__cancel_post` — POST `/checkout/{token}/cancel` with the
   `cancel_secret` (merchant) or `abandon_checkout_checkout__token__abandon_post` — POST `/checkout/{token}/abandon`
   (customer-side, no secret) which emits `payment.cancelled` immediately. After a verified payment the on-chain
   transfer is final and no API reversal exists; a refund is an off-API event.

## Confirm out of band

6. Trust the signed webhook over polling: `payment.confirmed` arrives at the generic destination with
   `X-AlgoVoi-Signature: t=…,v1=…`; verify `HMAC-SHA256(algvw_secret, "{t}.{raw_body}")` within ±300 s, dedupe on
   `id` (at-least-once delivery), read `data.tx_id` and `data.amount_microunits` (a string). (MCP: `verify_webhook`.)

## Rules

- Conventions and idempotency scope: conventions/algovoi-co-uk-conventions.yml. Error envelope
  `{error, message, request_id}`: errors/algovoi-co-uk-problem-types.yml. Rate limits: rate-limits/.
- Never create mainnet links without the operator's explicit amount and network.
