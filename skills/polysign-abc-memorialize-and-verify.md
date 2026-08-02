---
name: Memorialize and verify a digest on the ABC ledger
description: Register an ABC account for an AtomicNet participant, sign and verify a digest, memorialize it to the ABC ledger and prove the memorialization — including the testnet path for development.
api: openapi/polysign-abc-proxy-service-openapi.json
operations:
  - PUT /v1/abc_accounts/
  - GET /v1/abc_accounts/{atomicnet_id}
  - GET /v1/abc_accounts/info/{abc_account_id}
  - PUT /v1/abc_signing/
  - GET /v1/abc_signing/
  - PUT /v1/abc_memorials/
  - GET /v1/abc_memorials/{digest}
  - PUT /v1/abc_testnet/
  - GET /v1/abc_testnet/faucet/{abc_account_id}
generated: '2026-08-02'
method: generated
source: openapi/polysign-abc-proxy-service-openapi.json
---

# Memorialize and verify a digest on the ABC ledger

The ABC Proxy Service is how AtomicNet records get anchored: a record is canonicalized, signed with an
ABC account, and its digest is memorialized on the ABC ledger. Settlements and settlement
confirmations carry `abc_memorial_digest` and `abc_memorialization_block` pointing back at that
anchor. Authenticate first — see `polysign-authenticate-atomicnet.md`; the ABC Proxy Service uses the
same `POST /v1/auth/token` client-credentials flow and the same `participant` scope.

## Steps

1. **Register the ABC account.** `PUT /v1/abc_accounts/` registers an ABC account. Look it up by the
   participant with `GET /v1/abc_accounts/{atomicnet_id}`, or by the account itself with
   `GET /v1/abc_accounts/info/{abc_account_id}`.

2. **Canonicalize the record first.** On the AtomicNet API Server, `GET /v1/utils/canonicalize`
   returns the canonical form of an AtomicNet object record. Hash that form — never your own
   serialization — to get the digest you sign and memorialize.

3. **Sign the digest.** `PUT /v1/abc_signing/` signs a digest using the ABC account bound to the given
   AtomicNet ID. The `AbcSignature` shape is `{abc_account_id, digest, digest_type, signature}`, all
   four required.

4. **Verify a signature.** `GET /v1/abc_signing/` verifies a signature against the digest and account.
   Do this on anything you received from a counterparty node before acting on it.

5. **Memorialize.** `PUT /v1/abc_memorials/` memorializes the digest using the ABC account for the
   given AtomicNet ID. This is what makes the record provable on the ledger.

6. **Prove it.** `GET /v1/abc_memorials/{digest}` verifies whether a digest has been memorialized to
   ABC. Use this before publishing a settlement — `PUT /v1/settlements/` on the AtomicNet API Server
   is documented as publishing a settlement *after* it has been memorialized.

## Development on the ABC testnet

- `PUT /v1/abc_testnet/` creates an account on the ABC testnet. This is the only operation in any of
  the three specs that documents a `500` response ("failed to create"), with no body schema — handle
  it as an opaque failure and retry.
- `GET /v1/abc_testnet/faucet/{abc_account_id}` funds a testnet account.
- PolySign publishes **no** test client credentials, no test/live key prefixes, and no magic
  identifiers. Get sandbox credentials from PolySign (hello@polysign.io); do not invent them.

## Rules

- Verify before you trust: signature verification (`GET /v1/abc_signing/`) and memorial verification
  (`GET /v1/abc_memorials/{digest}`) are cheap reads and are the whole point of the service.
- The `digest_type` field is required on every signature — carry the counterparty's value through
  rather than defaulting it.
- The ABC Proxy Service ships the same `/v1/system/ping`, `/v1/system/deep_ping`, `/v1/system/version`
  and `/v1/system/circuit_breakers` health surface as the other two APIs.
