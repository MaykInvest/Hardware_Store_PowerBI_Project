# TechStore — Hardware Store Sales Dashboard (Power BI)

An executive dashboard for **TechStore**, a fictional Brazilian retailer of PC hardware
(GPUs, processors, motherboards, storage, memory, PSUs and coolers). It answers:

- How much revenue are we making, and is it on target?
- Which product categories drive the business?
- Are margins, return rates and customer satisfaction within goal?
- How does each year compare month by month?

![Executive Summary](imagens/Executive%20Summary.png)

## Dataset

Seven CSV files in [`TechStore_CSV/`](TechStore_CSV), covering **Jan 2022 – Dec 2024**.

| Table | Rows | Description |
|---|---:|---|
| `fact_sales` | 5,000 | Order lines (2,004 orders): price, cost, discount, margin, channel, payment, region |
| `fact_returns` | 400 | Returns linked to a sale: reason, status, refund amount, days to return |
| `fact_support` | 600 | Support tickets: category, priority, status, satisfaction score (1–5) |
| `fact_inventory` | 60 | Stock snapshot per product: stock, reorder point, units sold YTD |
| `dim_products` | 60 | Product catalog: category, brand, supplier, price, cost |
| `dim_customers` | 500 | Customers: city, state, segment, status, preferred channel |
| `dim_suppliers` | 19 | Suppliers: country, lead time, payment terms |

All values are in Brazilian Real (R$). The CSVs use `.` as the decimal separator.

## Data model

Star schema with a dedicated `Calendar` table (Year › Quarter › Month hierarchy, months
sorted by number) and a `_measures` table holding all DAX measures, grouped by prefix:

| Prefix | Area | Examples |
|---|---|---|
| `REV_` | Sales | Total Revenue, Gross Margin %, Total Orders |
| `RET_` | Returns | Return Rate % |
| `SUP_` | Support | Average Satisfaction Score |
| `TIME_` | Time intelligence | Revenue YTD |
| `TGT_` | Targets | Revenue, Gross Margin, Return Rate and Satisfaction targets |

## Report — Executive Summary

- **KPI cards:** Revenue, Gross Margin %, Total Orders, Return Rate %, each against a target
- **Revenue Trend:** monthly revenue, one line per year
- **Revenue by Category:** category share of revenue
- **Revenue YTD vs Target:** gauge
- **Customer Satisfaction:** average score against the 4.0 goal
- **Slicers:** Year, Quarter, Month

## How to open

1. Install [Power BI Desktop](https://powerbi.microsoft.com/desktop/).
2. Open `Project_TechStore.pbix`.
3. If the data source path differs on your machine, go to
   *Transform data → Data source settings* and point it to the `TechStore_CSV` folder,
   then click **Refresh**.

## Tools

Power BI Desktop · Power Query (M) · DAX · Git

## License

[MIT](LICENSE)
