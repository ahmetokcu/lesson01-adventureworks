# lesson01-adventureworks — AdventureWorks Star Schemas on Databricks

Team repository for the course **Data Pipelines with Databricks and Fabric** (Dr. Andreas Wagner), built on the **Microsoft AdventureWorks2022** dataset.

**Team:** Ahmet Okcu, Harsha Vardhan Thigulla, Vikas Pandey

---

## Contents

| File | Description |
| --- | --- |
| `PurchasingStar_Ahmet.ipynb` | **Homework 2** — Build a Purchasing Star and link it to the Sales star |
| `HelloWorld.ipynb` | Lesson 01 Git/team exercise |

## One Lakehouse, Two Stars Sharing `dim_product`

| Star | Fact table | Dimensions |
| --- | --- | --- |
| **Sales** | `adventure_works.default.fact_sales` (121,317 rows) | `default.dim_product`, `default.dim_sales_territory` |
| **Purchasing** | `adventure_works.silver.fact_purchases` (8,845 rows) | `silver.dim_vendor` (+ shared `default.dim_product`) |

Both stars share `dim_product`, so sales and purchasing can be analyzed side by side per product (margin analysis, stock coverage).

## Homework 2 — Purchasing Star (Medallion Architecture)

The notebook `PurchasingStar_Ahmet.ipynb` builds the purchasing star with a strict **Bronze → Silver → Gold** flow. Source CSVs (tab-delimited, no header row) live in the UC volume `/Volumes/adventure_works/default/raw_data`.

### Bronze — raw staging (`adventure_works.bronze`)

* Load 3 raw CSVs as-is: `stg_purchase_order_detail` (8,845 rows × 11 cols), `stg_purchase_order_header` (4,012 × 13), `dim_vendor` (104 × 8)
* `header=False` → Spark assigns generic names `_c0`, `_c1`, …; all columns stay STRING

### Silver — names, types, data quality (`adventure_works.silver`)

* Real column names and types taken from the `CREATE TABLE [Purchasing].…` statements in `instawdb.sql` (single source of truth)
* SQL Server → Databricks type mapping: `int`→INT, `smallint`/`tinyint`→SMALLINT, `money`→DECIMAL(19,4), `datetime`→TIMESTAMP, `Flag`→INT
* **Data-quality rule** on `stg_purchase_order_detail`: drop rows with NULL `ProductID` or negative `LineTotal` (0 rows dropped — source data is clean)

### Gold — `fact_purchases`

* `stg_purchase_order_detail` ⋈ `stg_purchase_order_header` on `PurchaseOrderID`, pulling in `VendorID` and `OrderDate` from the header
* One row per purchase-order line: fact rows (8,845) = detail rows (8,845) — the join does not fan out (`PurchaseOrderID` is unique in the header, verified before joining)

## Business Queries

### Query 1 — Thinnest margin between sales price and purchase price

Averages each price per product first (to avoid many-to-many join distortion), then joins on `ProductID`. Top-5 thinnest margins are **all negative** — these products are sold below their purchase price:

| Product | Avg sales price | Avg purchase price | Margin |
| --- | --- | --- | --- |
| Front Brakes | 63.8760 | 82.8345 | −18.9585 |
| Rear Brakes | 63.9000 | 82.8345 | −18.9345 |
| ML Road Tire | 24.9900 | 39.4583 | −14.4683 |
| HL Mountain Pedal | 48.5686 | 62.9895 | −14.4209 |
| HL Road Pedal | 48.5940 | 62.9895 | −14.3955 |

### Query 2 — Units sold vs. units purchased, by product and month

FULL OUTER JOIN of monthly sales and purchase totals per product; flags months where units sold exceed units purchased (restocking shortfall). The biggest shortfalls are finished accessories that are never purchased from vendors (produced in-house): *Water Bottle – 30 oz.* (up to 673 units/month), *AWC Logo Cap*, *Classic Vest, S*, jerseys and helmets.

## Reproducing

1. Workspace with Unity Catalog; AdventureWorks2022 CSVs (`PurchaseOrderDetail.csv`, `PurchaseOrderHeader.csv`, `Vendor.csv`, sales CSVs) in `/Volumes/adventure_works/default/raw_data`
2. Import `PurchasingStar_Ahmet.ipynb` and **Run All** — it creates the catalog/schemas and tables from scratch

## Lesson 01 (previous exercise)

Sales star (`fact_sales` + `dim_product` + `dim_sales_territory`) built with the same Bronze/Silver/Gold pattern — see `HelloWorld.ipynb` for the team/Git exercise.