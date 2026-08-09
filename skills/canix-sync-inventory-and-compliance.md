---
name: Sync Canix inventory and compliance records
description: Pull a facility's current cannabis inventory out of Canix — packages, locations,
  plants, plant batches, harvests and transfers — incrementally, and read the audit trail behind
  every adjustment, split, combine and location change.
api: openapi/canix-openapi-original.yml
generated: '2026-08-09'
method: generated
source: openapi/canix-openapi-original.yml
operations:
  - GetFacilities
  - GetLocations
  - GetLocationsCount
  - GetPackages
  - GetPackageById
  - GetPlantBatches
  - GetPlants
  - GetPlantsCount
  - GetPlantById
  - GetHarvests
  - GetHarvestById
  - GetTransfers
  - GetTransferById
  - GetTransferDestinationById
  - GetAuditedActions
---

# Sync Canix inventory and compliance records

Base URL `https://api.canix.com/api/v1`, `X-API-KEY: <key>`. This is a read-only skill —
every operation below is a GET. Nothing here mutates Canix or the state track-and-trace system.

## The incremental sync idiom

Canix has no changed-since endpoint and no events. The documented pattern is a `where` clause
on a timestamp column, combined with deterministic ordering:

```
GET /packages?where=updated_at >= '2026-08-01' AND facility_id=123&order_by=id desc&limit=2000&offset=0
```

Rules that apply to every collection call:

- `limit` defaults to **2000** and **cannot exceed 2000**. A larger value returns 400.
- Collections come back as a **bare JSON array** — no total, no cursor, no `Link` header. Page by
  incrementing `offset` by `limit` until a page returns fewer than `limit` records.
- Always set `order_by` (e.g. `id desc`). Paging an unordered collection can skip or duplicate rows.
- `where` is **SQL-like string syntax**: `=`, `>`, `<`, `>=`, `<=`, `BETWEEN`, `IN`, `LIKE`, `AND`,
  `OR`. Never interpolate untrusted user text into it — build clauses from validated values only.
- URL-encode the clause. `%` in a `LIKE` pattern must be sent as `%25`.

## Steps

1. **Scope to a facility.** `GetFacilities` → hold the `id`. Every subsequent filter should carry
   `facility_id=<id>`; a company key otherwise returns every facility's inventory mixed together.

2. **Load the location tree.** `GetLocationsCount` to size the job, then `GetLocations` paged.
   `Location` carries `parent_location`, so build the hierarchy client-side before assigning
   packages or plants to rooms.

3. **Pull packages — the compliance unit.** `GetPackages` with a `facility_id` + `updated_at`
   filter. Each `Package` embeds `item`, `location`, `brand`, `source_packages`,
   `destination_packages`, `test_results` and `lab_test_info`. `source_packages` and
   `destination_packages` are the lineage graph: they are what let you reconstruct splits and
   combines. Use `GetPackageById` only to re-read a single package after an audited change.

4. **Pull the cultivation spine.** In order: `GetPlantBatches` (batch → `strain`, `location`),
   then `GetPlants` (plant → `plant_batch`, `strain`, `location`, `harvest`) sized with
   `GetPlantsCount`, then `GetHarvests` (harvest → `strain`, `drying_location`). `GetPlantById`
   and `GetHarvestById` are for single-record refresh. Plant counts run large — page carefully.

5. **Pull transfers.** `GetTransfers` returns the manifest header with `destinations` and any
   linked `sales_order`. `GetTransferById` and `GetTransferDestinationById` give the per-stop
   detail, where `contents` is the list of `PackageTransfer` records — the packages actually
   moving to that licensee.

6. **Read the audit trail.** `GetAuditedActions` is the single most useful compliance endpoint
   in this API. It returns package adjustments, package splits, package combines, location
   changes for plant batches / plants / packages, immatures destroyed in plant batches, and the
   submitting and approving user for each — email and name on both sides. Filter it on
   `facility_id` and a date range to reconstruct exactly who changed what and who signed off.

## Reconciliation guidance

- Reconcile **packages against audited actions**, not against your own prior snapshot. A package
  whose weight moved without a matching audited action is the anomaly worth surfacing.
- Metrc, BioTrack and CCRS are the systems of record for the state, not Canix. If a Canix record
  disagrees with the state system, that is an operator escalation — report the discrepancy, do
  not attempt to reconcile it through this API.
- `status.canix.com` tracks **Metrc API**, **LeafLink**, **QuickBooks Online** and **Onfleet** as
  named components. Before reporting a sync gap as data loss, check whether an upstream
  integration was degraded during the window.

## Cost and courtesy

Canix publishes no rate limits and returns no `RateLimit-*` or `Retry-After` headers, which means
there is no signal telling you when you are being too aggressive. Treat the absence as a reason
for restraint, not licence: run full syncs off-hours, keep incremental windows narrow, and back
off on any 500. Check `https://status.canix.com/` before escalating repeated failures.
