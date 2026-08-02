---
name: Authenticate against an AtomicNet node
description: Obtain an OAuth 2.0 client-credentials access token for an AtomicNet node, Merchant Gate or ABC Proxy Service, then confirm the node is healthy before doing any work.
api: openapi/polysign-atomicnet-api-server-openapi.json
operations:
  - POST /v1/auth/token
  - GET /v1/system/ping
  - GET /v1/system/deep_ping
  - GET /v1/system/version
  - GET /v1/system/circuit_breakers
generated: '2026-08-02'
method: generated
source: openapi/polysign-atomicnet-api-server-openapi.json
---

# Authenticate against an AtomicNet node

All three PolySign APIs — the AtomicNet API Server, the Merchant Gate Node and the ABC Proxy Service —
share the same token endpoint and the same system surface. Do this first for every flow.

**Base URL is not published.** The OpenAPI documents declare no `servers[]` block; AtomicNet nodes are
deployed by each participant. Use the node host your PolySign onboarding gave you and root every path
at `/v1`. Do not guess a host.

**No `operationId` exists** in any of these specs. Address operations by method + path, exactly as
written below.

## Steps

1. **Get an access token.** `POST /v1/auth/token`
   - Header `authorization: Basic <base64 of client_id:secret>` (required).
   - Header `accept: application/json`, header `cache-control: no-cache`.
   - Body, `application/x-www-form-urlencoded`: `grant_type=client_credentials&scope=participant`.
   - Both `grant_type` and `scope` are required. `participant` is the only scope the specs declare.
   - Success is `200` with `{access_token, token_type, expires_in, scope}` (`refresh_token` and
     `id_token` are optional and may be absent).
   - Failure is `401` with the OAuth 2.0 envelope `{error, error_description}` — **not** RFC 9457
     problem+json. Do not expect a `type`/`title`/`detail` body anywhere in this API.

2. **Cache the token for `expires_in` seconds.** Every other operation in all three documents is
   covered by the document-level requirement `security: [{OAuth2: [participant]}]`, so send
   `Authorization: Bearer <access_token>` on all of them.

3. **Confirm the node is up.** `GET /v1/system/ping` for liveness, then `GET /v1/system/deep_ping`
   for a dependency-aware check.

4. **Record what you talked to.** `GET /v1/system/version` returns the node build. The `info.version`
   of each spec is a git commit SHA, so pinning the node version is the only reliable way to know
   which contract revision you are on.

5. **Check for load shedding before a write burst.** `GET /v1/system/circuit_breakers` returns
   server-side circuit-breaker state. PolySign publishes no rate-limit headers or quota policy, so
   this is the only backpressure signal available — poll it rather than inferring throttling from
   failures.

## Rules

- There is no `Idempotency-Key` header anywhere in these APIs. Writes are `PUT` with a client-supplied
  record id in the body, which makes republishing the same record safe at the ledger level — but
  PolySign publishes no idempotency contract, so never assume a retried `POST` is deduplicated.
- 74 of the 78 operations document a `200` response only. Treat any non-2xx as undocumented, log the
  raw body, and do not pattern-match on an error schema that is not in the contract.
- Workflow failure is carried **in band**, not as HTTP status: read `execution_status`,
  `metadata.status` and `metadata.expired` on returned records, and poll the matching
  `*_status_updates` operation for transitions.
