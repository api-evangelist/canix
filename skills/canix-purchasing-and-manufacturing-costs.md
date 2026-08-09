---
name: Run Canix purchasing and roll up manufacturing costs
description: Onboard a vendor, raise a purchase order against it, then trace a manufacturing
  batch run from its bill of materials through its cannabis and non-cannabis inputs, labor and
  waste, and maintain the standard costs the roll-up depends on.
api: openapi/canix-openapi-original.yml
generated: '2026-08-09'
method: generated
source: openapi/canix-openapi-original.yml
operations:
  - GetFacilities
  - GetVendors
  - CreateVendor
  - GetVendorById
  - GetWeightUnits
  - GetItems
  - GetNonCannabisProducts
  - PostPurchaseOrder
  - GetPurchaseOrderById
  - GetPurchaseOrderContentsById
  - GetpurchaseOrderPayments
  - GetBillOfMaterialsById
  - GetNonCannabisProductBOMs
  - GetManuBatches
  - GetManuBatchById
  - GetManuBatchRuns
  - GetManuBatchRunById
  - AddItemStandardCost
  - GetStandardCost
  - UpdateStandardCost
---

# Run Canix purchasing and roll up manufacturing costs

Base URL `https://api.canix.com/api/v1`, `X-API-KEY: <key>`.

This skill spans two write-bearing flows and one read-only roll-up. The writes —
`CreateVendor`, `PostPurchaseOrder`, `AddItemStandardCost`, `UpdateStandardCost` — have **no
idempotency contract**. Treat every one as create-once and verify.

## Part 1 — vendor and purchase order

1. **Resolve the facility.** `GetFacilities`, hold the `id`.

2. **Find or create the vendor.** `GetVendors` with
   `where=name LIKE '<vendor>%25'` first. Only call `CreateVendor` when the search genuinely
   misses — duplicate vendor records corrupt the purchasing history and are painful to unwind.
   Confirm with `GetVendorById`.

3. **Resolve units and line items.** `GetWeightUnits` for the `weight_unit_id` each line needs.
   Then resolve the goods being bought: `GetItems` for cannabis items, `GetNonCannabisProducts`
   for packaging, nutrients and supplies. A purchase order line references **either** an
   `item_id` **or** a `non_cannabis_product_id`, not both.

4. **Raise the purchase order.** `PostPurchaseOrder` with `facility_id`, `vendor_id` and the
   line array. Send once.

   4a. **If it times out**, call `GetPurchaseOrders` with
   `where=vendor_id=<vendor_id>`, `order_by=id desc`, `limit=10` and check before re-sending.

5. **Verify and track.** `GetPurchaseOrderById` for the header,
   `GetPurchaseOrderContentsById` for the lines, `GetpurchaseOrderPayments` for money booked
   against it. Note the lowercase `p` in `GetpurchaseOrderPayments` — it is spelled that way in
   Canix's spec and must be used verbatim.

## Part 2 — manufacturing roll-up (read-only)

6. **Start from the bill of materials.** `GetBillOfMaterialsById` returns a BOM with
   `source_cannabis_items`, `source_non_cannabis_products` and `output_items`. For a specific
   packaging product, `GetNonCannabisProductBOMs` lists the BOMs it participates in.

7. **List the batches.** `GetManuBatches`, then `GetManuBatchById` for one batch.

8. **Open the runs.** `GetManuBatchRuns`, then `GetManuBatchRunById`. A `ManuBatchRun` is the
   full cost object: `bill_of_materials_id`, `location_id`, `machine_info`, plus arrays of
   `cannabis_inputs` (each referencing a `package_id`), `non_cannabis_inputs` (each referencing
   a `non_cannabis_product_id`), `cannabis_outputs` (each producing a `package_id`), `labors`
   and `wastes` (each referencing the `package_id` destroyed).

9. **Roll up.** Cost of a run = valued cannabis inputs + valued non-cannabis inputs + labor,
   apportioned across the cannabis outputs, with waste written off. Values come from standard
   costs, which is why Part 3 matters.

## Part 3 — maintaining standard costs

10. **Set a cost.** `AddItemStandardCost` on `/items/{item_id}/standard_cost` creates a new
    standard cost for an item. Read it back with `GetStandardCost`.

11. **Amend a cost.** `UpdateStandardCost` on `/standard_costs/{standard_cost_id}`.
    `DeleteStandardCost` exists but is destructive and changes historical valuation — do not
    call it autonomously. Surface the intent to a human and let them confirm.

12. **Check what an item currently values at.** `GetItemById` returns
    `current_standard_cost` inline, which is cheaper than resolving the standard-cost record.

## Guardrails

- **Never blind-retry a write.** No `Idempotency-Key` header exists. A duplicated purchase order
  or a doubled standard cost silently corrupts margin reporting.
- **Deletes are out of scope for autonomous execution.** `DeleteStandardCost`, `DeleteItem` and
  `DeleteVendor` all change historical records. Escalate to a human.
- **Costing is financial data.** A wrong standard cost propagates into every manufacturing run
  that consumes the item. Verify a written cost by reading it back before reporting success.
- Poll `GetSubmissionById` if a write returns a `Submission` rather than the created record.

See `conventions/canix-conventions.yml` for the shared query grammar and
`errors/canix-problem-types.yml` for the error envelope.
