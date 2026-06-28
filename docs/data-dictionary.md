# Project GIA Data Dictionary

This document defines the Phase 1 source-file contracts for Project GIA.
These contracts are the shared assumptions for ETL, DynamoDB design, fixtures,
and agent recommendations.

Phase 1 uses synthetic grocery data only. Do not include customer-level data,
payment data, loyalty identifiers, or other personally identifiable information.

## Mental Model

A data dictionary is a contract for source files.

For each file, it answers:

- What does one row represent?
- Which columns are required?
- Which columns are optional?
- What type and unit does each value use?
- Can the value be blank?
- What rules make a row valid or invalid?

If a source file follows this contract, the ETL loader should be able to validate
and load it consistently.

## Global Conventions

These rules apply to all Phase 1 source files unless a file-specific contract
says otherwise.

| Topic | Convention |
| --- | --- |
| File format | UTF-8 CSV with a header row |
| Delimiter | Comma |
| Empty strings | Treat as null only for optional fields |
| Required fields | Must be present, non-null, and non-blank |
| Date format | `YYYY-MM-DD`, for example `2026-06-28` |
| Timestamp format | ISO-8601 UTC, for example `2026-06-28T14:30:00Z` |
| Currency | Decimal US dollars, for example `3.99` |
| Quantity unit | Whole units for Phase 1, unless a column says otherwise |
| IDs | Stable strings with no leading or trailing whitespace |
| Boolean values | `true` or `false`, lowercase |
| Schema version | Use `1` for the first Phase 1 contract version |

Recommended ID examples:

- `store_id`: `S001`
- `sku_id`: `SKU-1001`
- `category_id`: `CAT-DAIRY`
- `supplier_id`: `SUP-LOCAL-01`

## Source File Summary

| File | Purpose | Row grain |
| --- | --- | --- |
| `sku_master.csv` | Defines the product catalog and product attributes | One row per SKU |
| `inventory_snapshot.csv` | Captures store-level inventory at a point in time | One row per store, SKU, and snapshot date |
| `daily_sales.csv` | Captures daily store-level sales | One row per store, SKU, and sales date |

## SKU Master Contract

### Purpose

`sku_master.csv` defines the grocery products that can appear in inventory and
sales files. ETL should load this file before validating inventory and sales so
those files can check that each `sku_id` exists.

### Grain

One row represents one active or inactive SKU in the product catalog.

### Required Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `sku_id` | string | No | Stable ID | Internal product identifier | Must be unique within `sku_master.csv` |
| `sku_name` | string | No | Text | Human-readable product name | Must not be blank |
| `category` | string | No | Text | Product category | Must not be blank |
| `department` | string | No | Text | Higher-level grocery department | Must not be blank |
| `unit_size` | decimal | No | See `unit_of_measure` | Sellable package size | Must be greater than `0` |
| `unit_of_measure` | string | No | `each`, `oz`, `lb`, `g`, `kg`, `ml`, `l` | Unit used with `unit_size` | Must be one of the allowed values |
| `base_price` | decimal | No | USD | Regular shelf price | Must be `0` or greater |
| `is_active` | boolean | No | `true` or `false` | Whether SKU is currently sold | Must be valid boolean |
| `schema_version` | integer | No | Version number | Contract version used by the row | Must be `1` for Phase 1 |

### Optional Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `upc` | string | Yes | Text digits | Barcode or UPC | If present, should contain only digits |
| `brand` | string | Yes | Text | Product brand | Blank allowed |
| `supplier_id` | string | Yes | Stable ID | Supplier identifier | Blank allowed |
| `case_pack` | integer | Yes | Units per case | Number of sellable units in one supplier case | If present, must be greater than `0` |
| `shelf_life_days` | integer | Yes | Days | Typical shelf life | If present, must be greater than `0` |
| `created_at` | timestamp | Yes | ISO-8601 UTC | Catalog row creation time | If present, must be valid timestamp |
| `updated_at` | timestamp | Yes | ISO-8601 UTC | Last catalog update time | If present, must be valid timestamp |

### Example Rows

```csv
sku_id,sku_name,category,department,unit_size,unit_of_measure,base_price,is_active,schema_version,upc,brand,supplier_id,case_pack,shelf_life_days
SKU-1001,Organic Whole Milk,Dairy,Refrigerated,64,oz,5.49,true,1,012345678901,Green Valley,SUP-DAIRY-01,6,14
SKU-2001,Classic Potato Chips,Snacks,Grocery,8,oz,3.99,true,1,012345678902,Crunch House,SUP-SNACK-02,12,180
```

### SKU Master Quality Checks

