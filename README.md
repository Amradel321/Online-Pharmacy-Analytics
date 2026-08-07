# E-Pharmacy Order & Fulfilment Analytics (SQL + Power BI)

An end-to-end analysis of a healthtech order dataset covering orders, order
state history, pharmacies, and products. The project loads and cleans raw
data in MySQL, answers a set of operational and business questions using
SQL (staging tables, window functions, CTEs), and visualizes the key
findings in an interactive four-page Power BI dashboard.

## Data Overview

The dataset consists of five tables:

- **orders** — one row per order (source, pharmacy, current state)
- **orderitems** — line items per order (product, quantity, price)
- **orderstates** — full state history per order (timestamped), used to trace
  an order's journey from creation to final resolution
- **orderstatetypes** — lookup table mapping state IDs to readable names
- **pharmacies** — pharmacy metadata, including a test-account flag

## Data Loading & Cleaning

Raw CSVs are loaded into MySQL using `LOAD DATA LOCAL INFILE`. A few cleaning
steps were needed before the data was analysis-ready:

- Date fields (`CreatedOn`, `Time_Stamp`, `ScheduledTo`) arrived as text and
  were parsed with `STR_TO_DATE` into proper `DATETIME` columns.
- `ScheduledTo` had empty strings instead of NULLs, and some values carried
  a trailing `\r` from the file encoding — both were cleaned before parsing.
- A staging table, `orders_staging`, excludes orders placed by internal test
  pharmacies (`TestFlag = 1`) — about 530 orders, under 0.5% of total volume.
  All downstream analysis is built on this cleaned table.

One additional data-quality finding worth noting: some orders pass through
the "scheduled" state more than once (roughly 750 out of ~8,000 scheduled
orders). Queries involving this state use `COUNT(DISTINCT OrderId)` rather
than `COUNT(*)` to avoid inflating the results.

## Business Questions Answered

**Orders**
- Total order volume and breakdown by order source
- Overall fulfilment rate (delivered vs. total orders)
- Month-over-month average order value
- Fulfilment rate specifically for scheduled orders
- Top reasons orders fail to complete, ranked by share of total orders
- Most common state an order was in immediately before being rejected

**Products**
- Top products by order count
- Top products by total sales value

**Pharmacies**
- Top pharmacies by order volume
- Top pharmacies by fulfilment rate (filtered to pharmacies above a minimum
  order threshold, to avoid low-volume pharmacies skewing the ranking)
- Total sales value per pharmacy
- Average customer wait time per pharmacy (order creation → last state update)
- Average delivery time per pharmacy (order accepted → order delivered)
- Top pharmacy by order volume within each order source

## SQL Techniques Used

- **Staging tables** to isolate clean data before analysis
- **Conditional aggregation** (`SUM(condition)`) to count rows matching a
  condition without a `CASE WHEN`
- **Window functions**:
  - `LAG()` to find the state an order was in immediately before rejection
  - `DENSE_RANK()` to find the top pharmacy per order source
- **CTEs (`WITH`)** for multi-step transformations (aggregate → then rank)
  and for logic with enough nesting that named steps improve readability.
  Simpler single-step queries were deliberately left as plain queries rather
  than wrapped in unnecessary CTEs.
- **`COUNT(DISTINCT ...)` vs `COUNT(*)`** applied carefully wherever a join
  could introduce duplicate rows per order

## Power BI Dashboard

The SQL results feed a four-page interactive dashboard, connected live to
the MySQL database (with a couple of the more complex queries — the
month-over-month average order value, and the rejection-state analysis —
loaded as direct SQL queries rather than rebuilt in DAX, since the
underlying logic is a better fit for SQL).

**Overview** — headline KPIs: total orders, overall fulfilment rate, total
sales, top non-fulfilment reasons, and an average order value trend by month.

**Orders Analysis** — scheduled order volume and fulfilment rate, order
volume by source, and order volume trend by month.

**Products Analysis** — top 5 products by order count, and top 5 products
by sales value.

**Pharmacies Analysis** — top 5 pharmacies by order count, by fulfilment
rate, and by total sales value.

A few visuals from the original SQL analysis were deliberately left out of
the dashboard — for example, fulfilment rate and top-pharmacy breakdowns by
order source — because one source (Call Center) has too few orders for
those metrics to be meaningful, and including them would have been more
misleading than informative. The underlying queries are still included in
the SQL file for reference.

**Data model:** built on `orders_staging`, `orderitems`, `pharmacies`, and
`orderstatetypes`, related through `PharmacyKey`/`Key` and `OrderId`/`Id`.
Relationships are single-direction, many-to-one, matching the natural
grain of each table (many orders per pharmacy, many line items per order).

## Notes

Raw data files are not included in this repository,
as they contain operational business data.
The SQL file and dashboard are structured so the project can be reproduced against any similarly-shaped dataset
