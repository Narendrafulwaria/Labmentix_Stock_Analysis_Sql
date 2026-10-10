# Implementation: Equity History in MySQL

Build steps for the database in `architecture.md` and the questions in `problemstatement.md`.

Target: MySQL 8.0 or later, database `equity_analysis`, run from this folder. Prices stay unadjusted.

Each task ends in a query. The comment block directly above the query is the result to expect. A run that does not match that block has not loaded the six CSVs correctly.

One data fact changes the delivery columns from the architecture note: six source rows have an empty deliverable quantity and an empty delivery percentage. Those two columns are nullable, and the empty values are stored as NULL. Every other field is present on all 5,334 rows.

## Task 1 — Create the database and the company list

Creates `equity_analysis` and loads the six companies with sector and source file name.

```sql
CREATE DATABASE IF NOT EXISTS equity_analysis
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_unicode_ci;

USE equity_analysis;

CREATE TABLE companies (
  company_id   SMALLINT UNSIGNED NOT NULL AUTO_INCREMENT,
  company_name VARCHAR(64) NOT NULL,
  sector       VARCHAR(16) NOT NULL,
  source_file  VARCHAR(64) NOT NULL,
  PRIMARY KEY (company_id),
  UNIQUE KEY uq_companies_name (company_name),
  UNIQUE KEY uq_companies_file (source_file)
);

INSERT INTO companies (company_name, sector, source_file) VALUES
  ('Bajaj Auto',     'Auto', 'Bajaj Auto.csv'),
  ('Eicher Motors',  'Auto', 'Eicher Motors.csv'),
  ('Hero Motocorp',  'Auto', 'Hero Motocorp.csv'),
  ('TVS Motors',     'Auto', 'TVS Motors.csv'),
  ('TCS',            'IT',   'TCS.csv'),
  ('Infosys',        'IT',   'Infosys.csv');

-- Output: 6 rows, in insert order
-- company_id | company_name   | sector | source_file
-- 1          | Bajaj Auto     | Auto   | Bajaj Auto.csv
-- 2          | Eicher Motors  | Auto   | Eicher Motors.csv
-- 3          | Hero Motocorp  | Auto   | Hero Motocorp.csv
-- 4          | TVS Motors     | Auto   | TVS Motors.csv
-- 5          | TCS            | IT     | TCS.csv
-- 6          | Infosys        | IT     | Infosys.csv
SELECT company_id, company_name, sector, source_file
FROM companies
ORDER BY company_id;
```

## Task 2 — Land the CSVs in staging

Staging keeps every CSV field as text. `source_file` must match `companies.source_file` exactly.

The session needs local file loading (`mysql --local-infile=1`). Repeat the `LOAD DATA` once per file, changing the path and the `SET source_file` value. The six files are `Bajaj Auto.csv`, `Eicher Motors.csv`, `Hero Motocorp.csv`, `TVS Motors.csv`, `TCS.csv`, and `Infosys.csv`.

```sql
CREATE TABLE stg_daily_prices (
  source_file        VARCHAR(64) NOT NULL,
  date_text          VARCHAR(32) NOT NULL,
  open_price         VARCHAR(32) NOT NULL,
  high_price         VARCHAR(32) NOT NULL,
  low_price          VARCHAR(32) NOT NULL,
  close_price        VARCHAR(32) NOT NULL,
  wap                VARCHAR(64) NOT NULL,
  shares             VARCHAR(32) NOT NULL,
  trades             VARCHAR(32) NOT NULL,
  turnover           VARCHAR(32) NOT NULL,
  deliverable_qty    VARCHAR(32) NOT NULL,
  delivery_pct       VARCHAR(32) NOT NULL,
  spread_high_low    VARCHAR(32) NOT NULL,
  spread_close_open  VARCHAR(32) NOT NULL
);

LOAD DATA LOCAL INFILE 'C:/Users/dell/OneDrive/Desktop/Labmentix/sql/Bajaj Auto.csv'
INTO TABLE stg_daily_prices
FIELDS TERMINATED BY ','
OPTIONALLY ENCLOSED BY '"'
LINES TERMINATED BY '\n'
IGNORE 1 LINES
(date_text, open_price, high_price, low_price, close_price, wap,
 shares, trades, turnover, deliverable_qty, delivery_pct,
 spread_high_low, spread_close_open)
SET source_file = 'Bajaj Auto.csv';

-- Output after all six files:
-- source_file          | rows_loaded
-- Bajaj Auto.csv       | 889
-- Eicher Motors.csv    | 889
-- Hero Motocorp.csv    | 889
-- Infosys.csv          | 889
-- TCS.csv              | 889
-- TVS Motors.csv       | 889
SELECT source_file, COUNT(*) AS rows_loaded
FROM stg_daily_prices
GROUP BY source_file
ORDER BY source_file;
```

