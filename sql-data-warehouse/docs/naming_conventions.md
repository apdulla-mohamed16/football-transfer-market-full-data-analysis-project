# **Naming Conventions**

This document outlines the naming conventions used for schemas, tables, views, columns, and other objects in the data warehouse.

## **Table of Contents**

1. [General Principles](#general-principles)
2. [Table Naming Conventions](#table-naming-conventions)
   - [Bronze Rules](#bronze-rules)
   - [Silver Rules](#silver-rules)
   - [Gold Rules](#gold-rules)
3. [Column Naming Conventions](#column-naming-conventions)
   - [Surrogate Keys](#surrogate-keys)
   - [Technical Columns](#technical-columns)
4. [Stored Procedure](#stored-procedure-naming-conventions)
5. [File Naming Conventions](#file-naming-conventions)
---

## **General Principles**

- **Naming Conventions**: Use snake_case, with lowercase letters and underscores (`_`) to separate words.
- **Language**: Use English for all names.
- **Avoid Reserved Words**: Do not use SQL reserved words or keywords as object names (for example `date`, `type`, `name`, `position`, `round`, `aggregate`). When a source column clashes, rename it in Silver (for example `date` becomes `game_date`).

## **Table Naming Conventions**

### **Bronze Rules**
- Table names must match the original CSV file names without renaming.
- This project has **one** source system (Transfermarkt), so the `<sourcesystem>_` prefix used in multi-source projects is **omitted**. If a second source is added later, prefix every table with its source name (for example `tm_transfers`).
- **`<entity>`**
  - `<entity>`: Exact file name of the source table.
  - Example: `transfers` → Transfers of players between clubs.

### **Silver Rules**
- Table names are identical to the Bronze table names.
- **`<entity>`**
  - Example: `players` → Cleaned and typed player data.

### **Gold Rules**
- All names must use meaningful, business-aligned names for tables, starting with the category prefix.
- **`<category>_<entity>`**
  - `<category>`: Describes the role of the object, such as `dim` (dimension), `fact` (fact table) or `report` (report view).
  - `<entity>`: Descriptive name aligned with the business domain (for example `players`, `clubs`, `transfers`).
  - Examples:
    - `dim_players` → Dimension table for player data.
    - `fact_transfers` → Fact table containing transfers.
    - `report_club_transfer_summary` → Report view answering Q1.

#### **Glossary of Category Patterns**

| Pattern   | Meaning                     | Example(s)                                        |
| --------- | --------------------------- | ------------------------------------------------- |
| `dim_`    | Dimension table             | `dim_players`, `dim_clubs`, `dim_date`            |
| `fact_`   | Fact table                  | `fact_transfers`, `fact_appearances`              |
| `report_` | Report view (feeds Tableau) | `report_player_age_value`, `report_club_strategy` |

## **Column Naming Conventions**

### **Surrogate Keys**
- All primary keys in dimension tables must use the suffix `_key`.
- **`<table_name>_key`**
  - `<table_name>`: Refers to the name of the entity the key belongs to.
  - `_key`: A suffix indicating that this column is a surrogate key.
  - Example: `player_key` → Surrogate key in the `dim_players` table.
- Exception: `dim_date` uses a smart key `date_key` in `YYYYMMDD` format.
- Foreign keys in fact tables keep the dimension's key name. When one dimension plays two roles, add a role prefix: `from_club_key` and `to_club_key` both point to `dim_clubs.club_key`.

### **Business Columns**
- Business keys keep the source name with the `_id` suffix (`player_id`, `club_id`).
- Monetary columns end with `_eur`; ratios end with `_ratio`; dates end with `_date`; date keys end with `_date_key`.
- Booleans start with `is_` (for example `is_future_transfer`).

### **Technical Columns**
- All technical columns must start with the prefix `dwh_`, followed by a descriptive name indicating the column's purpose.
- **`dwh_<column_name>`**
  - `dwh`: Prefix exclusively for system-generated metadata.
  - `<column_name>`: Descriptive name indicating the column's purpose.
  - Example: `dwh_create_date` → System-generated column used to store the date when the record was loaded into Silver.
- Bronze tables have **no** technical columns, because `BULK INSERT` maps columns by position and the table must match the file exactly. Load metadata (table, rows, duration, status) is written to `bronze.load_log` instead.

## **Stored Procedure**

- All stored procedures used for loading data must follow the naming pattern:
- **`load_<layer>`**.

  - `<layer>`: Represents the layer being loaded, such as `bronze`, `silver`, or `gold`.
  - Example:
    - `bronze.load_bronze` → Stored procedure for loading data into the Bronze layer.
    - `silver.load_silver` → Stored procedure for loading data into the Silver layer.

## **File Naming Conventions**

| Type              | Pattern                      | Example                                    |
| ----------------- | ---------------------------- | ------------------------------------------ |
| Table definitions | `ddl_<layer>.sql`            | `ddl_bronze.sql`, `ddl_silver.sql`         |
| Load procedures   | `proc_load_<layer>.sql`      | `proc_load_bronze.sql`                     |
| Quality checks    | `quality_checks_<layer>.sql` | `quality_checks_silver.sql`                |
| Gold views        | `ddl_gold.sql`               | `ddl_gold.sql`                             |
| Documents         | `snake_case.md`              | `data_catalog.md`, `naming_conventions.md` |

### **Repository layout**

```
graduation-project/
├── data/raw/                  # source CSV files (never edited)
├── sql/
│   ├── 01_bronze/             # 00_init_database, ddl_bronze, proc_load_bronze, validate_bronze
│   ├── 02_silver/             # ddl_silver, proc_load_silver
│   └── 03_gold/               # ddl_gold
├── tests/                     # quality_checks_silver.sql, quality_checks_gold.sql
├── excel/
├── tableau/
├── docs/                      # naming_conventions, data_catalog, data_architecture.drawio
└── README.md
```
