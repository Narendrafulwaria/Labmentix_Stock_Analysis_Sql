# Problem Statement: MySQL Analysis of Selected Indian Equities

## Goal

Load the daily equity history in the `data` folder into MySQL and use SQL to compare price performance, trading activity, and volatility across six listed companies from 1 January 2015 through 31 July 2018.

The analysis should answer which stocks gained the most, how auto and IT names behaved relative to each other, and which names were the most actively traded and the most volatile over the full period and by calendar year.

## Context

The `data` folder contains one CSV per company. Each file is a daily trading series (BSE-style OHLC and volume fields), newest date first. Prices are not adjusted for corporate actions in the source files, so results describe the recorded series as given.

| File | Company | Sector | Rows (excluding header) | Date range |
| --- | --- | --- | --- | --- |
| `data/Bajaj Auto.csv` | Bajaj Auto | Auto | 889 | 1 Jan 2015 – 31 Jul 2018 |
| `data/Eicher Motors.csv` | Eicher Motors | Auto | 889 | 1 Jan 2015 – 31 Jul 2018 |
| `data/Hero Motocorp.csv` | Hero Motocorp | Auto | 889 | 1 Jan 2015 – 31 Jul 2018 |
| `data/TVS Motors.csv` | TVS Motors | Auto | 889 | 1 Jan 2015 – 31 Jul 2018 |
| `data/TCS.csv` | TCS | IT | 889 | 1 Jan 2015 – 31 Jul 2018 |
| `data/Infosys.csv` | Infosys | IT | 889 | 1 Jan 2015 – 31 Jul 2018 |

Shared columns:

- `Date`
- `Open Price`, `High Price`, `Low Price`, `Close Price`
- `WAP` (weighted average price)
- `No.of Shares`, `No. of Trades`, `Total Turnover (Rs.)`
- `Deliverable Quantity`, `% Deli. Qty to Traded Qty`
- `Spread High-Low`, `Spread Close-Open`

## Questions the analysis must answer

1. **Returns.** What was each stock’s close-to-close return from the first trading day to the last? Rank the six companies by total return.
2. **Yearly path.** For each calendar year (2015, 2016, 2017, and 2018 year-to-date), what was each stock’s return, and which name led that year?
3. **Sector comparison.** How did the four auto stocks compare with the two IT stocks on average total return and average daily turnover?
4. **Volatility.** Which stocks had the widest typical daily range (`Spread High-Low` relative to close) and the largest day-to-day close moves?
5. **Liquidity.** Which stocks had the highest average shares traded, trade count, and turnover? Did higher turnover line up with lower daily ranges?
6. **Delivery.** Which names showed a higher share of deliverable quantity versus traded quantity, as a rough signal of delivery-based (less speculative) trading?
7. **Extremes.** What were the highest close, lowest close, and the single largest up and down close-to-close days for each stock?

## Scope

In scope:

- Design a small relational schema (company dimension plus a daily price fact table).
- Load all six CSVs into MySQL with consistent types and a parsed date.
- Write SQL that produces the answers above as result sets.
- Summarize findings in plain language tied to those queries.

Out of scope:

- Live market data, forecasts, or buy/sell recommendations.
- Adjusting prices for splits, bonuses, or dividends unless that adjustment is added later as a separate step.
- Charts or a web dashboard. The deliverable is the database and the SQL analysis.

## Success criteria

- All 889 rows per company load without dropped dates or duplicated `(company, date)` pairs.
- Dates are real date values, ordered correctly (the CSVs arrive newest-first).
- Each question above is answered by a saved SQL query that runs against the loaded tables.
- The written summary names the top and bottom performers on return, names the most and least liquid stocks, and states the yearly leaders, with figures taken from the query results.
