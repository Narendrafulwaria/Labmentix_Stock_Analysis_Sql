# Equity History in MySQL

Daily price analysis for six Indian listed companies, from **1 January 2015** through **31 July 2018**. The series is loaded into MySQL and compared with SQL on return, sector, volatility, liquidity, delivery, and extreme days.

Prices stay as recorded in the source files. They are not adjusted for splits, bonuses, or dividends.

## Companies

| File | Company | Sector | Trading days |
| --- | --- | --- | --- |
| `Bajaj Auto.csv` | Bajaj Auto | Auto | 889 |
| `Eicher Motors.csv` | Eicher Motors | Auto | 889 |
| `Hero Motocorp.csv` | Hero Motocorp | Auto | 889 |
| `TVS Motors.csv` | TVS Motors | Auto | 889 |
| `TCS.csv` | TCS | IT | 889 |
| `Infosys.csv` | Infosys | IT | 889 |

Each CSV is newest-date first and has open, high, low, close, weighted average price, shares, trades, turnover, deliverable quantity, delivery percentage, and the high-low and close-open spreads. Six source rows have a blank deliverable quantity and delivery percentage. Those values are stored as `NULL`.

## What the analysis answers

1. Full-period close-to-close return, ranked from the largest gain to the largest loss.
2. Return by calendar year, and the leader of each year. 2018 is year-to-date through 31 July 2018.
3. Auto versus IT on average total return and average daily turnover.
4. Typical daily range and the size of day-to-day close moves.
5. Average shares, trade count, and turnover, set beside the daily range.
6. Delivery percentage, both as a simple average and weighted by shares traded.
7. Highest close, lowest close, and the largest up and down close-to-close days.

## Repository

| File | Role |
| --- | --- |
| `problemstatement.md` | Goal, questions, and success criteria |
| `architecture.md` | Database design: staging, `companies`, and `daily_prices` |
| `implementation.md` | Build steps. Each task ends in a query, with the expected result in the comment above it |
| `equity_analysis_report.docx` | Written findings taken from those result sets |
| `*.csv` | Source history, one company per file |

## Database

MySQL **8.0 or later**. Database name: `equity_analysis`.

```text
CSV files
   |
   v
stg_daily_prices     text as loaded, plus the source file name
   |
   v
companies            6 rows: name, sector, source file
daily_prices         5,334 typed rows (889 x 6), one row per company per day
   |
   v
analysis queries
```

`daily_prices` is unique on `(company_id, trade_date)`. Dates such as `31-July-2018` are parsed with `STR_TO_DATE(..., '%d-%M-%Y')`. Return, volatility, and day-move figures are calculated in queries, not stored. Window functions (`LAG`, `ROW_NUMBER`) need MySQL 8.0.

## How to load it

Run the tasks in `implementation.md` in order, from this folder, against a local MySQL server.

1. **Task 1** creates `equity_analysis` and inserts the six companies.
2. **Task 2** creates `stg_daily_prices` and loads each CSV with `LOAD DATA LOCAL INFILE`. The client must be started with `local-infile` enabled (`mysql --local-infile=1`). Change the file path in each statement to match your clone of this repository. `source_file` must match `companies.source_file` exactly.
3. **Task 3** casts staging into `daily_prices`. Empty delivery fields become `NULL`. A correct load is **5,334** rows and **6** null delivery rows.
4. **Task 4** checks 889 rows per company, the date span `2015-01-01` to `2018-07-31`, no duplicate company-dates, no invalid high-low ranges, and the six blank delivery days.
5. **Tasks 5–11** are the analysis queries. Compare each result with the comment block directly above that query.
6. **Task 12** points at the written report.

If `secure_file_priv` blocks `LOAD DATA`, load the same rows into `stg_daily_prices` another way and continue from Task 3. To reload a corrected CSV, follow the four steps at the end of `implementation.md`.

## Headline results

These figures are the recorded price change, not a split-adjusted return.

| Company | Sector | Total return |
| --- | --- | --- |
| TVS Motors | Auto | 86.91% |
| Eicher Motors | Auto | 82.56% |
| Bajaj Auto | Auto | 10.05% |
| Hero Motocorp | Auto | 6.00% |
| TCS | IT | −23.82% |
| Infosys | IT | −30.91% |

Year leaders: Eicher Motors in 2015 (10.60%), TVS Motors in 2016 (25.62%) and 2017 (110.65%), Infosys in 2018 (31.94%). Across the four auto names the average total return is 46.38%. Across the two IT names it is −27.37%. The IT decline is dominated by one session in each file where the recorded close roughly halved and stayed there: Infosys on 15 June 2015 and TCS on 31 May 2018.

The narrative, liquidity, volatility, delivery, and extreme-day figures are in `equity_analysis_report.docx`.
