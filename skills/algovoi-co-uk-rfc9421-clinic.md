---
name: algovoi-rfc9421-clinic
description: Verify or diagnose an RFC 9421 / RFC 9530 signed HTTP message for free against AlgoVoi's clinic, over REST or the anonymous MCP server, and fetch a known-good signed request to compare against.
api: openapi/algovoi-co-uk-clinic-openapi.yml
operations: [verify_verify_rfc9421_post, explain_verify_rfc9421_explain_post, sign_demo_verify_rfc9421_sign_post, health_health_get, agent_card__well_known_agent_card_json_get]
mcp_tools: [verify_rfc9421, explain_rfc9421, list_networks]
generated: '2026-09-19'
method: generated
source: Grounded in openapi/algovoi-co-uk-clinic-openapi.yml, mcp/algovoi-co-uk-clinic-tools-list.json (live tools/list 2026-09-19), https://agents.algovoi.co.uk/clinic/guide and the clinic agent card; every operationId and tool name exists verbatim.
---

# RFC 9421 signing clinic

Use when a signed agent-to-agent HTTP message will not verify, or when you must check that a message another
agent sent you is authentic and untampered. Free, anonymous, public data only. Base URL
`https://agents.algovoi.co.uk`. The only thing the provider asks is that you keep the citation it returns.

## Verify a message you received

1. Assemble the envelope: `method`, `authority` (host), `path`, `scheme` (default https), `headers` including
   `Signature-Input`, `Signature` and `Content-Digest`, and `body_b64` (base64 of the raw body).
2. Supply the key you **expect** — `public_key_hex` (32-byte Ed25519) or `did_key`. Never trust a key carried
   inside the envelope.
3. `verify_verify_rfc9421_post` — POST `/verify/rfc9421` with that JSON, or call MCP tool `verify_rfc9421` on
   `https://agents.algovoi.co.uk/mcp` (initialize + tools/call, no session, no credential). Read `{valid,
   errors, checked}`. Optional: `require_content_digest` (default true), `expected_tag`, `allowed_algorithms`
   (default `["ed25519"]`), `max_age_seconds`.

## Diagnose a message that fails

4. `explain_verify_rfc9421_explain_post` — POST `/verify/rfc9421/explain` (or MCP tool `explain_rfc9421`) with
   the same envelope. Read `signature_base` and `content_digest` and diff them against what your signer produced;
   `covered_components` shows which components the signature actually covers, `diagnosis` says what is wrong.
5. `sign_demo_verify_rfc9421_sign_post` — POST `/verify/rfc9421/sign` returns a known-good signed request under
   an ephemeral demo key; use it as a reference shape, never as a credential.

## Rules

- Read-only: nothing here mints a token, stores your message or changes state; `health_health_get` (GET `/health`)
  reports the verifier version (algovoi-rfc9421-verifier 0.4.4).
- Payment-proof verification is **not** on this host — that is the pay rail (skills/algovoi-co-uk-pay-per-call-verification.md).
- The A2A door (`POST /a2a`, message/send for 0.3.0 or SendMessage for 1.0.1) offers the same skills; discover
  it with `agent_card__well_known_agent_card_json_get`.
- Validation failures return FastAPI 422 `{detail:[...]}`; see errors/algovoi-co-uk-problem-types.yml.
