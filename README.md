# AdventureWorks Lakehouse: 
# Bronze → Silver → Gold on Databricks

An end-to-end data engineering project that builds a **medallion architecture** (Bronze / Silver / Gold) on Databricks using **PySpark, Spark SQL and Delta Lake**, turning raw AdventureWorks sales CSVs into business-ready customer analytics.

## Architecture

```
7 CSV files ──► BRONZE (raw Delta tables) ──► SILVER (cleaned + joined) ──► GOLD (business metrics)
                 workspace.bronze              workspace.silver              workspace.gold
```

| Layer | Tables | Purpose |
|---|---|---|
| Bronze | FactSales, FactSalesOrderDetails, FactSalesTerritoryDetails, DimCustomer, DimReseller, DimProduct, DimDate | Raw data, loaded as-is |
| Silver | FactSalesCombined (partitioned by `SalesTerritoryKey`), FactCustomerSegment | Cleaned, typed, joined, segmented |
| Gold | sales_analytics (partitioned by `SalesTerritoryKey`), customer_ltv | Customer lifetime value and loyalty tiers |

## Dataset

| Item | Value |
|---|---|
| Source tables | 7 |
| Sales lines (FactSales) | 121,253 |
| Customers | 18,485 (includes placeholder key `-1` for reseller sales) |
| Sales territories | 10 in sales data |

## What each layer does

**Bronze** (`01_Extract_AdventureWorks.py`)
- Uploads the 7 CSVs as Delta tables into the `bronze` schema, with no changes to the data.
- Lists all tables and validates row counts.

**Silver** (`02_Silver_Transformations.py`)
- Replaces spaces in column names with `_`, with a duplicate-name check.
- Casts 5 money columns (stored as text such as `$2,024.99`) to `double`, with failed-cast validation.
- Handles nulls: unshipped orders keep their rows (null `ShipDateKey` replaced with sentinel `0`), blank customer text becomes `Unknown`, and empty customer names are checked.
- Builds 5 joins on a FactSales-grain base (left joins, so no sales row is lost), then combines them on `SalesOrderLineKey` into one table with one row per sales line.
- Calculates total revenue per customer and segments customers as High_Value (> 50,000), Medium-Value (> 20,000) or Low-Value.
- Validates: row counts (121,253 and 18,485), duplicates, null keys, negative revenue and segment boundaries.

**Gold** (`03_Gold_Layer.py`)
- Saves `sales_analytics` partitioned by `SalesTerritoryKey` for regional queries.
- Builds `customer_ltv`: TotalSpend, TotalPurchases (distinct orders), AvgPurchaseValue, LoyaltyScore (Platinum / Gold / Silver / Bronze) and new vs return customer.
- Validates row counts, negative sales, metric accuracy and reconciles total spend with source sales.

## Data quality issues found and handled

- Money columns stored as strings with `$` and `,`, so a direct cast would fail.
- 2,113 unshipped orders with null `ShipDateKey`, kept rather than dropped.
- Placeholder `-1` customer and reseller rows representing sales without a named customer. 

## Tech stack

Databricks (Serverless), PySpark, Spark SQL, Delta Lake, Unity Catalog.

 
```

## How to run

1. Create schemas `bronze`, `silver`, `gold` in your catalog (default: `workspace`).
2. Upload the 7 AdventureWorks CSVs into `bronze` using the table names above.
3. Import the notebooks from `notebooks/` and run them in order: 01, 02, 03.

## Key learnings

- Keeping raw data untouched in Bronze makes every downstream step traceable and repeatable.
- Joining dimensions onto a fact table with left joins and checking row counts after each join prevents silent row loss or duplication.
- Partitioning by territory key speeds up regional queries.
- Validation checks (counts, nulls, reconciliation) are what make a pipeline trustworthy.
