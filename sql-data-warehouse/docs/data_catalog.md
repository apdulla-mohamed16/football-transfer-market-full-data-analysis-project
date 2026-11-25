# Data Catalog for Gold Layer

## Overview
The Gold Layer is the business-level data representation, structured to support analytical and reporting use cases. It consists of **dimension tables**, **fact tables** and **report views** for the transfer-market analysis. The model is a star schema with three fact tables that share the same dimensions (a constellation).

Source: Transfermarkt Dataset (Kaggle: `davidcariboo/player-scores`). Gold objects are **views** built on the Silver tables; there is no load step.

**Conventions used in this catalog**
- Surrogate keys end with `_key`. Every dimension (except `dim_date`) has an *Unknown* member with key `-1`, so a fact row never loses its club or player when a business key is missing.
- Monetary values are in euros (`_eur`).
- `dim_` = dimension, `fact_` = fact table, `report_` = report view for Tableau (see `naming_conventions.md`).

---

### 1. **gold.dim_players**
- **Purpose:** Player attributes used to slice every fact table.
- **Grain:** One row per player.
- **Columns:**

| Column Name              | Data Type     | Key | Description                                                             |
| ------------------------ | ------------- | --- | ----------------------------------------------------------------------- |
| player_key               | INT           | PK  | Surrogate key uniquely identifying each player record in the dimension. |
| player_id                | INT           |     | Transfermarkt player identifier (business key).                         |
| player_name              | NVARCHAR(200) |     | Full name of the player.                                                |
| first_name               | NVARCHAR(100) |     | First name of the player.                                               |
| last_name                | NVARCHAR(100) |     | Last name of the player.                                                |
| date_of_birth            | DATE          |     | Date of birth (YYYY-MM-DD).                                             |
| country_of_birth         | NVARCHAR(100) |     | Country where the player was born.                                      |
| country_of_citizenship   | NVARCHAR(100) |     | Main citizenship of the player.                                         |
| player_position          | NVARCHAR(50)  |     | Broad position: Goalkeeper, Defender, Midfield, Attack.                 |
| player_sub_position      | NVARCHAR(50)  |     | Detailed position (e.g. Centre-Back, Right Winger).                     |
| foot                     | NVARCHAR(20)  |     | Preferred foot: right, left or both.                                    |
| height_in_cm             | INT           |     | Height of the player in centimetres.                                    |
| market_value_eur         | DECIMAL(18,2) |     | Latest market value in euros available in the dataset.                  |
| highest_market_value_eur | DECIMAL(18,2) |     | Highest market value reached in the player's career, in euros.          |

---

### 2. **gold.dim_clubs**
- **Purpose:** Club attributes and the domestic league the club plays in.
- **Grain:** One row per club, plus one 'Unknown' member (club_key = -1).
- **Columns:**

| Column Name          | Data Type     | Key | Description                                                                       |
| -------------------- | ------------- | --- | --------------------------------------------------------------------------------- |
| club_key             | INT           | PK  | Surrogate key uniquely identifying each club record. -1 = unknown / without club. |
| club_id              | INT           |     | Transfermarkt club identifier (business key).                                     |
| club_name            | NVARCHAR(200) |     | Name of the club.                                                                 |
| domestic_league_id   | NVARCHAR(10)  |     | Identifier of the club's domestic league (e.g. GB1).                              |
| domestic_league_name | NVARCHAR(200) |     | Name of the club's domestic league.                                               |
| country_name         | NVARCHAR(100) |     | Country of the club's domestic league.                                            |
| squad_size           | INT           |     | Number of players in the squad (latest snapshot).                                 |
| average_age          | DECIMAL(4,1)  |     | Average age of the squad (latest snapshot).                                       |
| stadium_name         | NVARCHAR(200) |     | Name of the home stadium.                                                         |
| stadium_seats        | INT           |     | Capacity of the home stadium.                                                     |

---

### 3. **gold.dim_competitions**
- **Purpose:** Competitions in which matches are played.
- **Grain:** One row per competition.
- **Columns:**

| Column Name          | Data Type     | Key | Description                                                  |
| -------------------- | ------------- | --- | ------------------------------------------------------------ |
| competition_key      | INT           | PK  | Surrogate key uniquely identifying each competition record.  |
| competition_id       | NVARCHAR(10)  |     | Transfermarkt competition code (business key, e.g. GB1).     |
| competition_name     | NVARCHAR(200) |     | Name of the competition.                                     |
| competition_type     | NVARCHAR(100) |     | Type, e.g. domestic_league, domestic_cup, international_cup. |
| competition_sub_type | NVARCHAR(100) |     | Sub type, e.g. first_tier.                                   |
| country_name         | NVARCHAR(100) |     | Country of the competition (NULL for international ones).    |
| confederation        | NVARCHAR(50)  |     | Confederation, e.g. europa.                                  |

---

### 4. **gold.dim_date**
- **Purpose:** Calendar dimension with the football season label.
- **Grain:** One row per calendar day.
- **Columns:**

