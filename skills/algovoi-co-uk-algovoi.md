---
name: algovoi
description: Use when building payment infrastructure for AI agents, e-commerce platforms, or API monetization. Reach for this skill when you need to integrate stablecoin payments across multiple blockchains, implement compliance-aware payment flows, verify on-chain settlements, handle webhooks, or configure payment protocols (x402, MPP, AP2, A2A).
metadata:
    mintlify-proj: algovoi
    version: "1.0"
---

# AlgoVoi Skill

## Product summary

AlgoVoi is a compliance-aware payment gateway for stablecoin transactions across seven blockchains (Algorand, VOI, Hedera, Stellar, Base, Solana, Tempo). It handles settlement verification, sanctions screening, KYB gating, and audit-chain recording on every payment. Agents use AlgoVoi to monetize APIs via x402 (HTTP 402 Payment Required), gate MCP tools via MPP, authorize agent commerce via AP2/A2A, or accept payments in e-commerce and chat bots.

**Key entry points:**
- REST API: `https://api.algovoi.co.uk` (authenticated with Bearer key + X-Tenant-Id header)
- Dashboard: `https://dash.algovoi.co.uk` (tenant management, API keys, webhook config, payment history)
- Agent Trust Bench: `https://agent-trust-bench.algovoi.co.uk` (test agent payment behavior before deployment)

**Primary docs:** https://docs.algovoi.co.uk

## When to use

Reach for this skill when:
- **Building agent payment flows:** Agents need to pay for API calls, MCP tools, or other services with cryptographic proof of settlement
- **Monetizing APIs:** Gate endpoints with x402 and charge per request across multiple chains
- **Implementing compliance:** Need sanctions screening, KYB verification, audit trails, or regulatory attestation
- **Multi-chain settlement:** Accept payments on any of seven chains from a single integration
- **Webhook handling:** Receive and verify signed payment confirmation events
- **Testing agent behavior:** Validate that agents refuse adversarial payment scenarios before production
- **E-commerce/chat integration:** Add stablecoin payments to Shopify, Discord, Telegram, or other platforms

## Quick reference

### API authentication headers
```
Authorization: Bearer <API_KEY>
X-Tenant-Id: <TENANT_ID>
```

### API key prefixes and scopes
| Prefix | Purpose | Scope |
|--------|---------|-------|
| `algv_…` | Tenant API key | Normal payment operations |
| `algvc_…` | Admin/control-plane | Tenant management (operators only) |
| `algvw_…` | Webhook signing secret | Per-destination, for verifying inbound webhooks |

### Payment protocols at a glance
| Protocol | Use case | Initiator |
|----------|----------|-----------|
| **x402** | Per-request API monetization | Client sends `X-Payment` header with tx_id |
| **MPP** | MCP tool pricing | Agent negotiates subscription via payment context |
| **AP2** | Mandate-based agent commerce | Agent signs CartMandate, merchant renders Payment Request |
| **A2A** | Agent-to-agent settlement | Two agents negotiate chain and settle directly |

### Webhook signature verification
```
Header: X-AlgoVoi-Signature: t={unix_ts},v1={hex}
Verify: HMAC-SHA256(secret, "{ts}.{raw_body}") == v1
Tolerance: ±300 seconds
```

