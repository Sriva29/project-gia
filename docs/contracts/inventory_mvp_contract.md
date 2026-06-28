# Phase 1 Inventory MVP Contract

Status: proposed for GitHub issue #1  
Contract version: `inventory-mvp.v1`  
Scope: Phase 1 local MVP contracts for inventory inputs, deterministic policy calculations, and recommendation outputs.

This document is the shared contract between the data, MCP, and agent workstreams until the same shapes are implemented as versioned Pydantic and JSON Schema definitions in `packages/shared`.

For a visual walkthrough and review prompts, open [`inventory_mvp_contract_explainer.html`](inventory_mvp_contract_explainer.html).

## Goals

- Keep inventory, sales, run metadata, and recommendation records stable before implementation begins.
- Make slow-mover, reorder-point, and clearance decisions deterministic and reviewable.
- Ensure the LLM only explains supplied facts; it must not invent metrics, thresholds, dates, or financial impact.
- Preserve enough evidence for JSON/CSV output, tests, and later feedback records.

## Global Rules

- All records include `schema_version: "inventory-mvp.v1"` unless a narrower nested version is specified.
- Dates are ISO-8601 calendar dates, `YYYY-MM-DD`, interpreted in the store's local business calendar.
- Timestamps are ISO-8601 UTC datetimes with `Z`.
- Unit counts are non-negative numbers. Whole-unit fields use integers unless noted.
- Money values are decimal numbers in USD for Phase 1 fixtures.
- Percent values are decimals from `0` to `1`, for example `0.15` for 15%.
- IDs are stable strings. `run_id` and `recommendation_id` are UUID strings.
- Phase 1 uses synthetic, non-sensitive data only. No customer-level data or PII is allowed.
- Missing required fields fail validation. Missing optional fields must either use the documented default or produce a `data_insufficient` policy outcome.

## Input Contract: InventorySnapshot

One current inventory state for one SKU at one store as of a business date.

| Field | Type | Required | Units / allowed values | Notes |
| --- | --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1` | Contract version. |
| `store_id` | string | yes | stable store ID | Example: `S001`. |
| `sku_id` | string | yes | stable SKU ID | Example: `SKU-123`. |
| `snapshot_date` | date | yes | `YYYY-MM-DD` | Inventory position date. |
| `product_name` | string | yes | 1-160 chars | Human-readable output label. |
| `category` | string | yes | 1-80 chars | Used for grouping and review. |
| `on_hand_units` | integer | yes | units | Physical units in store. |
| `reserved_units` | integer | no | units, default `0` | Units not available for sale. |
| `on_order_units` | integer | no | units, default `0` | Open inbound units expected within normal lead time. |
| `unit_cost` | number | no | USD/unit | Used only as evidence; Phase 1 must not invent margin. |
| `retail_price` | number | no | USD/unit | Used only as evidence. |
| `last_received_date` | date | no | `YYYY-MM-DD` | Preferred source for inventory age. |
| `inventory_age_days` | integer | no | days | Required when `last_received_date` is unavailable and clearance is evaluated. |
| `supplier_lead_time_days` | integer | no | days | Overrides policy default when present. |
| `minimum_stock_units` | integer | no | units | Overrides policy default when present. |
| `case_pack_units` | integer | no | units, default `1` | Reorder quantity rounds up to this multiple. |
| `data_quality_flags` | string[] | no | see below | Non-fatal quality warnings. |
| `updated_at` | datetime | yes | UTC | Source freshness timestamp. |

Allowed `data_quality_flags` values for Phase 1:

- `missing_optional_cost`
- `missing_last_received_date`
- `lead_time_defaulted`
- `minimum_stock_defaulted`
- `stale_inventory_snapshot`
- `source_row_warning`

Derived inventory fields:

- `sellable_units = max(on_hand_units - reserved_units, 0)`
- `inventory_position_units = sellable_units + on_order_units`
- `age_days = inventory_age_days` when provided, otherwise `as_of_date - last_received_date`

If both `last_received_date` and `inventory_age_days` are missing, clearance evaluation returns `data_insufficient`.

## Input Contract: SalesDaily

One daily sales aggregate for one SKU at one store.

| Field | Type | Required | Units / allowed values | Notes |
| --- | --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1` | Contract version. |
| `store_id` | string | yes | stable store ID | Must match inventory store. |
| `sku_id` | string | yes | stable SKU ID | Must match inventory SKU. |
| `sales_date` | date | yes | `YYYY-MM-DD` | Business date. |
| `units_sold` | integer | yes | units | Can be `0`; cannot be negative. |
| `gross_sales_amount` | number | no | USD | Optional Phase 1 evidence. |
| `source_file` | string | no | file name | Traceability for fixture/debug output. |
| `loaded_at` | datetime | yes | UTC | ETL load timestamp. |
| `data_quality_flags` | string[] | no | see below | Non-fatal quality warnings. |