- Reject rows with missing required columns.
- Reject duplicate `sku_id` values.
- Reject negative prices.
- Reject unknown `unit_of_measure` values.
- Warn when optional `upc` is missing.
- Warn when `is_active=false` SKUs appear in current inventory or sales files.

## Inventory Snapshot Contract

### Purpose

`inventory_snapshot.csv` records how much inventory each store has for each SKU
on a specific business date. The Phase 1 agent will use this data to calculate
stock risk, days of supply, and clearance candidates.

### Grain

One row represents one SKU at one store on one snapshot date.

There should be at most one row for each combination of:

```text
store_id + sku_id + snapshot_date
```

### Required Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `store_id` | string | No | Stable ID | Store identifier | Must not be blank |
| `sku_id` | string | No | Stable ID | Product identifier | Must exist in `sku_master.csv` |
| `snapshot_date` | date | No | `YYYY-MM-DD` | Business date for the snapshot | Must be valid date |
| `on_hand_units` | integer | No | Sellable units | Current sellable units in the store | Must be `0` or greater |
| `on_order_units` | integer | No | Sellable units | Units already ordered but not received | Must be `0` or greater |
| `reorder_point_units` | integer | No | Sellable units | Current reorder threshold used by source system or fixture | Must be `0` or greater |
| `data_source` | string | No | Text | Source system or fixture name | Must not be blank |
| `schema_version` | integer | No | Version number | Contract version used by the row | Must be `1` for Phase 1 |

### Optional Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `reserved_units` | integer | Yes | Sellable units | Units reserved for orders or internal use | If present, must be `0` or greater |
| `damaged_units` | integer | Yes | Sellable units | Unsellable units still physically present | If present, must be `0` or greater |
| `last_received_date` | date | Yes | `YYYY-MM-DD` | Most recent receiving date | If present, must not be after `snapshot_date` |
| `last_counted_at` | timestamp | Yes | ISO-8601 UTC | Time of last inventory count | If present, must be valid timestamp |
| `location_code` | string | Yes | Text | Aisle, bay, or store fixture location | Blank allowed |
| `inventory_status` | string | Yes | `normal`, `low`, `out_of_stock`, `overstock`, `clearance` | Source inventory status | If present, must be one of the allowed values |

### Example Rows

```csv
store_id,sku_id,snapshot_date,on_hand_units,on_order_units,reorder_point_units,data_source,schema_version,reserved_units,damaged_units,last_received_date,inventory_status
S001,SKU-1001,2026-06-28,18,24,12,synthetic_fixture,1,0,0,2026-06-25,normal
S001,SKU-2001,2026-06-28,96,0,18,synthetic_fixture,1,0,2,2026-05-12,overstock
```

### Inventory Quality Checks

- Reject rows with missing required columns.
- Reject rows where `sku_id` is not found in `sku_master.csv`.
- Reject duplicate `store_id + sku_id + snapshot_date` rows.
- Reject negative inventory quantities.
- Reject `last_received_date` values after `snapshot_date`.
- Warn when `on_hand_units=0` and `inventory_status` is not `out_of_stock`.
- Warn when `on_hand_units` is much greater than `reorder_point_units`, because it may indicate overstock.

## Daily Sales Contract

### Purpose

`daily_sales.csv` records store-level SKU sales by business date. The Phase 1
agent will use this data to calculate sales velocity, slow-moving SKUs, and days
of supply.

### Grain

One row represents one SKU's total sales at one store on one sales date.

There should be at most one row for each combination of:

```text
store_id + sku_id + sales_date
```

### Required Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `store_id` | string | No | Stable ID | Store identifier | Must not be blank |
| `sku_id` | string | No | Stable ID | Product identifier | Must exist in `sku_master.csv` |
| `sales_date` | date | No | `YYYY-MM-DD` | Business date of sales | Must be valid date |
| `units_sold` | integer | No | Sellable units | Number of units sold that day | Must be `0` or greater |
| `gross_sales` | decimal | No | USD | Sales dollars before returns or discounts if source separates them | Must be `0` or greater |
| `data_source` | string | No | Text | Source system or fixture name | Must not be blank |
| `schema_version` | integer | No | Version number | Contract version used by the row | Must be `1` for Phase 1 |

### Optional Columns

| Column | Type | Null allowed? | Unit / format | Description | Validation rule |
| --- | --- | --- | --- | --- | --- |
| `net_sales` | decimal | Yes | USD | Sales dollars after returns and discounts | If present, must be `0` or greater |
| `return_units` | integer | Yes | Sellable units | Units returned on the sales date | If present, must be `0` or greater |
| `markdown_amount` | decimal | Yes | USD | Discount or markdown dollars applied | If present, must be `0` or greater |
| `promotion_id` | string | Yes | Stable ID | Promotion identifier | Blank allowed |
| `is_promotion` | boolean | Yes | `true` or `false` | Whether sales were promotion-influenced | If present, must be valid boolean |
| `updated_at` | timestamp | Yes | ISO-8601 UTC | Last source update time | If present, must be valid timestamp |

