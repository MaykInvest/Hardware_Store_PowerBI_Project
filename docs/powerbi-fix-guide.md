# Power BI fix guide

The data model inside `Project_TechStore.pbix` is stored in a compressed binary format,
so these fixes have to be made in Power BI Desktop. Do them in this order. Measure names
match the ones already in the `_measures` table.

> **Tip:** when you are done, use *File → Save as → Power BI Project (.pbip)*. The model,
> measures and Power Query steps are then saved as text files, so they can be reviewed in
> Git and edited outside Power BI Desktop.

---

## 1. Fix the 100× currency bug (critical)

**Symptom:** total revenue shows R$ 165.98M, but the CSV totals R$ 1,659,815.19.
Power Query read `111.99` as `11199` because it used Portuguese (Brazil) number
settings, where `.` is the thousands separator.

**Fix:** *Home → Transform data*. For **each** table, open the `Changed Type` step in
*Applied Steps* and change the locale on that step. In the formula bar, add `"en-US"` as
the last argument of `Table.TransformColumnTypes`:

```m
= Table.TransformColumnTypes(#"Promoted Headers", {
    {"unit_price", type number}, {"total_price", type number}, ...
}, "en-US")
```

(Or right-click a column → *Change Type → Using Locale… → Decimal Number / English
(United States)*, but the formula approach fixes all columns in one go.)

Decimal columns per table:

| Table | Columns |
|---|---|
| `fact_sales` | `unit_price`, `unit_cost`, `discount`, `total_price`, `gross_margin`, `margin_pct` |
| `fact_returns` | `original_amount`, `refund_amount` |
| `fact_support` | `satisfaction_score` |
| `fact_inventory` | `unit_cost`, `unit_price`, `gross_margin_unit`, `margin_pct` |
| `dim_products` | `unit_price`, `unit_cost` |
| `dim_customers` | `lifetime_value` |

**Check:** *Close & Apply*. A card with `REV_Total Revenue` and no filters must show
**R$ 1,659,815.19**, and 2024 must be **R$ 593,304.88**.

**Then rescale any hard-coded targets.** If `TGT_Revenue Target` is a fixed number
(e.g. `14000000`), it was sized for the inflated data. Divide it by 100, or better,
replace it with the dynamic version in step 3.

---

## 2. Remove "(Blank)" from the slicers

**Cause:** some fact rows have dates outside the `Calendar` table. Eight returns are dated
Jan 2025, but the calendar ends on 2024-12-31.

**Fix (recommended):** relate `fact_returns` to `Calendar` on `sale_date` instead of
`return_date`. A return then counts in the period of the sale it belongs to, which is
also what a return rate needs. In *Model view*, delete the `fact_returns[return_date]`
relationship and drag `fact_returns[sale_date]` onto `Calendar[Date]`.

**Alternative:** extend the calendar to cover every fact date. If `Calendar` is a DAX table:

```dax
Calendar =
VAR MinDate = MIN ( MIN ( fact_sales[order_date] ), MIN ( fact_support[created_date] ) )
VAR MaxDate = MAX ( MAX ( fact_sales[order_date] ), MAX ( fact_returns[return_date] ) )
RETURN
    CALENDAR ( DATE ( YEAR ( MinDate ), 1, 1 ), DATE ( YEAR ( MaxDate ), 12, 31 ) )
```

(Keep your existing added columns: Year, Quarter, Month Name, Year Quarter, sort columns.)
This adds a nearly empty 2025 to the Year slicer, which is why the first option is better.

**Check:** the Year, Quarter and Month slicers no longer show "(Blank)". If one still does,
look for another relationship whose fact dates fall outside the calendar.

---

## 3. Make the measures say what the visuals claim

```dax
RET_Return Rate % =
DIVIDE ( DISTINCTCOUNT ( fact_returns[sale_id] ), COUNTROWS ( fact_sales ) )
```
Returned order lines ÷ order lines sold (≈ 8.0% overall). Add a description to the
measure so the definition shows in tooltips.

```dax
TIME_Revenue YTD =
TOTALYTD ( [REV_Total Revenue], 'Calendar'[Date] )

TGT_Revenue Target =
CALCULATE ( [REV_Total Revenue], SAMEPERIODLASTYEAR ( 'Calendar'[Date] ) ) * 1.10

TGT_Revenue Target YTD =
CALCULATE ( [TGT_Revenue Target], DATESYTD ( 'Calendar'[Date] ) )
```
The target is "10% above the same period last year", so it works at any level: month,
quarter or year. 2022 has no prior year, so its target is blank. That's expected.

---

## 4. Fix the visuals

**KPI cards (Revenue, Gross Margin %, Return Rate %, Satisfaction).**
A KPI visual shows the **last point of its trend axis** (`Year Quarter`), so with no
filter the cards show Q4 2024, not the total. Pick one:
- Keep the KPI visuals and rename the titles to say what they show, e.g.
  *"Revenue · latest quarter"*, or
- Switch to the new **Card** visual, which shows the full filtered total, with the
  target as a reference label.

**Return Rate KPI:** *Format → Trend axis → Direction → Low is good*, so 8% against a
15% goal shows green for the right reason.

**Total Orders:** give it a target and trend like the other three, or keep it as a plain
card with a small "orders this year vs last year" reference label.

**Revenue YTD vs Target gauge:** *Value* = `TIME_Revenue YTD`, *Target value* =
`TGT_Revenue Target YTD`, *Maximum value* = a measure such as
`[TGT_Revenue Target YTD] * 1.2`.

**Currency formatting:** in *Model view*, select each currency measure (`REV_Total
Revenue`, `TIME_Revenue YTD`, the `TGT_` revenue measures) → *Format → Currency →
R$ Portuguese (Brazil)*, 0–2 decimals. Use display units **Thousands** on cards
(e.g. R$ 593K). This also fixes the "$14M" goal label.

**Revenue Trend:** *Format → Lines → Colors*. Make the current year a strong colour and
earlier years muted grey, or use three clearly different hues.

**Month slicer:** *Format → Slicer settings → Style → Dropdown* so it doesn't need
horizontal scrolling.

**Donut centre label:** after step 1 the total is R$ 1.66M, which fits. Set display
units to Millions with 2 decimals.

---

## 5. Refresh and update the screenshot

The CSVs were also cleaned (see the commit that touched `TechStore_CSV/`):

- `fact_sales.region` now holds the Brazilian macro-region of the customer's state
  (Norte, Nordeste, Centro-Oeste, Sudeste, Sul) instead of first names.
- `fact_sales.channel` and `payment_method` are now the same on every line of an order.
- `fact_returns.days_to_return` = `return_date − sale_date`.
- `fact_inventory.units_sold_ytd` = units sold in 2024, from `fact_sales`.
- `dim_customers.email` no longer has accented characters.

Click **Home → Refresh**, check the numbers from step 1, then replace
`imagens/Executive Summary.png` with a new screenshot.