Allowed `SalesDaily.data_quality_flags` values for Phase 1:

- `zero_sales_day`
- `corrected_duplicate_day`
- `source_row_warning`

Sales history completeness:

- The lookback window is inclusive of `as_of_date` and contains exactly `lookback_window_days` calendar dates.
- A zero-sales day must be represented by a `SalesDaily` record with `units_sold: 0`.
- Missing daily records inside the lookback window make sales history incomplete for that SKU.
- Phase 1 calculators may still emit `data_insufficient` output for incomplete history, but they must not fill missing days with zero without an explicit ETL quality decision.

## Calculation Input Contract

The agent or MCP layer passes one calculation request per store/SKU/as-of-date.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1`. |
| `run_id` | UUID string | yes | Shared across all SKUs in one run. |
| `as_of_date` | date | yes | Policy evaluation date. |
| `store_id` | string | yes | Store to evaluate. |
| `sku_id` | string | yes | SKU to evaluate. |
| `inventory_snapshot` | `InventorySnapshot` | yes | Current state. |
| `sales_history` | `SalesDaily[]` | yes | Daily rows for the configured lookback window. |
| `policy` | `InventoryPolicy` | yes | Defaults below, with per-run overrides allowed. |

## InventoryPolicy Defaults

These defaults are intentionally simple and deterministic for synthetic Phase 1 fixtures. Later phases can add category-specific policies behind explicit contract versions.

| Parameter | Default | Units | Applies to | Notes |
| --- | ---: | --- | --- | --- |
| `lookback_window_days` | `28` | days | all policies | Sales velocity window. |
| `min_complete_sales_days` | `28` | days | all policies | Missing days produce `data_insufficient`. |
| `slow_mover_max_units_sold` | `4` | units / lookback | slow mover | SKU qualifies when units sold are at or below this threshold and stock remains available. |
| `slow_mover_min_days_of_supply` | `60` | days | slow mover | SKU qualifies when days of supply is at or above this threshold. |
| `slow_mover_min_sellable_units` | `5` | units | slow mover | Avoids flagging trivial leftovers. |
| `default_lead_time_days` | `7` | days | reorder | Used when inventory snapshot has no SKU-specific lead time. |
| `safety_stock_days` | `3` | days of average demand | reorder | Phase 1 fixed assumption, not a calibrated service-level model. |
| `min_safety_stock_units` | `2` | units | reorder | Floor for safety stock when there is observed demand. |
| `default_minimum_stock_units` | `3` | units | reorder | Used when inventory snapshot has no SKU-specific minimum. |
| `clearance_min_age_days` | `45` | days | clearance | Minimum inventory age before clearance can be recommended. |
| `clearance_min_days_of_supply` | `45` | days | clearance | Excess supply threshold for clearance. |
| `clearance_min_sellable_units` | `5` | units | clearance | Avoids markdown recommendations for negligible stock. |
| `markdown_band_low` | `0.10` | percent | clearance | Suggested when age is 45-59 days or supply is 45-74 days. |
| `markdown_band_medium` | `0.20` | percent | clearance | Suggested when age is 60-89 days or supply is 75-119 days. |
| `markdown_band_high` | `0.30` | percent | clearance | Suggested when age is 90+ days or supply is 120+ days. |
| `confidence_floor` | `0.20` | score | all policies | Lowest score for valid but weak evidence. |
| `confidence_ceiling_phase1` | `0.90` | score | all policies | Phase 1 has no feedback retrieval, so confidence stays heuristic. |

## Deterministic Calculations

For a complete lookback window:

- `units_sold_lookback = sum(units_sold)`
- `average_daily_sales = units_sold_lookback / lookback_window_days`
- `sellable_units = max(on_hand_units - reserved_units, 0)`
- `inventory_position_units = sellable_units + on_order_units`
- `days_of_supply = sellable_units / average_daily_sales` when `average_daily_sales > 0`
- `days_of_supply = null` when `average_daily_sales = 0`; policy comparisons treat this as excess supply only when sellable stock is positive and sales history is complete.
- `effective_lead_time_days = supplier_lead_time_days` from inventory when present, otherwise `default_lead_time_days`
- `safety_stock_units = max(min_safety_stock_units, ceil(average_daily_sales * safety_stock_days))` when demand is observed
- `safety_stock_units = 0` when `average_daily_sales = 0`
- `reorder_point_units = max(minimum_stock_units or default_minimum_stock_units, ceil(average_daily_sales * effective_lead_time_days + safety_stock_units))`
- `reorder_quantity_units = max(0, reorder_point_units - inventory_position_units)`, rounded up to the next multiple of `case_pack_units`

Policy decisions:

- Slow mover: `decision = "recommend"` when sales history is complete, `sellable_units >= slow_mover_min_sellable_units`, and either `units_sold_lookback <= slow_mover_max_units_sold` or `days_of_supply >= slow_mover_min_days_of_supply`.
- Reorder: `decision = "recommend"` when sales history is complete and `inventory_position_units <= reorder_point_units`.
- Clearance: `decision = "recommend"` when sales history is complete, age is known, `sellable_units >= clearance_min_sellable_units`, `age_days >= clearance_min_age_days`, and days of supply is above the clearance threshold or zero demand was observed.
- Data insufficient: `decision = "data_insufficient"` when required inputs for a policy are missing, stale, invalid, or incomplete. Include missing fields in evidence and reasoning.
- No action: `decision = "no_action"` when inputs are valid but thresholds are not met.

Markdown band selection:

- Use the highest band whose age or days-of-supply threshold is met.
- Low: age 45-59 days or days of supply 45-74.
- Medium: age 60-89 days or days of supply 75-119.
- High: age 90+ days or days of supply 120+.
- If `average_daily_sales = 0` and age qualifies, use at least the medium band; use high when age is 90+ days.

## Calculation Output Contract

Each policy evaluation can produce a calculation result before recommendation wording is composed.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1`. |
| `run_id` | UUID string | yes | Parent run. |
| `store_id` | string | yes | Store evaluated. |
| `sku_id` | string | yes | SKU evaluated. |
| `policy_type` | string | yes | `slow_mover`, `reorder`, or `clearance`. |
| `decision` | string | yes | `recommend`, `watch`, `no_action`, or `data_insufficient`. |
| `metrics` | object | yes | Numeric results used by the policy. |
| `thresholds` | object | yes | Policy values used for the decision. |
| `missing_inputs` | string[] | yes | Empty when complete. |
| `reason_codes` | string[] | yes | Stable machine-readable reasons. |