If `LOAD DATA` is blocked by `secure_file_priv`, import with a client that can insert into `stg_daily_prices` and set `source_file` per batch. Task 3 does not change.

## Task 3 — Build the daily price table

Dates such as `31-July-2018` are parsed with `%d-%M-%Y`. Empty delivery fields become NULL. Re-running this task starts from an empty fact table.

```sql
CREATE TABLE daily_prices (
  price_id           INT UNSIGNED NOT NULL AUTO_INCREMENT,
  company_id         SMALLINT UNSIGNED NOT NULL,
  trade_date         DATE NOT NULL,
  open_price         DECIMAL(12,2) NOT NULL,
  high_price         DECIMAL(12,2) NOT NULL,
  low_price          DECIMAL(12,2) NOT NULL,
  close_price        DECIMAL(12,2) NOT NULL,
  wap                DECIMAL(20,6) NOT NULL,
  shares_traded      BIGINT UNSIGNED NOT NULL,
  trade_count        INT UNSIGNED NOT NULL,
  turnover           DECIMAL(18,2) NOT NULL,
  deliverable_qty    BIGINT UNSIGNED NULL,
  delivery_pct       DECIMAL(6,2) NULL,
  spread_high_low    DECIMAL(12,2) NOT NULL,
  spread_close_open  DECIMAL(12,2) NOT NULL,
  PRIMARY KEY (price_id),
  UNIQUE KEY uq_company_date (company_id, trade_date),
  KEY ix_trade_date (trade_date),
  CONSTRAINT fk_daily_prices_company
    FOREIGN KEY (company_id) REFERENCES companies (company_id)
);

TRUNCATE TABLE daily_prices;

INSERT INTO daily_prices (
  company_id, trade_date,
  open_price, high_price, low_price, close_price, wap,
  shares_traded, trade_count, turnover,
  deliverable_qty, delivery_pct,
  spread_high_low, spread_close_open
)
SELECT
  c.company_id,
  STR_TO_DATE(s.date_text, '%d-%M-%Y'),
  s.open_price, s.high_price, s.low_price, s.close_price, s.wap,
  s.shares, s.trades, s.turnover,
  NULLIF(s.deliverable_qty, ''),
  NULLIF(s.delivery_pct, ''),
  s.spread_high_low, s.spread_close_open
FROM stg_daily_prices AS s
JOIN companies AS c ON c.source_file = s.source_file
WHERE STR_TO_DATE(s.date_text, '%d-%M-%Y') IS NOT NULL;

-- Output:
-- loaded_rows | null_delivery_rows
-- 5334        | 6
SELECT
  COUNT(*) AS loaded_rows,
  SUM(deliverable_qty IS NULL) AS null_delivery_rows
FROM daily_prices;
```

## Task 4 — Validate the load

All three checks below must match before the analysis tasks are treated as final.

