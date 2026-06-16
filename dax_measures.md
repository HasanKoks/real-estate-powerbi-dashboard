# DAX Measures

Every measure used in the dashboard, with the formula and the reasoning
behind it.

## Aggregation measures

### Total Transactions
Simple row count of the fact table.

```DAX
Total Transactions = COUNTROWS(transactions)
```

Used in the Page 1 KPI card and the donut chart values.

### Total Revenue
Sum of every closing price in the fact table.

```DAX
Total Revenue = SUM(transactions[closing_price])
```

Used in the Page 1 KPI card, the monthly trend line, and the regional bar
chart. Formatted as Whole Number with thousand separators, prefixed with
the Turkish Lira symbol.

### Avg Closing Price
Average closing price across all transactions.

```DAX
Avg Closing Price = AVERAGE(transactions[closing_price])
```

Used in the Page 1 KPI card.

## Cross-table measures using RELATED

These measures pull data across the star schema relationships, which is
the real test of Power BI modeling competence.

### Avg Price per m²
Total revenue divided by total square metres across all transactions.

```DAX
Avg Price per m2 = 
DIVIDE(
    SUM(transactions[closing_price]),
    SUMX(transactions, RELATED(property_attributes[size_m2]))
)
```

`RELATED` pulls `size_m2` from the `property_attributes` dimension table
via the `property_id` relationship. Used in the Page 2 matrix and the
Page 3 trend line.

### Avg Days on Market
Average number of days between listing date and closing date.

```DAX
Avg Days on Market = 
AVERAGEX(
    transactions,
    DATEDIFF(RELATED(listings[listed_date]), transactions[closing_date], DAY)
)
```

`RELATED` pulls `listed_date` from the `listings` dimension via
`property_id`. Calculates the difference row by row using `AVERAGEX`.
Used in the Page 1 KPI card.

### Avg Price Gap %
The percentage gap between asking price and closing price, weighted by
asking price.

```DAX
Avg Price Gap % = 
DIVIDE(
    SUMX(transactions, transactions[closing_price] - RELATED(listings[asking_price])),
    SUMX(transactions, RELATED(listings[asking_price]))
)
```

Tells the story of negotiation pressure in the market. Negative means
properties typically close below asking price, positive means above.

## Time intelligence measures

### Revenue YoY %
Year-over-year revenue growth percentage, comparing the current period
to the same period last year.

```DAX
Revenue YoY % = 
VAR Curr = [Total Revenue]
VAR Prior = CALCULATE([Total Revenue], SAMEPERIODLASTYEAR(calendar[date]))
RETURN DIVIDE(Curr - Prior, Prior)
```

Requires the `calendar` table to be marked as a date table. Used as
context next to KPI cards.

### Transactions MoM
Month-over-month change in transaction count.

```DAX
Transactions MoM = 
VAR Curr = [Total Transactions]
VAR Prior = CALCULATE([Total Transactions], DATEADD(calendar[date], -1, MONTH))
RETURN Curr - Prior
```

Used in trend analysis.

## Lookup-based calculated column

### Mortgage Rate (in calendar table)
Pulls the mortgage rate for each calendar month directly into the
calendar table, bypassing relationship aggregation issues.

```DAX
mortgage_rate = 
LOOKUPVALUE(
    mortgage_rates[mortgage_rate],
    mortgage_rates[year_month],
    calendar[year_month]
)
```

This is a calculated column, not a measure. Used because relationship-
based aggregation collapsed the monthly variation into a flat line on
the Page 3 chart. The lookup column gives one rate value per calendar
day, so when plotted by year_month, the rate shows its actual monthly
variation.

## Why these patterns matter

The dashboard uses three different DAX patterns that together demonstrate
mid-level modeling competence:

1. **RELATED** for pulling dimension data into fact-level calculations
2. **SAMEPERIODLASTYEAR** and **DATEADD** for proper time intelligence on
   a marked date table
3. **LOOKUPVALUE** as a calculated column when relationship-based
   aggregation does not give the right granularity

These three patterns cover roughly 80% of real-world Power BI use cases.
