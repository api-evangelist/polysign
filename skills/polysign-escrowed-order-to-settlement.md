---
name: Escrowed order to settlement on AtomicNet
description: Register an asset, obtain escrow and beneficiary authorizations, publish an order, follow it to a settlement, and release the settlement confirmation on the AtomicNet API Server.
api: openapi/polysign-atomicnet-api-server-openapi.json
operations:
  - PUT /v1/assets/
  - POST /v1/escrow_authorizations/request
  - PUT /v1/escrow_authorizations/
  - POST /v1/beneficiary_authorizations/request
  - PUT /v1/beneficiary_authorizations/
  - GET /v1/investors/real_investor_id/{real_investor_id}
  - PUT /v1/orders/
  - GET /v1/orders/{order_id}
  - GET /v1/order_status_updates/{order_id}
  - GET /v1/settlements/order/{order_id}
  - GET /v1/settlement_confirmations/{give_order_id}/{take_order_id}
  - PUT /v1/settlement_confirmations/
  - GET /v1/settlement_status_updates/{settlement_id}
generated: '2026-08-02'
method: generated
source: openapi/polysign-atomicnet-api-server-openapi.json
---

# Escrowed order to settlement on AtomicNet

This is the marquee AtomicNet flow: assets are escrowed, a beneficiary is pre-authorized, an order is
published against those authorizations, the network matches it into a settlement, and the settlement
confirmation is released on execution. Authenticate first — see
`polysign-authenticate-atomicnet.md`.

## Steps

1. **Make sure the asset is registered.** `GET /v1/assets/{asset_id}`; if it is absent,
   `PUT /v1/assets/` with an `AssetRecord` (`asset_id`, `book_operator_id`, `exchange_id`,
   `precision`, `status` are all required). Register many at once with `PUT /v1/assets/list`. If your
   own system uses a different identifier, map it with `PUT /v1/assets/update_asset_mapping`
   (`partner_asset_id`).

2. **Pseudonymize the investor.** `GET /v1/investors/real_investor_id/{real_investor_id}` mints a
   one-time investor ID. Use the one-time ID on the network; only entitled nodes can resolve it back
   with `GET /v1/investors/one_time_investor_id/{one_time_investor_id}`. Never put a real investor
   identifier on a published record when a one-time ID is available.

3. **Request an escrow authorization.** `POST /v1/escrow_authorizations/request` with an
   `EscrowAuthRecord` (`escrow_auth_id`, `investor_id`, `asset_id`, `asset_quantity`, `expiration`
   required). The counterparty node approves with `PUT /v1/escrow_authorizations/`.
   Read it back with `GET /v1/escrow_authorizations/{escrow_auth_id}` and check `execution_status`
   and `remaining_quantity`. `GET /v1/escrow_authorizations/list/local` scopes the list to
   authorizations this node is involved in; `?role=` narrows by caller role.

4. **Request a beneficiary authorization.** `POST /v1/beneficiary_authorizations/request` with a
   `BeneficiaryAuthRecord` (`beneficiary_auth_id`, `investor_id`, `asset_id`, `expiration`), approved
   with `PUT /v1/beneficiary_authorizations/`. This is the take side: it names who may receive the
   asset.

5. **Publish the order.** `PUT /v1/orders/` with an `OrderRecord` (`investor_id`, `give_asset_id`,
   `give_quantity`, `take_asset_id`, `take_quantity`, `expiration` required) plus the
   `escrow_auth_id` and `beneficiary_auth_id` from steps 3 and 4 on the enclosing `Order`. The
   authorizations must already be approved — the order references them, it does not create them.

6. **Follow the order.** `GET /v1/orders/{order_id}` for `execution_status` and `give_remaining`;
   `GET /v1/order_status_updates/{order_id}` for the dated transition history. There are no webhooks
   and no AsyncAPI — polling the status-update operations is the only event mechanism PolySign
   publishes.

7. **Find the settlement.** `GET /v1/settlements/order/{order_id}` returns every settlement the order
   participated in. A `SettlementRecord` pairs a `left_order_id` and a `right_order_id` with the
   give quantities and commissions on each side. `GET /v1/settlements/{settlement_id}` fetches one
   directly; `abc_memorial_digest` and `abc_memorialization_block` show where it was memorialized on
   the ABC ledger.

8. **Release the settlement confirmation.** `GET /v1/settlement_confirmations/{give_order_id}/{take_order_id}`
   to read the confirmation, then `PUT /v1/settlement_confirmations/` to release it on execution.
   Track it with `GET /v1/settlement_confirmation_status_updates/{give_order_id}/{take_order_id}` and
   the settlement with `GET /v1/settlement_status_updates/{settlement_id}`.

9. **Publish the settlement if you are the memorializing node.** `PUT /v1/settlements/` publishes a
   settlement to AtomicNet *after* it has been memorialized — memorialize the digest first via the
   ABC Proxy Service (`polysign-abc-memorialize-and-verify.md`).

## Rules

- Every record you publish is `{record, metadata, signature, schema_version}`. The `signature` is an
  `AbcSignature` over the **canonical** form of `record` — produce that form with
  `GET /v1/utils/canonicalize`, never by serializing the object yourself.
- Need a fresh identifier? `GET /v1/utils/atomicnet_uid`.
- `metadata.request_id` is the correlation field; there is no request-id HTTP header.
- List operations accept `status` (and sometimes `role`) filters only. There is no pagination — no
  `limit`, `offset`, `cursor` or `page` — so a large list is returned whole.
- Order, escrow and beneficiary records all carry an `expiration`; check `metadata.expired` before
  acting on a record you fetched earlier.