```sql
-- Output:
-- company_name   | row_count | first_date | last_date
-- Bajaj Auto     | 889       | 2015-01-01 | 2018-07-31
-- Eicher Motors  | 889       | 2015-01-01 | 2018-07-31
-- Hero Motocorp  | 889       | 2015-01-01 | 2018-07-31
-- Infosys        | 889       | 2015-01-01 | 2018-07-31
-- TCS            | 889       | 2015-01-01 | 2018-07-31
-- TVS Motors     | 889       | 2015-01-01 | 2018-07-31
SELECT
  c.company_name,
  COUNT(*) AS row_count,
  MIN(p.trade_date) AS first_date,
  MAX(p.trade_date) AS last_date
FROM daily_prices AS p
JOIN companies AS c ON c.company_id = p.company_id
GROUP BY c.company_name
ORDER BY c.company_name;

-- Output: 0 rows
SELECT company_id, trade_date, COUNT(*) AS dupes
FROM daily_prices
GROUP BY company_id, trade_date
HAVING COUNT(*) > 1;

-- Output:
-- bad_ranges
-- 0
SELECT COUNT(*) AS bad_ranges
FROM daily_prices
WHERE high_price < low_price
   OR high_price < open_price
   OR high_price < close_price
   OR low_price > open_price
   OR low_price > close_price;

-- Output: the six source rows whose delivery fields were blank
-- company_name   | trade_date
-- Bajaj Auto     | 2017-08-31
-- Eicher Motors  | 2015-12-09
-- Hero Motocorp  | 2015-12-09
-- Infosys        | 2017-08-31
-- TCS            | 2015-12-09
-- TVS Motors     | 2015-12-09
SELECT c.company_name, p.trade_date
FROM daily_prices AS p
JOIN companies AS c ON c.company_id = p.company_id
WHERE p.deliverable_qty IS NULL
ORDER BY c.company_name;
```

## Task 5 — Total return

Close on the last trading day versus close on the first trading day, ranked from the largest gain to the largest loss. This is the recorded price change. It is not adjusted for a bonus or a split.

```sql
-- Output:
-- company_name   | sector | first_date | first_close | last_date  | last_close | total_return_pct
-- TVS Motors     | Auto   | 2015-01-01 | 276.85      | 2018-07-31 | 517.45     | 86.91
-- Eicher Motors  | Auto   | 2015-01-01 | 15239.15    | 2018-07-31 | 27820.95   | 82.56
-- Bajaj Auto     | Auto   | 2015-01-01 | 2454.10     | 2018-07-31 | 2700.70    | 10.05
-- Hero Motocorp  | Auto   | 2015-01-01 | 3107.30     | 2018-07-31 | 3293.80    | 6.00
-- TCS            | IT     | 2015-01-01 | 2548.20     | 2018-07-31 | 1941.25    | -23.82
-- Infosys        | IT     | 2015-01-01 | 1975.80     | 2018-07-31 | 1365.00    | -30.91
WITH bounds AS (
  SELECT
    company_id,
    MIN(trade_date) AS first_date,
    MAX(trade_date) AS last_date
  FROM daily_prices
  GROUP BY company_id
)
SELECT
  c.company_name,
  c.sector,
  b.first_date,
  first_px.close_price AS first_close,
  b.last_date,
  last_px.close_price AS last_close,
  ROUND((last_px.close_price - first_px.close_price)
        / first_px.close_price * 100, 2) AS total_return_pct
FROM bounds AS b
JOIN companies AS c ON c.company_id = b.company_id
JOIN daily_prices AS first_px
  ON first_px.company_id = b.company_id
 AND first_px.trade_date = b.first_date
JOIN daily_prices AS last_px
  ON last_px.company_id = b.company_id
 AND last_px.trade_date = b.last_date
ORDER BY total_return_pct DESC;
```

## Task 6 — Return by calendar year

The first statement is every company in every year, best return first within the year. The second statement keeps only the leader of each year. 2018 uses the first and last session inside 2018, so it is year-to-date through 31 July 2018.