Required `metrics` keys where calculable:

- `units_sold_lookback`
- `average_daily_sales`
- `sellable_units`
- `inventory_position_units`
- `days_of_supply`
- `age_days`
- `safety_stock_units`
- `reorder_point_units`
- `reorder_quantity_units`
- `suggested_markdown_pct`

Use `null` for non-applicable metrics and include a reason code. Do not omit a metric because a policy did not need it.

## Output Contract: Recommendation

The recommendation is the stable record written to JSON and CSV. The LLM may help write `action` and `reasoning`, but only from the calculation result and source evidence.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1`. |
| `recommendation_id` | UUID string | yes | Stable record ID. |
| `run_id` | UUID string | yes | Parent run. |
| `store_id` | string | yes | Store evaluated. |
| `sku_id` | string | yes | SKU evaluated. |
| `product_name` | string | yes | Copied from `InventorySnapshot`. |
| `category` | string | yes | Copied from `InventorySnapshot`. |
| `type` | string | yes | `slow_mover`, `reorder`, or `clearance`. |
| `decision` | string | yes | `recommend`, `watch`, `no_action`, or `data_insufficient`. |
| `priority` | string | yes | `high`, `medium`, `low`, or `none`. |
| `action` | string | yes | Concise action or explicit insufficiency statement. |
| `evidence` | object | yes | Source fields and calculated metrics. |
| `reasoning` | string[] | yes | Short, evidence-backed bullets. |
| `confidence` | number | yes | `0` to `1`, heuristic. |
| `confidence_factors` | string[] | yes | Why confidence is high/low. |
| `policy_version` | string | yes | `inventory-mvp.v1`. |
| `status` | string | yes | `new`, `reviewed`, `accepted`, or `rejected`; default `new`. |
| `created_at` | datetime | yes | UTC. |

Priority guidance:

- `high`: stockout/reorder risk with positive demand, or clearance age 90+ days, or very large excess supply.
- `medium`: qualifying slow mover or clearance candidate with clear evidence.
- `low`: valid but weak signal, watch-list style recommendation.
- `none`: `no_action` or `data_insufficient`.

Evidence must include, at minimum:

- `snapshot_date`
- `lookback_start_date`
- `lookback_end_date`
- `complete_sales_days`
- `units_sold_lookback`
- `average_daily_sales`
- `sellable_units`
- `inventory_position_units`
- `days_of_supply`
- `age_days`
- policy-specific threshold values used

## Output Contract: RunMetadata

One record per agent run.

| Field | Type | Required | Notes |
| --- | --- | --- | --- |
| `schema_version` | string | yes | `inventory-mvp.v1`. |
| `run_id` | UUID string | yes | Shared run ID. |
| `store_id` | string | yes | Store evaluated. |
| `as_of_date` | date | yes | Business date. |
| `started_at` | datetime | yes | UTC. |
| `completed_at` | datetime | no | UTC, present on successful or failed completion. |
| `status` | string | yes | `running`, `succeeded`, `failed`, or `partial`. |
| `policy_version` | string | yes | `inventory-mvp.v1`. |
| `input_counts` | object | yes | Counts for inventory snapshots, sales rows, and SKUs evaluated. |
| `output_counts` | object | yes | Counts by recommendation type and decision. |
| `data_freshness` | object | yes | Max/min source dates and load timestamps. |
| `model` | object | yes | Local model metadata or `{ "used": false }` for deterministic-only tests. |
| `warnings` | string[] | yes | Empty when none. |
| `errors` | string[] | yes | Empty when none. |

## JSON Output Shape

`recommendations.json` contains one run envelope:

```json
{
  "schema_version": "inventory-mvp.v1",
  "run_metadata": {},
  "recommendations": []
}
```

`recommendations.csv` contains one row per `Recommendation`. Nested `evidence`, `reasoning`, and `confidence_factors` fields are serialized as compact JSON strings unless a later CSV-specific contract narrows the columns.

`run-metadata.json` contains the same `RunMetadata` object included in the JSON envelope.

## Example Recommendation

```json
{
  "schema_version": "inventory-mvp.v1",
  "recommendation_id": "9f10d90c-6f88-42f2-9f14-47cfb25af9ac",
  "run_id": "8ce39496-d8a1-4d77-911c-166f8d8e83b1",
  "store_id": "S001",
  "sku_id": "SKU-123",
  "product_name": "Organic Blueberry Yogurt",
  "category": "Dairy",
  "type": "clearance",
  "decision": "recommend",
  "priority": "medium",
  "action": "Apply a 20% markdown and review shelf placement for the next replenishment cycle.",
  "evidence": {
    "snapshot_date": "2026-06-28",
    "lookback_start_date": "2026-06-01",
    "lookback_end_date": "2026-06-28",
    "complete_sales_days": 28,
    "units_sold_lookback": 4,
    "average_daily_sales": 0.14,
    "sellable_units": 18,
    "inventory_position_units": 18,
    "days_of_supply": 126,
    "age_days": 64,
    "clearance_min_age_days": 45,
    "clearance_min_days_of_supply": 45,
    "suggested_markdown_pct": 0.2
  },
  "reasoning": [
    "The SKU has sold 4 units over the 28-day lookback window.",
    "Current sellable inventory covers about 126 days at the observed sales rate.",
    "Inventory age and days of supply both exceed the Phase 1 clearance thresholds."
  ],
  "confidence": 0.82,
  "confidence_factors": [
    "28 complete daily sales records",
    "Inventory age is available",
    "Clearance signal exceeds both age and supply thresholds"
  ],
  "policy_version": "inventory-mvp.v1",
  "status": "new",
  "created_at": "2026-06-28T22:00:00Z"
}
```

## Confidence Assumptions

Phase 1 confidence is an explainable heuristic, not a calibrated probability.

Recommended scoring inputs:

- Data completeness: full sales window, current inventory snapshot, age availability.
- Signal strength: distance from relevant thresholds.
- Data quality: stale snapshots, defaulted lead time, missing optional commercial fields.
- Feedback agreement: not used in Phase 1; include a confidence factor such as `feedback_not_enabled_phase1` only if needed for traceability.

The score must never exceed `confidence_ceiling_phase1` and must include plain-language factors.

## Acceptance Criteria

This contract is accepted for issue #1 when:

- Versioned contracts for `InventorySnapshot`, `SalesDaily`, `Recommendation`, `RunMetadata`, and calculation inputs/outputs are documented.
- Slow-mover, reorder-point, and clearance policy parameters have units and Phase 1 defaults.
- Deterministic formulas define sales velocity, days of supply, reorder point, reorder quantity, clearance eligibility, markdown bands, and data-insufficient outcomes.
- JSON and CSV output expectations are clear enough for implementation and tests.
- `PROJECT_BRIEF.md` links to this document as the Phase 1 inventory contract source.
- Srivatsan and Beula approve the contract in the linked GitHub issue or pull request before data and agent implementation work begins.
