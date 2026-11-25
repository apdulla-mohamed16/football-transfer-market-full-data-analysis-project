# Data Architecture: Medallion (Bronze / Silver / Gold)

> The diagrams below are written in Mermaid. They render automatically on GitHub and in most Markdown viewers (VS Code with the Mermaid extension, Notion, Obsidian). To get an image for the report, paste a diagram into https://mermaid.live and export PNG or SVG.

## 1. Data source

- **Dataset:** Transfermarkt Dataset (Kaggle: `davidcariboo/player-scores`)
- **Version note:** the maintainer announced that updates are paused and the data is current to **6 July 2026**. If you downloaded the files after that, you hold the latest available snapshot.
- **Download date:** `____-__-__` (fill in, and keep the same files for the whole project)

## 2. End-to-end flow

```mermaid
flowchart LR
    SRC[("Kaggle CSV files<br/>data/raw")]

    subgraph DB["SQL Server (FootballTransfers)"]
        direction LR
        B["BRONZE<br/>raw copy<br/>all columns TEXT"]
        S["SILVER<br/>typed, clean,<br/>scoped to project"]
        G["GOLD<br/>one view per question"]
    end

    X["Excel<br/>validation + budget model"]
    T["Tableau<br/>3 dashboards"]
    R["Report + Presentation<br/>+ Recommendations"]

    SRC -->|load as is| B
    B -->|clean + cast + filter| S
    S -->|join + aggregate| G
    G -->|export / connect| X
    G -->|export / connect| T
    X --> R
    T --> R
```

## 3. Layer contracts

|                    | Bronze                                       | Silver                          | Gold                          |
| ------------------ | -------------------------------------------- | ------------------------------- | ----------------------------- |
| Purpose            | Safety copy of the source                    | Trustworthy clean data          | Answers to business questions |
| Column types       | all NVARCHAR(MAX)                            | real types (DATE, NUMERIC, INT) | real types                    |
| Filtering          | none                                         | project scope applied here      | already scoped                |
| Who reads it       | Silver only                                  | Gold only                       | Excel, Tableau                |
| Allowed to change? | never                                        | when cleaning rules change      | when questions change         |
| Check              | row count = file row count (bronze.load_log) | `data_quality_log`              | numbers match Excel           |

## 4. Tables per layer

```mermaid
flowchart LR
    subgraph BRONZE["BRONZE (raw)"]
        direction TB
        b1[bronze.players]
        b2[bronze.player_valuations]
        b3[bronze.transfers]
        b4[bronze.clubs]
        b5[bronze.competitions]
        b6[bronze.games]
        b7[bronze.appearances]
        bl[bronze.load_log]
    end

    subgraph SILVER["SILVER (clean)"]
        direction TB
        s1[silver.players]
        s2[silver.player_valuations]
        s3[silver.transfers]
        s4[silver.clubs]
        s5[silver.competitions]
        s6[silver.games]
        s7[silver.appearances]
        sl[silver.data_quality_log]
    end

    subgraph GOLD["GOLD (analysis)"]
        direction TB
        g1[gold.club_transfer_summary]
        g2[gold.player_age_value]
        g3[gold.transfer_value_vs_output]
        g4[gold.sell_vs_market_value]
        g5[gold.youth_development_profit]
        g6[gold.club_profile]
    end

    b1 --> s1
    b2 --> s2
    b3 --> s3
    b4 --> s4
    b5 --> s5
    b6 --> s6
    b7 --> s7

    s3 --> g1
    s4 --> g1
    s1 --> g2
    s2 --> g2
    s3 --> g3
    s7 --> g3
    s1 --> g3
    s3 --> g4
    s1 --> g4
    s5 --> g4
    s3 --> g5
    s1 --> g5
    s4 --> g6
    s3 --> g6
    s1 --> g6
```

## 5. Relationships between tables (ERD)

Based on the relationships documented by the dataset maintainer. The `transfers` links are the standard ones, but **check the exact column names against your own file** (for example `from_club_id` and `to_club_id`) before writing joins.

```mermaid
erDiagram
    competitions ||--o{ games : "competition_id"
    competitions ||--o{ clubs : "domestic_competition_id"
    clubs ||--o{ players : "current_club_id"
    clubs ||--o{ games : "home_club_id / away_club_id"
    clubs ||--o{ transfers : "from_club_id / to_club_id"
    players ||--o{ appearances : "player_id"
    players ||--o{ player_valuations : "player_id"
    players ||--o{ transfers : "player_id"
    games ||--o{ appearances : "game_id"
```

## 6. What happens inside each layer

### Bronze

