---
name: Book transfer between investors on AtomicNet
description: Move a quantity of an asset from one investor to another on the same book, against pre-approved escrow and beneficiary authorizations, and release the book transfer confirmation.
api: openapi/polysign-atomicnet-api-server-openapi.json
operations:
  - POST /v1/escrow_authorizations/request
  - PUT /v1/escrow_authorizations/
  - POST /v1/beneficiary_authorizations/request
  - PUT /v1/beneficiary_authorizations/
  - PUT /v1/book_transfers/
  - GET /v1/book_transfers/{book_transfer_id}
  - GET /v1/book_transfers/list/local
  - GET /v1/book_transfer_confirmations/{book_transfer_id}
  - PUT /v1/book_transfer_confirmations/
  - PUT /v1/book_transfers/submit
generated: '2026-08-02'
method: generated
source: openapi/polysign-atomicnet-api-server-openapi.json, openapi/polysign-merchant-gate-openapi.json
---

# Book transfer between investors on AtomicNet

A book transfer moves a quantity of one asset from a giving investor to a taking investor without an
order match. It uses the same authorization pair as an order. Authenticate first — see
`polysign-authenticate-atomicnet.md`.

## Steps

1. **Escrow the give side.** `POST /v1/escrow_authorizations/request` then
   `PUT /v1/escrow_authorizations/` to approve. Capture the `escrow_auth_id`.

2. **Authorize the take side.** `POST /v1/beneficiary_authorizations/request` then
   `PUT /v1/beneficiary_authorizations/` to approve. Capture the `beneficiary_auth_id`.

3. **Publish the transfer.** `PUT /v1/book_transfers/` with a `BookTransferRecord`
   (`asset_id`, `quantity`, `give_investor_id`, `take_investor_id` required) and the
   `escrow_auth_id` / `beneficiary_auth_id` on the enclosing `BookTransfer`.

4. **Read it back.** `GET /v1/book_transfers/{book_transfer_id}` and check `execution_status`.
   `GET /v1/book_transfers/list/local` lists only transfers this node is involved in;
   `GET /v1/book_transfers/list` lists all of them.

5. **Release the confirmation.** `GET /v1/book_transfer_confirmations/{book_transfer_id}` to read it,
   `PUT /v1/book_transfer_confirmations/` to release it on execution.
   `GET /v1/book_transfer_confirmations/list/local` scopes the list to this node.

## Merchant Gate variant

If you are integrating as a merchant rather than running a node, the Merchant Gate Node exposes the
same intent through three write operations against a node:

- `PUT /v1/book_transfers/submit` — submit a book transfer to an AtomicNet node
- `PUT /v1/orders/submit_order` — submit an order to an AtomicNet node
- `PUT /v1/assets/register` — register assets to an AtomicNet node

The Merchant Gate uses the same `POST /v1/auth/token` client-credentials flow and the same
`participant` scope, and ships the same four `/v1/system/*` health operations. It has no read
operations — fetch state from the AtomicNet API Server.

## Rules

- The book transfer confirmation carries `asset_id`, `asset_quantity`, `escrow_auth_id` and
  `beneficiary_auth_id` but not the investor ids — join back to the `BookTransferRecord` for those.
- No `Idempotency-Key`. Re-publishing the same `book_transfer_id` is the retry mechanism.
- No status-update operation exists for book transfers (unlike orders, escrow, beneficiary and
  settlements) — poll `GET /v1/book_transfers/{book_transfer_id}` and read `execution_status`.