| Column Name  | Data Type    | Key | Description                                                       |
| ------------ | ------------ | --- | ----------------------------------------------------------------- |
| date_key     | INT          | PK  | Smart key in YYYYMMDD format (e.g. 20240701).                     |
| full_date    | DATE         |     | The calendar date.                                                |
| year         | INT          |     | Calendar year.                                                    |
| quarter      | INT          |     | Calendar quarter (1-4).                                           |
| month        | INT          |     | Month number (1-12).                                              |
| month_name   | NVARCHAR(20) |     | Month name (e.g. July).                                           |
| season_label | NVARCHAR(9)  |     | Football season the date belongs to (July to June), e.g. 2024/25. |

---

### 5. **gold.fact_transfers**
- **Purpose:** Player transfers between clubs: the base for spending, income and profit analysis (Q1, Q3, Q4, Q5, Q6).
- **Grain:** One row per transfer (player, date, from club, to club).
- **Columns:**

| Column Name        | Data Type     | Key | Description                                                            |
| ------------------ | ------------- | --- | ---------------------------------------------------------------------- |
| player_key         | INT           | FK  | Links to dim_players.player_key.                                       |
| from_club_key      | INT           | FK  | Selling club. Links to dim_clubs.club_key.                             |
| to_club_key        | INT           | FK  | Buying club. Links to dim_clubs.club_key.                              |
| transfer_date_key  | INT           | FK  | Links to dim_date.date_key.                                            |
| transfer_season    | NVARCHAR(10)  |     | Season label as in the source (e.g. 24/25).                            |
| transfer_fee_eur   | DECIMAL(18,2) |     | Transfer fee in euros. NULL when unknown.                              |
| market_value_eur   | DECIMAL(18,2) |     | Market value of the player at the time of transfer, in euros.          |
| fee_type           | NVARCHAR(20)  |     | paid, zero or unknown (see silver.transfers).                          |
| is_future_transfer | BIT           |     | 1 when the transfer date is after the dataset cut-off date.            |
| age_at_transfer    | INT           |     | Age of the player on the transfer date, in whole years.                |
| fee_to_value_ratio | DECIMAL(10,2) |     | transfer_fee_eur / market_value_eur. NULL when the value is NULL or 0. |

---

### 6. **gold.fact_player_valuations**
- **Purpose:** History of market values: the base for the age-versus-value analysis (Q2).
- **Grain:** One row per player per valuation date.
- **Columns:**

| Column Name        | Data Type     | Key | Description                                                        |
| ------------------ | ------------- | --- | ------------------------------------------------------------------ |
| player_key         | INT           | FK  | Links to dim_players.player_key.                                   |
| club_key           | INT           | FK  | Club of the player at valuation time. Links to dim_clubs.club_key. |
| valuation_date_key | INT           | FK  | Links to dim_date.date_key.                                        |
| market_value_eur   | DECIMAL(18,2) |     | Market value in euros on that date.                                |
| age_at_valuation   | INT           |     | Age of the player on the valuation date, in whole years.           |

---

### 7. **gold.fact_appearances**
- **Purpose:** Player performance per match: the base for output versus price (Q3).
- **Grain:** One row per player per match.
- **Columns:**

| Column Name     | Data Type    | Key | Description                                              |
| --------------- | ------------ | --- | -------------------------------------------------------- |
| appearance_id   | NVARCHAR(30) |     | Source identifier (game_id_player_id).                   |
| player_key      | INT          | FK  | Links to dim_players.player_key.                         |
| club_key        | INT          | FK  | Club the player played for. Links to dim_clubs.club_key. |
| competition_key | INT          | FK  | Links to dim_competitions.competition_key.               |
| game_date_key   | INT          | FK  | Links to dim_date.date_key.                              |
| game_id         | INT          |     | Match identifier (degenerate dimension).                 |
| minutes_played  | INT          |     | Minutes played in the match.                             |
| goals           | INT          |     | Goals scored.                                            |
| assists         | INT          |     | Assists made.                                            |
| yellow_cards    | INT          |     | Yellow cards received.                                   |
| red_cards       | INT          |     | Red cards received.                                      |

---

### 8. **Report views (for Tableau)**
- **Purpose:** One view per project question. Each view is documented with its grain in a comment at the top of its script. Columns are defined when the views are built in the Gold step.

| View                                 | Question | Grain                                                         | Built from                                    |
| ------------------------------------ | -------- | ------------------------------------------------------------- | --------------------------------------------- |
| gold.report_club_transfer_summary    | Q1       | One row per club per season.                                  | fact_transfers, dim_clubs, dim_date           |
| gold.report_player_age_value         | Q2       | One row per age and position.                                 | fact_player_valuations, dim_players           |
| gold.report_transfer_value_vs_output | Q3       | One row per transfer with output per 90 minutes.              | fact_transfers, fact_appearances, dim_players |
| gold.report_sell_vs_market_value     | Q4       | One row per league and position.                              | fact_transfers, dim_clubs, dim_players        |
| gold.report_youth_development        | Q5       | One row per club (profit from selling young players).         | fact_transfers, dim_clubs, dim_players        |
| gold.report_club_strategy            | Q6       | One row per club per strategy (ready stars vs young talents). | fact_transfers, dim_clubs, dim_players        |

**Questions:** Q1 spending and profit by club, Q2 age at peak value, Q3 best and worst deals, Q4 selling price versus market value, Q5 youth development profit, Q6 best transfer strategy.