```mermaid
flowchart TD
    A[CSV file] --> B["BULK INSERT<br/>all columns NVARCHAR(MAX)"]
    B --> C["Write row count + duration<br/>to bronze.load_log"]
    C --> D[(bronze table)]
    D --> E{"Rows in file<br/>= rows in table?"}
    E -->|yes| F["Profile the data<br/>validate_bronze.sql"]
    E -->|no| G["Stop and fix the load"]
```

### Silver

```mermaid
flowchart TD
    A[(bronze table)] --> B["Cast types<br/>TEXT to DATE / NUMERIC / INT"]
    B --> C["Handle NULLs and empty values<br/>documented rule per column"]
    C --> D["Remove duplicates<br/>ROW_NUMBER"]
    D --> E["Standardize text<br/>TRIM, case, names"]
    E --> F["Add flags and fields<br/>season, age_at_transfer,<br/>is_loan, is_free_transfer"]
    F --> G["Apply scope<br/>5 leagues + 2015 onward"]
    G --> H["Check keys<br/>club_id exists in clubs"]
    H --> I[(silver table)]
    D --> L[("data_quality_log<br/>rows in / out / removed + reason")]
    G --> L
    H --> L
```

### Gold

```mermaid
flowchart TD
    A[(silver tables)] --> B["Join<br/>players + transfers + clubs"]
    B --> C["Aggregate<br/>SUM, AVG per club / season / age group"]
    C --> D["Window functions<br/>RANK, LAG"]
    D --> E["Business columns<br/>net_profit_eur, price_to_value_ratio"]
    E --> F[("gold view<br/>grain documented in a comment")]
```

## 7. From questions to dashboards

```mermaid
flowchart LR
    Q1["Q1 Spending and profit by club"] --> G1[club_transfer_summary]
    Q4["Q4 Selling price vs market value"] --> G4[sell_vs_market_value]
    Q2["Q2 Age at peak value"] --> G2[player_age_value]
    Q3["Q3 Best and worst deals"] --> G3[transfer_value_vs_output]
    Q5["Q5 Youth development profit"] --> G5[youth_development_profit]
    Q6["Q6 Best transfer strategy"] --> G6[club_profile]

    G1 --> D1["Dashboard 1<br/>Transfer market"]
    G4 --> D1
    G2 --> D2["Dashboard 2<br/>Age, value, deals"]
    G3 --> D2
    G5 --> D3["Dashboard 3<br/>Clubs and strategy"]
    G6 --> D3

    D1 --> REC["Final recommendations"]
    D2 --> REC
    D3 --> REC
```

## 8. How Tableau connects

```mermaid
flowchart TD
    G[("Gold views")] --> Q{"Tableau edition?"}
    Q -->|"Desktop (student license)"| D["Direct database connection<br/>use Extract, not Live"]
    Q -->|"Public (free)"| E["Export Gold views to CSV<br/>data/gold_exports"]
    E --> F["Connect to the CSV files<br/>use Extract"]
    D --> T["Dashboards"]
    F --> T
    T --> P["Publish to Tableau Public<br/>link goes in report + GitHub"]
```

Tableau reads **Gold only**. If you need a new calculation, create it as a Gold view in SQL instead of using Custom SQL inside Tableau.

## 9. Project structure

```
graduation-project/
├── data/
│   ├── raw/                     # the CSV files (never edited)
│   └── gold_exports/            # Gold views exported for Tableau Public
├── sql/
│   ├── 00_explore/
│   ├── 01_bronze/
│   │   ├── 00_init_database.sql
│   │   ├── ddl_bronze.sql
│   │   ├── proc_load_bronze.sql
│   │   ├── validate_bronze.sql
│   │   └── bronze_optional_tables.sql
│   ├── 02_silver/
│   ├── 03_gold/
│   └── 04_analysis/
├── excel/
├── tableau/
├── docs/
│   ├── 01_data_architecture.md
│   ├── erd/
│   └── data_dictionary/
└── README.md
```

## 10. Bronze profiling checklist (answer these before writing Silver)

- Date range of `transfers`, `player_valuations`, `games`.
- Which leagues exist in `competitions` (decides the project scope).
- What `0`, empty, and NULL mean in the transfer fee column (loan, free transfer, or unknown).
- How many `club_id` values in `transfers` do not exist in `clubs`.
- Whether a player can appear twice for the same transfer (duplicates).
- Empty cells are loaded as NULL (KEEPNULLS), but still use `NULLIF(LTRIM(RTRIM(col)), '')` in Silver to be safe.
- Text values with stray spaces exist (for example `city_of_birth` = `Opole  `): trim in Silver.
- `transfer_date` / `transfer_season` contain future dates (for example 2030-06-30, season 25/26): decide how to treat them.
- `clubs.net_transfer_record` is text such as `+€5.90m` and `€-25.00m`: parse it or ignore it and compute from `transfers`.
- Date columns are text; `players.date_of_birth` carries a time part (`1978-06-09 00:00:00`).