### Common API endpoints
| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/v1/payment-links` | POST | Create hosted checkout link |
| `/v1/x402/requirements` | GET | Get 402 Payment Required challenge |
| `/v1/x402/verify` | POST | Verify x402 payment and get receipt |
| `/v1/protected/{resource_id}` | GET | Access x402-gated resource |
| `/compliance/screen` | POST | Pre-screen payer before payment |
| `/compliance/attestation` | GET | Retrieve compliance posture (public) |

### Compliance screening verdicts
| Verdict | Meaning | Action |
|---------|---------|--------|
| **ALLOW** | Payer passed all checks | Payment proceeds |
| **REFER** | Matched watchlist/risk threshold | Manual review required (SAR obligation) |
| **DENY** | Matched sanctions list | Block payment (SAMLA tipping-off rules apply) |

### Chain identifiers (CAIP-2 format)
```
algorand:mainnet, algorand:testnet
voi:mainnet
hedera:mainnet
stellar:mainnet
eip155:8453 (Base mainnet)
solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp (Solana mainnet)
tempo:mainnet
```

## Decision guidance

### When to use x402 vs MPP vs AP2 vs A2A

| Scenario | Protocol | Why |
|----------|----------|-----|
| Charge per HTTP request (LLM call, image gen, data lookup) | **x402** | Stateless, per-request, no pre-authorization needed |
| Price an MCP tool or recurring service | **MPP** | Built for subscription/mandate model with charge/recurring intent |
| Agent buys from merchant with cart/mandate | **AP2** | Binds payment to CartMandate, W3C Payment Request compatible |
| Agent-to-agent direct settlement (no facilitator) | **A2A** | Mutual Falcon-1024 auth, no per-tx fee, chain-agnostic |
| Simple checkout link for humans | **Hosted checkout** | Easiest path; no code needed, QR code + memo |

### When to use testnet vs mainnet

| Condition | Network | Notes |
|-----------|---------|-------|
| Building/testing, no KYC | **Testnet** | 60-day free trial, all 7 chains available |
| Ready for real money, KYC passed | **Mainnet** | $1,000 free allowance across all 7 chains after approval |
| Trying mainnet without KYC | **Error** | Returns `kyb_required_for_mainnet`; complete identity verification first |

### When to test with Agent Trust Bench

| Situation | Action |
|-----------|--------|
| Before deploying agent to production | Run full 187-scenario suite; pass if ≥177 correct decisions |
| Agent needs reputation discount | Score above threshold → fetch Falcon-1024 ATB Pass Certificate |
| Monitoring agent population behavior | Check live stats dashboard for ecosystem-wide payment patterns |

## Workflow

### 1. Set up a tenant and get API keys

1. Go to `https://dash.algovoi.co.uk/signup` and create an account
2. Verify email (testnet access granted immediately)
3. Go to **Settings → API keys** and create a new key
   - Choose environment: test or live
   - Choose scope: payment-write, read-only, etc.
   - Copy the key (shown once; store securely)
4. Note your Tenant ID from **Settings** (a UUID)
5. For webhooks, go to **Settings → Notifications** and add a destination (generic HTTPS, Slack, Discord, Teams, Telegram)
   - Copy the per-destination webhook signing secret

### 2. Add a payout address

1. Go to **Settings → Networks**
2. Pick a chain (Algorand testnet is fastest to demo)
3. Paste a wallet address you control
4. Repeat for other chains if needed

### 3. Create and verify a payment (hosted checkout path)

```bash
# Create a checkout link
curl -X POST https://api.algovoi.co.uk/v1/payment-links \
  -H "Authorization: Bearer $ALGOVOI_API_KEY" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 1.00,
    "currency": "USD",
    "label": "Test payment",
    "preferred_network": "algorand_testnet"
  }'
```

4. Open the returned `checkout_url` in a browser
5. Send the requested amount from any testnet wallet (include the memo shown)
6. Page auto-detects payment and flips to "Paid"
7. Receive webhook event at your configured destination with signed `X-AlgoVoi-Signature` header

### 4. Implement x402 for API monetization

1. Define a protected resource in **Resources → New**
   - Set `resource_id`, `price_microalgos`, `payment_network`, `asset_id`
2. Client calls your endpoint without payment → you return `HTTP 402 Payment Required` with `payment_requirements` body
3. Client pays on-chain and retries with `X-Payment` header containing tx_id
4. Call `POST /v1/x402/verify` with the tx_id to confirm settlement
5. Return the resource + signed receipt to client
6. Receive `payment.confirmed` webhook event

### 5. Verify webhook signatures

```python
import hmac
import hashlib

def verify_webhook(header, secret, raw_body):
    # Parse header: "t=1234567890,v1=abc123def456"
    parts = dict(p.split('=') for p in header.split(','))
    ts = int(parts['t'])
    sig = parts['v1']
    
    # Check timestamp (±300 seconds)
    if abs(time.time() - ts) > 300:
        return False
    
    # Compute HMAC-SHA256
    expected = hmac.new(
        secret.encode(),
        f"{ts}.{raw_body}".encode(),
        hashlib.sha256
    ).hexdigest()
    
    # Constant-time comparison
    return hmac.compare_digest(sig, expected)
```

### 6. Screen a payer before payment (compliance)

```bash
curl -X POST https://api.algovoi.co.uk/compliance/screen \
  -H "Authorization: Bearer $ALGOVOI_API_KEY" \
  -H "X-Tenant-Id: $TENANT_ID" \
  -H "Content-Type: application/json" \
  -d '{
    "payer_address": "GHSRL2SAY247...MWI",
    "network": "algorand:mainnet"
  }'
```

