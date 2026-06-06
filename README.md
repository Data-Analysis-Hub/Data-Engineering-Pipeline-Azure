# 🏗️ Azure Retail Data Pipeline — Medallion Architecture

End-to-end Data Engineering pipeline built on Azure, processing retail sales data from raw ingestion to analytical dashboard using a Bronze → Silver → Gold architecture.

---

## 🏛️ Architecture Overview

```
Azure SQL DB ──┐
               ├──► Azure Data Factory ──► ADLS Gen2 (Bronze) ──► Databricks (Silver) ──► Databricks (Gold) ──► Power BI
GitHub API ────┘
```

| Layer | Description |
|-------|-------------|
| **Bronze** | Raw Parquet files ingested as-is from sources |
| **Silver** | Cleaned, typed, joined, enriched data — Delta format |
| **Gold** | Business aggregations ready for reporting |

---

## ⚙️ Tech Stack

- **Azure SQL Database** — Source for Products, Stores, Transactions
- **GitHub API (JSON)** — Source for Customers data
- **Azure Data Factory** — Orchestration & ingestion (4 Copy Activities)
- **Azure Data Lake Storage Gen2** — Medallion storage (`hkhiar-datalake`)
- **Databricks Community Edition** — PySpark transformations (Silver & Gold)
- **Delta Lake** — Silver layer storage format
- **Power BI Desktop** — Final dashboard

---

## 📁 Project Structure

```
├── notebooks/
│   ├── ADLS_Bronze.py       # Load Bronze → clean → write Silver (Delta)
│   └── ADLS_Silver.py       # Load Silver → aggregate → write Gold
├── dashboard/
│   └── SalesAnalysis.pbix   # Power BI dashboard
└── README.md
```

---

## 🔄 Pipeline Phases

### Phase 1 — Source Setup
- Azure SQL tables: `Products`, `Stores`, `Transactions`
- External JSON source via GitHub API: `Customers`

### Phase 2 — Bronze (ADF Ingestion)
4 Copy Activities copying data to ADLS Gen2 in **Parquet** format:
```
SQL → bronze/transaction/
SQL → bronze/product/
SQL → bronze/store/
API → bronze/customer/
```

### Phase 3 — Silver (Databricks)
- Type casting (`IntegerType`, `FloatType`, `DateType`)
- Deduplication & null handling
- 4-way join: transactions ⋈ products ⋈ stores ⋈ customers
- Computed column: `total_amount = quantity × price`
- Written as **Delta** to `Silver/`

### Phase 4 — Gold (Databricks)
Aggregations by `product_id`, `product_name`, `category`, `transaction_date`:
- `total_quantity` — units sold
- `total_revenue` — revenue (DH)

### Phase 5 — Power BI Dashboard
Gold exported to CSV → imported into Power BI with a generated Date Table and Measure Table.

---

## 📊 Dashboard

**Sales Analysis — Office Supplies**

| KPI | Value |
|-----|-------|
| Total Sales | 35.26K DH |
| Total Quantity | 101 units |
| Total Products | 8 |
| Total Categories | 4 |

Visuals: Revenue by Date · Quantity by Date · Sales by Category (pie) · Sales by Product (100% bar) · Quantity by Category (donut)

---

## 🧮 DAX Measures

```dax
Total Sales     = SUM(gold_layer_transformations[total_revenue])
Total Quantity  = SUM(gold_layer_transformations[total_quantity])
Total Products  = DISTINCTCOUNT(gold_layer_transformations[product_name])
Total Categories = DISTINCTCOUNT(gold_layer_transformations[category])

Date Table =
VAR MinDate = MIN(gold_layer_transformations[transaction_date])
VAR MaxDate = MAX(gold_layer_transformations[transaction_date])
RETURN
ADDCOLUMNS(
    CALENDAR(MinDate, MaxDate),
    "Year",      YEAR([Date]),
    "MonthNum",  MONTH([Date]),
    "MonthName", FORMAT([Date], "MMMM")
)
```

---

## 🔐 Secrets Management

Storage account key managed via Databricks secrets:

```python
storage_account_key = dbutils.secrets.get(scope="azure_access_key", key="storage_key")
```

Paths use `abfss://` protocol — no mount points:

```python
base_path = f"abfss://{container_name}@{storage_account_name}.dfs.core.windows.net/"
```

---

## 🚀 How to Run

1. Provision Azure SQL DB and insert source data
2. Create ADLS Gen2 storage account with Hierarchical Namespace enabled
3. Set up Databricks secret scope with your storage key
4. Trigger ADF pipeline — verify Parquet files in Bronze
5. Run `ADLS_Bronze.py` notebook — Silver Delta table is written
6. Run `ADLS_Silver.py` notebook — Gold aggregations are written
7. Export Gold as CSV and open `SalesAnalysis.pbix` in Power BI Desktop