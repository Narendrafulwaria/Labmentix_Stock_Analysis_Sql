# Architecture: Equity History in MySQL

This design supports the goal in `problemstatement.md`: load six daily equity CSVs into MySQL and answer return, sector, volatility, liquidity, delivery, and extreme-day questions for 1 January 2015 through 31 July 2018.

## Overview

The system is a single MySQL database with three layers.

1. **Source.** Six CSV files in this folder, one company each, newest date first.
2. **Staging.** One raw table that accepts the CSV text as loaded, including the original date string and the source file name.
3. **Analysis model.** A company dimension and a daily price fact table. All analysis queries read only these two tables.

```text
CSV files
   |
   v
stg_daily_prices          (text as loaded, one row per source line)
   |
   v
companies                 (6 rows: name, sector, source file)
daily_prices              (889 rows x 6 companies, typed columns)
   |
   v
analysis queries          (one saved query per problem-statement question)
   |
   v
written findings          (figures taken from those result sets)
```

No application server, charts, or external market feed. MySQL is both the store and the analysis engine.

## Schema

Database name: `equity_analysis`.

### `companies`

One row per listed company. Sector is stored here so queries do not hard-code file names.

| Column | Type | Role |
| --- | --- | --- |
| `company_id` | `SMALLINT UNSIGNED` PK, auto-increment | Surrogate key |
| `company_name` | `VARCHAR(64)` NOT NULL, unique | Display name |
| `sector` | `VARCHAR(16)` NOT NULL | `Auto` or `IT` |
| `source_file` | `VARCHAR(64)` NOT NULL | CSV file name |

Seed rows:

| company_name | sector | source_file |
| --- | --- | --- |
| Bajaj Auto | Auto | `Bajaj Auto.csv` |
| Eicher Motors | Auto | `Eicher Motors.csv` |
| Hero Motocorp | Auto | `Hero Motocorp.csv` |
| TVS Motors | Auto | `TVS Motors.csv` |
| TCS | IT | `TCS.csv` |
| Infosys | IT | `Infosys.csv` |

### `stg_daily_prices`

Landing table. Every CSV column stays as text so a bad value fails in the transform step, where it can be counted, rather than during the file load.

| Column | Type |
| --- | --- |
| `source_file` | `VARCHAR(64)` |
| `date_text` | `VARCHAR(32)` |
| `open_price` | `VARCHAR(32)` |
| `high_price` | `VARCHAR(32)` |
| `low_price` | `VARCHAR(32)` |
| `close_price` | `VARCHAR(32)` |
| `wap` | `VARCHAR(64)` |
| `shares` | `VARCHAR(32)` |
| `trades` | `VARCHAR(32)` |
| `turnover` | `VARCHAR(32)` |
| `deliverable_qty` | `VARCHAR(32)` |
| `delivery_pct` | `VARCHAR(32)` |
| `spread_high_low` | `VARCHAR(32)` |
| `spread_close_open` | `VARCHAR(32)` |

### `daily_prices`

One row per company per trading day. This is the only table the analysis queries join.

| Column | Type | Source column |
| --- | --- | --- |
| `price_id` | `INT UNSIGNED` PK, auto-increment | — |
| `company_id` | `SMALLINT UNSIGNED` NOT NULL, FK → `companies` | from `source_file` |
| `trade_date` | `DATE` NOT NULL | `Date`, parsed with `%d-%M-%Y` |
| `open_price` | `DECIMAL(12,2)` NOT NULL | `Open Price` |
| `high_price` | `DECIMAL(12,2)` NOT NULL | `High Price` |
| `low_price` | `DECIMAL(12,2)` NOT NULL | `Low Price` |
| `close_price` | `DECIMAL(12,2)` NOT NULL | `Close Price` |
| `wap` | `DECIMAL(20,6)` NOT NULL | `WAP` |
| `shares_traded` | `BIGINT UNSIGNED` NOT NULL | `No.of Shares` |
| `trade_count` | `INT UNSIGNED` NOT NULL | `No. of Trades` |
| `turnover` | `DECIMAL(18,2)` NOT NULL | `Total Turnover (Rs.)` |
| `deliverable_qty` | `BIGINT UNSIGNED` NOT NULL | `Deliverable Quantity` |
| `delivery_pct` | `DECIMAL(6,2)` NOT NULL | `% Deli. Qty to Traded Qty` |
| `spread_high_low` | `DECIMAL(12,2)` NOT NULL | `Spread High-Low` |
| `spread_close_open` | `DECIMAL(12,2)` NOT NULL | `Spread Close-Open` |

Constraints:

- `UNIQUE (company_id, trade_date)` so a company cannot have two rows on the same day.
- `CHECK` style rules enforced in the load query: `high_price >= low_price`, and `high_price` is at least the open and close.
- Index on `(trade_date)` for yearly filters. The unique key already covers company-then-date lookups.

Prices stay as recorded. The load does not adjust for splits, bonuses, or dividends.

`DECIMAL(12,2)` covers the observed price range (TVS near 270 through Eicher near 29,000). `wap` keeps six decimal places. The longer fractional tail in the CSV is rounded at load.

## Derived measures

These are computed in queries, not stored.

| Measure | Definition |
| --- | --- |
| Total return | `(last close - first close) / first close`, using the earliest and latest `trade_date` per company |
| Year return | Same formula inside each calendar year, from that year’s first close to its last close. 2018 is year-to-date through 31 July 2018 |
| Daily range % | `spread_high_low / close_price` |
| Day-to-day move | `(close - previous close) / previous close`, previous close from `LAG(close_price)` partitioned by `company_id` ordered by `trade_date` |
| Sector averages | Average of the company-level total returns, and average of each company’s mean daily turnover, grouped by `sector` |

Window functions require MySQL 8.0 or later. That version is the baseline for this design.

## Query map

Each problem-statement question is one saved script under `sql/`. Scripts read `companies` and `daily_prices` only.

| Script | Question |
| --- | --- |
| `01_total_returns.sql` | Rank companies by full-period close-to-close return |
| `02_yearly_returns.sql` | Return by company and year, plus the leader each year |
| `03_sector_comparison.sql` | Auto vs IT on average total return and average daily turnover |
| `04_volatility.sql` | Typical daily range and the size of day-to-day close moves |
| `05_liquidity.sql` | Average shares, trade count, and turnover, set beside average daily range |
| `06_delivery.sql` | Average delivery percentage by company |
| `07_extremes.sql` | Highest close, lowest close, largest up day, largest down day |

A short `00_validate.sql` checks the success criteria before any analysis query is treated as final: 889 rows per company, no duplicate `(company_id, trade_date)`, and `MIN(trade_date) = 2015-01-01`, `MAX(trade_date) = 2018-07-31`.

## Data rules

- CSV order is newest-first. After `trade_date` is a real `DATE`, sort order comes from the column, not from the file.
- Company identity comes from the file name, matched to `companies.source_file`.
- Empty numeric fields, if any appear, are rejected by the transform so the row count check fails visibly.
- Turnover and delivery describe the recorded session. They are not adjusted to a common share count across companies, so liquidity comparisons are in rupees and in shares as published, not in free-float terms.

## What this design leaves out

Live prices, forecasts, recommendations, corporate-action adjustments, and any UI. Adding an adjustment later means a new column or a new table, not a change to the grain of `daily_prices`.
