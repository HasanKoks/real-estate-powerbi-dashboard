# Data Model

The dashboard runs on a star schema, the standard pattern for BI
reporting. One fact table sits at the centre, surrounded by dimension
tables that describe the context of each transaction.

## Star schema

![Data Model](../screenshots/data_model.png)

## Fact table

### transactions
The central table. One row per closed deal. 3,446 rows after cleaning.

Columns:
- `transaction_id` (text, unique)
- `property_id` (text, foreign key)
- `agent_id` (text, foreign key)
- `closing_date` (date)
- `closing_price` (decimal)
- `YearMonth` (calculated text, format YYYY-MM)

## Dimension tables

### property_attributes
One row per property. 5,000 rows.

Columns:
- `property_id` (text, primary key)
- `region_id` (text, foreign key)
- `property_type` (text: Apartment, Detached House, Penthouse, Studio, Duplex)
- `size_m2` (whole number)
- `rooms` (whole number)
- `year_built` (whole number)
- `has_parking` (whole number, 0 or 1)

### listings
One row per property listing. 5,000 rows.

Columns:
- `property_id` (text, primary key)
- `listed_date` (date)
- `asking_price` (decimal)

### agents
One row per real estate agent. 40 rows.

Columns:
- `agent_id` (text, primary key)
- `agent_name` (text)
- `region_id` (text, foreign key)
- `years_experience` (whole number)

### regions
One row per Istanbul region. 6 rows.

Columns:
- `region_id` (text, primary key)
- `region_name` (text)
- `city` (text)
- `avg_income_index` (decimal)

### calendar
One row per day across 2 years. 730 rows.

Columns:
- `date` (date, primary key)
- `year` (whole number)
- `month` (whole number)
- `quarter` (whole number)
- `year_month` (text, format YYYY-MM)
- `mortgage_rate` (decimal, calculated via LOOKUPVALUE)

Marked as a date table in Power BI to enable time intelligence functions.

### market_index
Monthly market index, one row per month. 24 rows.

Columns:
- `year_month` (text, primary key)
- `market_index` (decimal)

### mortgage_rates
Monthly mortgage rates, one row per month. 24 rows.

Columns:
- `year_month` (text, primary key)
- `mortgage_rate` (decimal)

## Relationships

All relationships are Many-to-one with Single cross-filter direction.

| From (Many)                       | To (One)                         | Status   |
|-----------------------------------|----------------------------------|----------|
| transactions[property_id]         | property_attributes[property_id] | Active   |
| transactions[property_id]         | listings[property_id]            | Active   |
| transactions[agent_id]            | agents[agent_id]                 | Active   |
| transactions[closing_date]        | calendar[date]                   | Active   |
| transactions[YearMonth]           | market_index[year_month]         | Active   |
| transactions[YearMonth]           | mortgage_rates[year_month]       | Active   |
| property_attributes[region_id]    | regions[region_id]               | Active   |

## Design decisions

### Why a star schema and not a single flat table

A flat table is simpler but does not let visuals filter independently
across categories. With the star schema, a slicer on `regions` filters
both transactions and property aggregations correctly through the model,
not just one denormalised set.

### Why YearMonth is a calculated column on transactions

`market_index` and `mortgage_rates` are monthly granularity, but
`transactions` are daily. Adding a `YearMonth` column on the fact table
gives a clean foreign key into the monthly dimension tables without
needing a complex join.

### Why calendar is marked as a date table

Power BI's time intelligence functions (`SAMEPERIODLASTYEAR`, `DATEADD`,
`DATESYTD`, etc) require a date table to work reliably. Marking it
explicitly avoids subtle bugs in YoY and MoM calculations.

### Why mortgage_rate is a LOOKUPVALUE column instead of a relationship pull

Power BI averages dimension values across grouping contexts. When
`mortgage_rate` was visualised by month via the standard relationship,
each calendar month received the average of all transactions' mortgage
rates, which collapsed the monthly variation. Adding `mortgage_rate` as
a `LOOKUPVALUE` column directly on the calendar table sidesteps this and
gives one rate per calendar day, preserving the actual monthly trend.