Response: `{ "screen_result": "ALLOW" | "REFER" | "DENY" }`

### 7. Test agent payment behavior before production

1. Go to `https://agent-trust-bench.algovoi.co.uk`
2. Run your agent against the full 187-scenario suite
3. Check results: pass if ≥177 correct decisions (refuse all adversarial, pay only honest baselines)
4. If passing, fetch ATB Pass Certificate and present via `X-ATB-Credential` header for 20% discount

## Common gotchas

- **Webhook signature verification is mandatory.** Don't skip the HMAC check; always verify `X-AlgoVoi-Signature` before processing events. Timestamp tolerance is ±300 seconds; reject stale signatures.

- **Mainnet requires KYC.** Testnet is open to everyone, but mainnet blocks all payments until `kyb_status = approved`. Individuals auto-approve in minutes; companies require MLRO review (1–2 business days).

- **Self-payment is blocked.** A tenant cannot pay themselves (payer address == payout address). This prevents burning the $1,000 allowance for ~$1 of gas.

- **API keys are returned once.** If you lose a key, rotate it in the dashboard. Both old and new keys are valid for 30 days during the overlap window.

- **Webhook secrets rotate independently.** Rotating a webhook secret invalidates the old one immediately, but AlgoVoi signs with the new secret and accepts the old one for 30 days. Update your verifier during the overlap.

- **Idempotency keys prevent duplicate checkouts.** Include `Idempotency-Key` header on `POST /v1/payment-links` to replay safely within 24 hours.

- **Rate limits are per-tenant.** Default is 300 req/min with 60-req burst. Returns `429 Too Many Requests` with `Retry-After` header. Include `X-Request-Id` in support requests.

- **Compliance verdicts are categorical.** ALLOW, REFER, DENY are the only valid values. Probability scores or risk tiers are not acceptable in canonical records; regulators need byte-distinct verdicts.

- **Testnet and live keys cannot switch modes.** Create a new key in the other mode if you need to switch environments.

- **Webhook delivery retries with backoff.** Events retry with 30s → 32h escalating backoff. Return 200 for events you intentionally ignore; 4xx will trigger retry storms.

- **Cross-chain settlement requires xChain enabled.** A customer on Base can pay a checkout on Algorand only if you've configured payout addresses on both chains and the gateway has xChain enabled for your tenant.

## Verification checklist

Before submitting work:

- [ ] API key and Tenant ID are correct and match the environment (test vs live)
- [ ] Webhook signature verification is implemented and tested (HMAC-SHA256, timestamp check)
- [ ] Payout addresses are configured for all chains you plan to accept
- [ ] KYC is complete if using mainnet (check `kyb_status = approved` in dashboard)
- [ ] Compliance screening is called before payment if required by your use case
- [ ] Error handling covers all documented error codes (401, 403, 422, 429, etc.)
- [ ] Idempotency keys are included on POST requests that create resources
- [ ] Webhook destination is configured and receiving signed events
- [ ] Agent has passed Agent Trust Bench (≥177/187) before production deployment
- [ ] Settlement verification confirms on-chain finality (use settlement-verify if needed)
- [ ] Audit logs are reviewed for compliance (check `/compliance/attestation` endpoint)

## Resources

**Comprehensive page listing:** https://docs.algovoi.co.uk/llms.txt

**Critical documentation pages:**
1. [Quickstart](https://docs.algovoi.co.uk/quickstart) — Five-minute walkthrough of your first payment
2. [API Reference](https://docs.algovoi.co.uk/api-reference/introduction) — Base URL, authentication, versioning, rate limits
3. [x402 Protocol](https://docs.algovoi.co.uk/protocols/x402) — Per-request API monetization and multi-chain support
4. [Compliance Overview](https://docs.algovoi.co.uk/compliance) — Screening, KYB, audit chain, regulatory frameworks
5. [Outbound Webhooks](https://docs.algovoi.co.uk/integrations/outbound-webhooks) — Webhook payload schema and HMAC verification
6. [Agent Trust Bench](https://docs.algovoi.co.uk/agent-trust-bench) — Testing agents before production

---

> For additional documentation and navigation, see: https://docs.algovoi.co.uk/llms.txt