```sql
-- Output:
-- company_name   | sector | yr   | year_return_pct
-- Eicher Motors  | Auto   | 2015 | 10.60
-- TVS Motors     | Auto   | 2015 | 4.62
-- Bajaj Auto     | Auto   | 2015 | 3.24
-- TCS            | IT     | 2015 | -4.28
-- Hero Motocorp  | Auto   | 2015 | -13.26
-- Infosys        | IT     | 2015 | -44.10
-- TVS Motors     | Auto   | 2016 | 25.62
-- Eicher Motors  | Auto   | 2016 | 25.61
-- Hero Motocorp  | Auto   | 2016 | 13.28
-- Bajaj Auto     | Auto   | 2016 | 4.52
-- TCS            | IT     | 2016 | -2.25
-- Infosys        | IT     | 2016 | -8.38
-- TVS Motors     | Auto   | 2017 | 110.65
-- Eicher Motors  | Auto   | 2017 | 34.56
-- Bajaj Auto     | Auto   | 2017 | 27.97
-- Hero Motocorp  | Auto   | 2017 | 25.01
-- TCS            | IT     | 2017 | 14.47
-- Infosys        | IT     | 2017 | 3.76
-- Infosys        | IT     | 2018 | 31.94
-- Eicher Motors  | Auto   | 2018 | -6.96
-- Hero Motocorp  | Auto   | 2018 | -12.60
-- Bajaj Auto     | Auto   | 2018 | -18.00
-- TCS            | IT     | 2018 | -26.87
-- TVS Motors     | Auto   | 2018 | -32.48
WITH year_bounds AS (
  SELECT
    company_id,
    YEAR(trade_date) AS yr,
    MIN(trade_date) AS first_date,
    MAX(trade_date) AS last_date
  FROM daily_prices
  GROUP BY company_id, YEAR(trade_date)
),
yearly AS (
  SELECT
    c.company_name,
    c.sector,
    y.yr,
    ROUND((last_px.close_price - first_px.close_price)
          / first_px.close_price * 100, 2) AS year_return_pct
  FROM year_bounds AS y
  JOIN companies AS c ON c.company_id = y.company_id
  JOIN daily_prices AS first_px
    ON first_px.company_id = y.company_id
   AND first_px.trade_date = y.first_date
  JOIN daily_prices AS last_px
    ON last_px.company_id = y.company_id
   AND last_px.trade_date = y.last_date
)
SELECT * FROM yearly
ORDER BY yr, year_return_pct DESC;

-- Output:
-- yr   | company_name  | year_return_pct
-- 2015 | Eicher Motors | 10.60
-- 2016 | TVS Motors    | 25.62
-- 2017 | TVS Motors    | 110.65
-- 2018 | Infosys       | 31.94
WITH year_bounds AS (
  SELECT
    company_id,
    YEAR(trade_date) AS yr,
    MIN(trade_date) AS first_date,
    MAX(trade_date) AS last_date
  FROM daily_prices
  GROUP BY company_id, YEAR(trade_date)
),
yearly AS (
  SELECT
    c.company_name,
    y.yr,
    (last_px.close_price - first_px.close_price)
      / first_px.close_price AS year_return
  FROM year_bounds AS y
  JOIN companies AS c ON c.company_id = y.company_id
  JOIN daily_prices AS first_px
    ON first_px.company_id = y.company_id
   AND first_px.trade_date = y.first_date
  JOIN daily_prices AS last_px
    ON last_px.company_id = y.company_id
   AND last_px.trade_date = y.last_date
)
SELECT yr, company_name, ROUND(year_return * 100, 2) AS year_return_pct
FROM (
  SELECT
    yr, company_name, year_return,
    ROW_NUMBER() OVER (PARTITION BY yr ORDER BY year_return DESC) AS rn
  FROM yearly
) AS ranked
WHERE rn = 1
ORDER BY yr;
```

## Task 7 — Auto compared with IT

Average of the company-level total returns, and the average of each company’s own mean daily turnover. Turnover is in rupees.

```sql
-- Output:
-- sector | companies | avg_total_return_pct | avg_of_company_daily_turnover
-- Auto   | 4         | 46.38                | 108619004.67
-- IT     | 2         | -27.37               | 281433335.95
WITH bounds AS (
  SELECT company_id,
         MIN(trade_date) AS first_date,
         MAX(trade_date) AS last_date
  FROM daily_prices
  GROUP BY company_id
),
company_return AS (
  SELECT
    b.company_id,
    (last_px.close_price - first_px.close_price)
      / first_px.close_price AS total_return
  FROM bounds AS b
  JOIN daily_prices AS first_px
    ON first_px.company_id = b.company_id
   AND first_px.trade_date = b.first_date
  JOIN daily_prices AS last_px
    ON last_px.company_id = b.company_id
   AND last_px.trade_date = b.last_date
),
company_turnover AS (
  SELECT company_id, AVG(turnover) AS avg_daily_turnover
  FROM daily_prices
  GROUP BY company_id
)
SELECT
  c.sector,
  COUNT(*) AS companies,
  ROUND(AVG(r.total_return) * 100, 2) AS avg_total_return_pct,
  ROUND(AVG(t.avg_daily_turnover), 2) AS avg_of_company_daily_turnover
FROM companies AS c
JOIN company_return AS r ON r.company_id = c.company_id
JOIN company_turnover AS t ON t.company_id = c.company_id
GROUP BY c.sector
ORDER BY c.sector;
```

