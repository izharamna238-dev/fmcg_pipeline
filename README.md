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

## How to Run

1. **Prerequisites** — a Databricks workspace with Unity Catalog enabled, a compute cluster (DBR 13+ recommended), and access to the raw S3 bucket (`s3://amina-sportsdb`).
2. **Setup** — run `1_setup/setup` to create the `fmcg` catalog and its `bronze` / `silver` / `gold` schemas, then run `1_setup/dim_date_table_creation` for the date dimension.
3. **Dimensions** — run the three notebooks in `2_dimension_data/` in order (customers → products → pricing).
4. **Facts** — run `fact_data_processing/full_load_fact` for the initial full order load; use `fact_data_processing/2_incremental_load_fact` for subsequent daily order extracts.
5. **Shared config** — `1_setup/utilities` holds the schema variables used across notebooks; `%run` it (without the `/Workspace` prefix) before any processing notebook.

Re-running any notebook is safe: every load is idempotent via Delta `MERGE` on the natural keys.

## Data Quality Highlights

- **Messy date handling** — order dates arrive in non-standard formats (e.g. `"Tuesday, September 30, 2025"`) and are parsed before typing.
- **Invalid key remediation** — unknown / `INVALID` customer IDs are remapped to a sentinel key (`999999`) instead of being dropped.
- **Deduplication** — duplicate orders are removed in Silver with `dropDuplicates` on the natural key.
- **Referential integrity** — facts are inner-joined to the product dimension so only valid products reach Gold.
- **Incremental safety** — daily appends to Bronze never duplicate downstream thanks to MERGE-based Silver/Gold loads.

## Tech Stack

- Databricks Notebooks (PySpark)
- Delta Lake (MERGE, Change Data Feed)
- Unity Catalog (`fmcg` catalog with `bronze` / `silver` / `gold` schemas)
- AWS S3 (raw data lake)

## CI/CD — Auto-run on Push

A GitHub Actions workflow (`.github/workflows/run-pipeline.yml`) triggers the Databricks job `fmcg_pipeline` on every push to `main` (and manually via *Run workflow*). The job runs the six pipeline notebooks in order on serverless compute: setup → date dimension → customers → products → pricing → full fact load.

To enable it, add two repository secrets under **Settings → Secrets and variables → Actions**:

- `DATABRICKS_HOST` — your Databricks workspace URL
- `DATABRICKS_TOKEN` — a Databricks access token (service principal recommended)

## Author

Built by [izharamna238-dev](https://github.com/izharamna238-dev) as an end-to-end FMCG retail data engineering showcase.
