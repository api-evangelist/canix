---
name: Create and fulfill a Canix sales order
description: Create a wholesale sales order for a cannabis customer in Canix, verify what
  landed on it, advance it through its status workflow, and reconcile the payments against it.
api: openapi/canix-openapi-original.yml
generated: '2026-08-09'
method: generated
source: openapi/canix-openapi-original.yml
operations:
  - GetFacilities
  - GetCustomers
  - GetCustomerById
  - GetItems
  - CreateSalesOrder
  - GetSalesOrderById
  - GetSalesOrderContentsById
  - UpdateSalesOrderStatus
  - GetSalesOrderPayments
  - GetSubmissionById
---

# Create and fulfill a Canix sales order

Base URL `https://api.canix.com/api/v1`. Every request carries `X-API-KEY: <key>`.
The key is company-scoped — it can see every facility in the company, so the facility must be
chosen explicitly, never assumed.

## Before you start

- **Writes are not idempotent.** Canix publishes no `Idempotency-Key` contract. If
  `CreateSalesOrder` times out, do **not** retry it. Re-list with `GetSalesOrders` filtered on
  the customer and a recent `created_at` and check whether the order already exists.
- **403 is ambiguous.** An anonymous or unauthorized call returns
  `403 {"message":"Access Denied"}`, not 401. Treat 403 as "check the key and its company scope".
- Keep the `x-request-id` response header from any failing call; it is the only correlation
  handle Canix support has.

## Steps

1. **Resolve the facility.** Call `GetFacilities`. Pick the facility whose license the order is
   being written against and hold its `id`. If the company has one facility this is still worth
   doing — the id is required downstream.

2. **Resolve the customer.** Call `GetCustomers` with a `where` filter rather than paging the
   whole book, e.g. `where=name LIKE 'Green%25' AND is_active=true`. Confirm the match with
   `GetCustomerById`, which returns the extended record including `address`. If more than one
   customer matches, stop and ask — do not guess which licensee is being sold to.

3. **Resolve the items being sold.** Call `GetItems` with a `where` clause on the facility and
   the SKU or name, e.g. `where=facility_id=<facility_id> AND sku LIKE 'ABC%25'`. Each `Item`
   carries `strain`, `type`, `sub_type`, `brand` and `current_standard_cost`. Capture the item
   `id` and the unit of measure for each line.

4. **Create the order.** `POST` to `CreateSalesOrder` with the customer id, the facility, and
   the line contents. Send exactly one request. If the connection drops, go to step 4a.

   4a. **Recovery, not retry.** Call `GetSalesOrders` with
   `where=customer_id=<customer_id>` and `order_by=id desc`, `limit=10`. If the order is there,
   continue from step 5 with its id. Only create again if it is genuinely absent.

5. **Handle the queued case.** Some Canix writes are queued rather than applied inline and
   return a `Submission`. If the response is a submission, poll `GetSubmissionById` on
   `/submissions/{submission_id}` until `status` reaches a terminal value. Back off between
   polls; there is no rate-limit header telling you how fast is too fast, so be conservative.

6. **Verify what actually landed.** Call `GetSalesOrderById`, then `GetSalesOrderContentsById`
   for the line detail. Compare every line's item, quantity and price against what you sent.
   Do not report success on the basis of the create response alone.

7. **Advance the status.** Call `UpdateSalesOrderStatus` with the target `status_name` on
   `/sales_orders/{sales_order_id}/status/{status_name}`. Status names are configured per
   company — read the current value off `GetSalesOrderById` first and only move to a status you
   have seen the company use. Do not invent a status string.

8. **Reconcile payments.** Call `GetSalesOrderPayments` for payments booked against this order.
   For a company-wide view use `GetPayments` with a `where` clause on the date range.

## Compliance note

A Canix sales order for cannabis inventory has a downstream track-and-trace consequence — the
associated packages and transfers are synchronized to Metrc, BioTrack or CCRS depending on the
state. This skill covers the Canix side only. Do not assume a sales order alone satisfies a
state reporting obligation, and do not attempt to reconcile state-system records through this
API; the operator's compliance workflow governs that.

## Failure modes

| Status | Meaning here | Do |
|---|---|---|
| 400 | Malformed `where`, non-integer `limit`/`offset`, or `limit` > 2000 | Fix the query; cap `limit` at 2000 |
| 401 | No `X-API-KEY` header | Supply the key |
| 403 | Key not authorized for this company/facility/record — also returned anonymously | Re-check key scope |
| 404 | Id does not exist or is outside the key's company scope | List the collection first |
| 422 | Body well-formed but semantically invalid | Check required attributes and referential ids |
| 500 | Server error | Back off, then check https://status.canix.com/ and quote `x-request-id` |

See `errors/canix-problem-types.yml` and `conventions/canix-conventions.yml`.