## Task 8 — Volatility

`avg_daily_range_pct` is the mean of `Spread High-Low / Close`. `avg_abs_day_move_pct` is the mean absolute close-to-close change. The first session of each company has no previous close and is left out of the day-move average.

```sql
-- Output:
-- company_name   | avg_daily_range_pct | avg_abs_day_move_pct | max_up_day_pct | max_down_day_pct
-- TVS Motors     | 3.010               | 1.375                | 13.15          | -9.66
-- Eicher Motors  | 2.598               | 1.355                | 7.01           | -9.33
-- Bajaj Auto     | 2.209               | 1.048                | 7.40           | -9.09
-- Hero Motocorp  | 2.175               | 1.045                | 6.69           | -6.20
-- Infosys        | 2.112               | 1.141                | 11.05          | -49.86
-- TCS            | 1.933               | 1.046                | 6.76           | -50.40
WITH moves AS (
  SELECT
    company_id,
    trade_date,
    spread_high_low / close_price AS range_pct,
    close_price / LAG(close_price) OVER (
      PARTITION BY company_id ORDER BY trade_date
    ) - 1 AS day_return
  FROM daily_prices
)
SELECT
  c.company_name,
  ROUND(AVG(m.range_pct) * 100, 3) AS avg_daily_range_pct,
  ROUND(AVG(ABS(m.day_return)) * 100, 3) AS avg_abs_day_move_pct,
  ROUND(MAX(m.day_return) * 100, 2) AS max_up_day_pct,
  ROUND(MIN(m.day_return) * 100, 2) AS max_down_day_pct
FROM moves AS m
JOIN companies AS c ON c.company_id = m.company_id
GROUP BY c.company_name
ORDER BY avg_daily_range_pct DESC;
```

## Task 9 — Liquidity beside the daily range

Average shares, trade count, and turnover, with the same average daily range as Task 8. Read down the turnover column and across to the range column to see whether heavier rupee turnover sits with a tighter range.

```sql
-- Output:
-- company_name   | sector | avg_shares | avg_trades | avg_turnover   | avg_daily_range_pct
-- Infosys        | IT     | 266669     | 7742       | 304146432.34   | 2.112
-- TCS            | IT     | 104137     | 5024       | 258720239.55   | 1.933
-- Eicher Motors  | Auto   | 8288       | 1669       | 166438113.24   | 2.598
-- Hero Motocorp  | Auto   | 37109      | 2329       | 116218585.99   | 2.175
-- Bajaj Auto     | Auto   | 32645      | 2010       | 86514505.71    | 2.209
-- TVS Motors     | Auto   | 182805     | 3447       | 65304813.76    | 3.010
SELECT
  c.company_name,
  c.sector,
  ROUND(AVG(p.shares_traded), 0) AS avg_shares,
  ROUND(AVG(p.trade_count), 0) AS avg_trades,
  ROUND(AVG(p.turnover), 2) AS avg_turnover,
  ROUND(AVG(p.spread_high_low / p.close_price) * 100, 3) AS avg_daily_range_pct
FROM daily_prices AS p
JOIN companies AS c ON c.company_id = p.company_id
GROUP BY c.company_name, c.sector
ORDER BY avg_turnover DESC;
```

## Task 10 — Delivery

`avg_delivery_pct` averages the published daily percentage and skips the six NULL days. `delivery_pct_share_weighted` divides delivered shares by traded shares on the days that have a delivery quantity.