### Example Rows

```csv
store_id,sku_id,sales_date,units_sold,gross_sales,data_source,schema_version,net_sales,return_units,markdown_amount,is_promotion
S001,SKU-1001,2026-06-27,14,76.86,synthetic_fixture,1,76.86,0,0.00,false
S001,SKU-2001,2026-06-27,1,3.99,synthetic_fixture,1,3.49,0,0.50,true
```

### Daily Sales Quality Checks

- Reject rows with missing required columns.
- Reject rows where `sku_id` is not found in `sku_master.csv`.
- Reject duplicate `store_id + sku_id + sales_date` rows.
- Reject negative units, sales, returns, or markdown values.
- Warn when `units_sold=0` but `gross_sales>0`.
- Warn when `units_sold>0` but `gross_sales=0`.
- Warn when `return_units` is greater than `units_sold`.
- Warn when `net_sales` is greater than `gross_sales` unless there is a documented source-system reason.

## Cross-File Validation Rules

ETL should validate each file independently first, then validate relationships
across files.

| Rule | Severity | Reason |
| --- | --- | --- |
| Every inventory `sku_id` must exist in `sku_master.csv` | Reject | Inventory cannot be interpreted without product metadata |
| Every sales `sku_id` must exist in `sku_master.csv` | Reject | Sales cannot be interpreted without product metadata |
| `schema_version` must be `1` in all Phase 1 files | Reject | Prevents mixing incompatible contracts |
| Inventory and sales dates should not be in the future relative to ETL run date | Reject or warn | Future business dates usually indicate bad source data |
| Inventory snapshots should cover the same `snapshot_date` for a given ETL run | Warn | Mixed dates may still be valid, but the agent needs clear freshness |
| Sales history should include a continuous lookback window where possible | Warn | Missing dates reduce recommendation confidence |
| Inactive SKUs with recent sales or inventory should be flagged | Warn | May indicate closeout, setup error, or delayed catalog update |

## ETL Load Expectations

The Phase 1 ETL should produce actionable validation output.

Minimum load result fields:

| Field | Type | Description |
| --- | --- | --- |
| `run_id` | string | Unique load run identifier |
| `source_file` | string | File that was processed |
| `schema_version` | integer | Contract version used |
| `started_at` | timestamp | Load start time |
| `completed_at` | timestamp | Load completion time |
| `row_count` | integer | Total rows read |
| `accepted_count` | integer | Rows accepted |
| `rejected_count` | integer | Rows rejected |
| `warning_count` | integer | Non-blocking issues found |
| `freshness_date` | date | Business date represented by the file |

Validation errors should include:

- file name
- row number
- column name
- rejected value
- plain-language reason

Example:

```text
inventory_snapshot.csv row 18 column on_hand_units: value "-3" is invalid because inventory quantities must be 0 or greater.
```

## Assumptions

- Phase 1 works with synthetic, non-sensitive grocery data only.
- `sku_id` is the stable internal product key used across all source files.
- `store_id` is the stable internal store key.
- Inventory quantities are whole sellable units in Phase 1.
- Daily sales are already aggregated by store, SKU, and business date before ETL.
- The SKU master is loaded before inventory and sales validation.
- Dates use the store's business calendar date, not the exact transaction timestamp.
- Currency values are stored as decimal dollars in source CSVs; implementation may use a safer numeric type internally.
- `gross_sales` and `net_sales` semantics may vary by source system, so Phase 1 recommendations should rely primarily on units for movement calculations.

## Open Questions for Teammate Review

- Should Phase 1 support weighted grocery items, such as produce sold by pounds, or keep all quantities as whole units until a later phase?
- Should `reorder_point_units` come from the source inventory file, or should Project GIA calculate it entirely from configured rules?
- Do we need a separate `store_master.csv`, or is `store_id` alone enough for the Phase 1 MVP?
- Should discontinued or clearance SKUs use `is_active=false`, `inventory_status=clearance`, or both?
- Should sales files include zero-sale rows for every SKU/date, or should missing rows be interpreted as zero sales only after ETL fills the date grid?
- What exact lookback windows should fixtures support for slow-mover and reorder calculations, such as 7 days, 28 days, or both?
- Should markdowns be tracked in dollars, percent, or both for clearance recommendations?
