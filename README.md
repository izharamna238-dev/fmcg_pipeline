# FMCG Retail Data Engineering Pipeline

An end-to-end batch data engineering pipeline built on **Databricks** that ingests FMCG retail data (customers, products, pricing, and orders) from an **AWS S3 data lake** into a **Unity Catalog** lakehouse using the **medallion architecture** (Bronze → Silver → Gold), with Delta Lake `MERGE`-based idempotent loading for both full and incremental loads.

## Architecture

```
AWS S3 (raw CSVs)
        │
        ▼
┌──────────────────────────────────────────────┐
│  Bronze (fmcg.bronze)   – raw, as-is landing  │
├──────────────────────────────────────────────┤
│  Silver (fmcg.silver)   – cleaned, deduped,   │
│                         typed, conformed      │
├──────────────────────────────────────────────┤
│  Gold (fmcg.gold)       – fact tables &        │
│                         monthly aggregates    │
└──────────────────────────────────────────────┘
```

- **Bronze**: raw CSV data landed as Delta tables exactly as received (with Change Data Feed on orders).
- **Silver**: cleaned data — date parsing for messy formats, invalid customer ID handling, deduplication, inner joins to product dimension.
- **Gold**: order-level fact table plus a monthly aggregated fact table, loaded idempotently via `MERGE`.

## Data Source

Raw CSV files live flat at the root of the `s3://amina-sportsdb` bucket:

| File | Description |
| --- | --- |
| `products.csv` | Product master (product_name, product_id, category) |
| `gross_price.csv` | Product pricing |
| `orders_YYYY_MM_DD.csv` | Daily order extracts (Jul–Nov 2025, ~43.7k rows) |

## Project Structure

```
consolidated_pipeline/
├── 1_setup/
│   ├── setup                        # workspace/catalog setup
│   ├── utilities                    # shared schema variables (bronze/silver/gold)
│   └── dim_date_table_creation      # date dimension
├── 2_dimension_data/
│   ├── 1_customer dat processing    # customers → bronze/silver/gold
│   ├── 2_products_data_processing   # products → bronze/silver/gold
│   └── 3_pricing_data_processing    # gross price → bronze/silver/gold
└── fact_data_processing/
    ├── full_load_fact                # full order load: bronze → silver MERGE → gold
    └── 2_incremental_load_fact       # incremental order load (appends to bronze)
```

Run notebooks in folder order (`%run` the `utilities` notebook first — without the `/Workspace` prefix).

## Fact Processing Details

- **Bronze**: daily orders CSVs read and written to `fmcg.bronze.orders` (overwrite + CDF for full load; append for incremental).
- **Silver**: cleaning includes parsing non-standard dates (e.g. `"Tuesday, September 30, 2025"`), remapping invalid/`INVALID` customer IDs to `999999`, deduplication, and inner join to the products dimension.
- **Gold**:
  - `fmcg.gold.sb_fact_orders` — order-level fact table built from silver.
  - `fmcg.gold.fact_orders` — monthly rollup (month-start truncation) MERGEd on `date + product_code + customer_code`.
- Silver MERGE keys: `order_placement_date + order_id + product_code + customer_id`, making re-runs idempotent.

## Current Table State (after successful run)

| Table | Rows |
| --- | --- |
| `fmcg.bronze.orders` | 43,701 |
| `fmcg.silver.orders` | 34,418 |
| `fmcg.gold.sb_fact_orders` | 34,418 |
| `fmcg.gold.fact_orders` | ~96,112 (monthly aggregates) |

## Tech Stack

- Databricks Notebooks (PySpark)
- Delta Lake (MERGE, Change Data Feed)
- Unity Catalog (`fmcg` catalog with `bronze` / `silver` / `gold` schemas)
- AWS S3 (raw data lake)