```sql
-- Output:
-- company_name   | sector | avg_delivery_pct | delivery_pct_share_weighted
-- Infosys        | IT     | 41.86            | 48.98
-- TCS            | IT     | 40.09            | 51.69
-- Hero Motocorp  | Auto   | 37.05            | 54.66
-- Eicher Motors  | Auto   | 33.27            | 57.44
-- Bajaj Auto     | Auto   | 31.00            | 48.23
-- TVS Motors     | Auto   | 23.56            | 28.61
SELECT
  c.company_name,
  c.sector,
  ROUND(AVG(p.delivery_pct), 2) AS avg_delivery_pct,
  ROUND(
    SUM(p.deliverable_qty)
    / SUM(CASE WHEN p.deliverable_qty IS NOT NULL THEN p.shares_traded END)
    * 100, 2
  ) AS delivery_pct_share_weighted
FROM daily_prices AS p
JOIN companies AS c ON c.company_id = p.company_id
GROUP BY c.company_name, c.sector
ORDER BY avg_delivery_pct DESC;
```

## Task 11 — Extremes

Highest close, lowest close, and the single largest up and down close-to-close days. Each extreme occurs on one date in this dataset, so the joins return one row per company.

```sql
-- Output:
-- company_name   | highest_close | highest_close_date | lowest_close | lowest_close_date | largest_up_date | largest_up_pct | largest_down_date | largest_down_pct
-- Bajaj Auto     | 3409.50       | 2018-02-01         | 1949.40      | 2015-04-30        | 2015-05-04      | 7.40           | 2015-08-24        | -9.09
-- Eicher Motors  | 32786.40      | 2017-09-07         | 14320.60     | 2015-04-21        | 2016-02-05      | 7.01           | 2015-08-24        | -9.33
-- Hero Motocorp  | 4047.30       | 2017-09-01         | 2279.80      | 2015-09-07        | 2016-03-01      | 6.69           | 2018-07-23        | -6.20
-- Infosys        | 2326.60       | 2015-02-19         | 873.50       | 2017-08-21        | 2015-07-21      | 11.05          | 2015-06-15        | -49.86
-- TCS            | 3604.80       | 2018-05-24         | 1721.20      | 2018-06-05        | 2018-04-20      | 6.76           | 2018-05-31        | -50.40
-- TVS Motors     | 785.30        | 2017-12-19         | 213.55       | 2015-08-24        | 2015-10-27      | 13.15          | 2016-05-03        | -9.66
WITH moves AS (
  SELECT
    company_id,
    trade_date,
    close_price,
    close_price / LAG(close_price) OVER (
      PARTITION BY company_id ORDER BY trade_date
    ) - 1 AS day_return
  FROM daily_prices
),
peaks AS (
  SELECT
    company_id,
    MAX(close_price) AS highest_close,
    MIN(close_price) AS lowest_close,
    MAX(day_return) AS best_day,
    MIN(day_return) AS worst_day
  FROM moves
  GROUP BY company_id
)
SELECT
  c.company_name,
  pk.highest_close,
  hi.trade_date AS highest_close_date,
  pk.lowest_close,
  lo.trade_date AS lowest_close_date,
  up.trade_date AS largest_up_date,
  ROUND(pk.best_day * 100, 2) AS largest_up_pct,
  dn.trade_date AS largest_down_date,
  ROUND(pk.worst_day * 100, 2) AS largest_down_pct
FROM peaks AS pk
JOIN companies AS c ON c.company_id = pk.company_id
JOIN moves AS hi
  ON hi.company_id = pk.company_id
 AND hi.close_price = pk.highest_close
JOIN moves AS lo
  ON lo.company_id = pk.company_id
 AND lo.close_price = pk.lowest_close
JOIN moves AS up
  ON up.company_id = pk.company_id
 AND up.day_return = pk.best_day
JOIN moves AS dn
  ON dn.company_id = pk.company_id
 AND dn.day_return = pk.worst_day
ORDER BY c.company_name;
```

## Task 12 — Read the results

The narrative for these result sets is in `equity_analysis_report.docx`. Every figure in that report comes from a comment block above.

To load a corrected CSV again:

1. `TRUNCATE TABLE stg_daily_prices;`
2. Repeat the six `LOAD DATA` statements in Task 2.
3. `TRUNCATE TABLE daily_prices;` and re-run the `INSERT ... SELECT` in Task 3.
4. Re-run Task 4, then Tasks 5 through 11, and confirm each result still matches its comment block.

`companies` stays as seeded unless a file name or a sector label changes.
