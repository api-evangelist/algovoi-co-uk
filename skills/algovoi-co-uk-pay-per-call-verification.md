---
name: algovoi-pay-per-call-verification
description: Buy one signed verification from the tenant-free AlgoVoi Pay rail — negotiate a settlement lane, take the 402 challenge, pay exactly once on-chain, resubmit with X-PAYMENT, and verify the Ed25519 receipt offline.
api: openapi/algovoi-co-uk-pay-openapi.yml
operations: [payable_index_pay_v1_index_get, negotiate_capabilities_pay_v1_negotiate_get, negotiate_pay_v1_negotiate_post, probe_receipt_pay_v1_verify_receipt_get, pay_verify_receipt_pay_v1_verify_receipt_post, pay_verify_rfc9421_pay_v1_verify_rfc9421_post, pay_screen_url_pay_v1_screen_url_post, verify_receipt_post_v1_receipt_verify_post]
generated: '2026-09-19'
method: generated
source: Grounded in openapi/algovoi-co-uk-pay-openapi.yml, https://pay.algovoi.co.uk/pay/v1/index, https://docs.algovoi.co.uk/algovoi-pay and the live 402 observed 2026-09-19; every operationId exists verbatim in the spec.
---

# Pay-per-call verification on AlgoVoi Pay

Use when an operator asks you to verify an x402/AP2 payment receipt, an RFC 9421 signature, or to screen a URL,
and has authorised spending **0.01 USDC per call** from a wallet you control on one of the twelve mainnet lanes.
No account, no API key: **payment is the credential**. Base URL `https://pay.algovoi.co.uk`.

## 0. Prefer the free tier when a receipt is not needed

`verify_receipt_post_v1_receipt_verify_post` — POST `/v1/receipt/verify` `{"jws": "..."}` is unsigned and free
(120/min per the index). It answers the same question without a countersigned receipt. Only pay when the operator
wants an offline-verifiable AlgoVoi receipt binding the request and response.

## 1. Read the contract, then negotiate a lane

1. `payable_index_pay_v1_index_get` — GET `/pay/v1/index`. Read `services[]` (url, price_microusdc, status),
   `networks[]` (CAIP-2 id, asset, decimals, binding) and `challenge_ttl_secs` (1800). Skip any service whose
   `status` is `coming_soon` (compliance-receipt returned 503 on 2026-09-19).
2. `negotiate_pay_v1_negotiate_post` — POST `/pay/v1/negotiate` `{"chains": ["eip155:8453", "solana", ...],
   "protocols": ["x402"]}` with only the chains you can actually pay from. Read `common_lanes` and `recommended`.
   Read-only: "mints no token, changes no state, settles nothing". `negotiate_capabilities_pay_v1_negotiate_get`
   returns the full lane set if you need it.

## 2. Take the challenge

3. `probe_receipt_pay_v1_verify_receipt_get` — GET `/pay/v1/verify/receipt` returns the strict x402 v2 **402**
   over an example body (a free rehearsal). For the real call, POST your body **without** `X-PAYMENT` first:
   `pay_verify_receipt_pay_v1_verify_receipt_post` with `{"jws": "<receipt>", "receipt_required": true}` → 402.
4. In `accepts[]` pick the entry whose `network` is your negotiated lane. Keep `payTo`, `amount` (10000 =
   0.01 USDC at 6 decimals; Stellar uses 7), `asset`, `maxTimeoutSeconds` and **`extra.payment_reference`** —
   the challenge token. It expires in 1800 s; do not reuse one across requests (`request_hash` differs).

## 3. Pay exactly once, welded to the challenge

5. Mined-tx lanes (Algorand, Voi, Stellar, Hedera, Solana, Tempo): the weld is **mandatory and fail-closed**.
   Put the raw `payment_reference` string in the tx note / MEMO_TEXT / memo (do **not** pre-base64 it), or for
   Solana include `extra.svm_reference_pubkey` as a read-only account; Tempo uses `transferWithMemo`.
6. EVM EIP-3009 lanes (Base, Monad, Polygon, Arbitrum, OP, Arc): sign a standard `TransferWithAuthorization`
   with `to == payTo`, `value == amount` exactly, under the `eip712_domain` in the accept; the nonce weld
   (`keccak256(payment_reference)`) is optional hardening.
7. Pay the exact amount only. Each `(network, tx_id)` buys **exactly one** call.

## 4. Present the payment and read the receipt

8. Re-POST the **identical** body with `X-PAYMENT: base64(JSON)` — `{tx_id, payment_reference}` on mined-tx
   lanes, `{authorization, signature[, challenge_token]}` on EVM lanes. 200 returns the result plus an Ed25519
   JWS payable receipt (kind `algovoi/payable-svc-receipt/1`) binding `settled_payment_ref -> request_hash ->
   response_hash`. 422 is a body validation error — fix and resend the same payment; an identical re-POST
   replays the stored response, a **different** body on the same tx is 409, never a leaked result.
9. Verify the receipt offline against `https://pay.algovoi.co.uk/.well-known/jwks.json` (kid
   `0226212c7d1243326cab5ae646e562a1`, EdDSA) or resolve `did:web:pay.algovoi.co.uk`; no callback needed.

## Rules

- Reversibility: on-chain payment is final; there is no refund operation. Confirm the spend with the operator
  before step 5 and never pay more than one challenge per requested verification.
- The same flow applies to `pay_verify_rfc9421_pay_v1_verify_rfc9421_post` (`/pay/v1/verify/rfc9421`) and
  `pay_screen_url_pay_v1_screen_url_post` (`/pay/v1/screen/url`).
- Errors follow errors/algovoi-co-uk-problem-types.yml; runtime conventions in conventions/algovoi-co-uk-conventions.yml.
