# Sales, Fulfilment & Marketing Data Model and Dashboard (Power BI)

A multi-fact star schema built in Power BI from a raw Excel workbook. This project focuses on the **data model**: source data, Power Query preparation, relationships, row-level security and DAX calculations. A one-page report (`Page 1`) on top of the model shows headline sales, orders and customer figures and sales against target by period. The business questions and insights below come only from that page.

**Tools:** Power BI Desktop · Power Query (M) · DAX · Excel

## Business problem

Sales, fulfilment and marketing data sits in separate operational sheets with text keys, placeholder rows and duplicated attributes. Leadership cannot easily see whether sales are on target, how many customers are buying, or how long orders take to be paid. This project integrates the sheets into one governed model and presents the key figures on a dashboard, with each regional manager seeing only their own region.

## Business questions

Each question is answered by an element on Page 1.

| # | Business question | Dashboard element |
|---|---|---|
| 1 | How much have we sold, and how many orders and units is that? | `total_sales` and `total_orders` cards; Sum of `quantity` in the table |
| 2 | Are we hitting our sales targets? | Table: Sum of `line_total` against Sum of `target_revenue` |
| 3 | How does sales performance change by year, quarter, month and day? | Table with the date hierarchy |
| 4 | How many of our customers are actually buying? | `total_active_customers` and `base_total_customers` cards |
| 5 | How long does it take to get paid? | `avg_order_to_pay` card |

## Report page (Page 1)

The report has one page with five cards and one table. There are no slicers or charts yet.

| Element | Field | Value |
|---|---|---|
| Card | `total_sales` | 526,644 |
| Card | `total_orders` | 80 |
| Card | `total_active_customers` | 47 |
| Card | `base_total_customers` | 60 |
| Card | `avg_order_to_pay` | 32.9 days |
| Table | Year, Quarter, Month, Day with Sum of `line_total`, Sum of `target_revenue` and Sum of `quantity` | Totals: 526,644 sales, 552,000 target, 1,095 units |

## Key insights

All figures come from the Page 1 cards and table, with the table read at year and month level. A figure that is divided from two dashboard values is marked as calculated.

1. **Sales are 4.6% below target.** The table totals 526,644 in sales against a target of 552,000, which is 95.4% attainment and a shortfall of 25,356.
2. **Almost every month with sales missed its target.** 19 of the 20 months with sales fell short, by between 2.2% and 7.0% (attainment ranged from 93.0% to 97.8%). Only June 2026 beat its target (6,133 against 6,000, or 102.2%). Because the misses are small and steady, the targets may be set slightly above what the business achieves.
3. **Both years finish near 95% of target.** 2025 reached 266,271 against 280,000 (95.1%). 2026, which has sales through October, reached 260,373 against 272,000 (95.7%).
4. **Monthly sales swing widely.** The best months are April 2025 (47,073), July 2026 (44,634) and December 2025 (43,333). The weakest are June 2026 (6,133), March 2026 (6,742) and September 2025 (7,556).
5. **Customer reach is 47 of 60.** 78% of customers (calculated) have placed an order, leaving 13 customers with none. The 80 orders and 1,095 units give an average of 6,583 per order (calculated).
6. **It takes about a month to get paid.** The `avg_order_to_pay` card shows 32.9 days from order to final payment.

## Recommendations

1. **Review how sales targets are set.** Sales missed the target in 19 of 20 months, by about 5% on average. Check whether the targets are realistic or whether the shortfall points to an execution gap.
2. **Win back the 13 customers who have not ordered.** They are 22% of the customer base. Target them with outreach to lift sales without having to find new customers.
3. **Look at why payment takes 32.9 days.** Split the order-to-pay figure into invoicing and payment time, using the fulfilment measures already in the model, and set a collection target.

## Data notes

- The workbook is sample data built to mimic messy operational systems, so these values illustrate the model and are not real business results.
- Months with no sales rows (January 2025, August 2025, and November and December 2026) do not appear as zero rows in the table. In total, 20 months have both sales and targets.
- Ten sales lines fall in October 2026, after the data snapshot date, so October 2026 is a partial month.
- Whether `total_sales` is before or after discount is not confirmed. `line_total` equals quantity times unit price on every line, even though `discount_percentage` is present on 99 lines.
- Region, shipping, billing and marketing measures exist in the model but are not on Page 1, so they are not used in the insights above.

## At a glance

| | |
|---|---|
| Source | 1 Excel workbook, 21 sheets |
| Staging queries | 22 (20 from sheets, 1 combined orders query, 1 embedded lookup) |
| Model tables | 7 dimensions, 6 facts, 1 security table, 1 measure table |
| Relationships | 23 (18 active, 5 inactive for role-playing dates and geography) |
| Measures | 39 in 4 display folders |
| Security | Region-based row-level security (RLS) |
| Date range | 1 Jan 2025 to 31 Dec 2026 (730 days) |

## Repository contents

| File | Description |
|---|---|
| `dataset.xlsx` | Raw source data (21 sheets) |
| `data_modeling_project.pbix` | Power BI file containing the queries, model, relationships, RLS and measures |
| `README.md` | This document |

## Source data

The workbook mimics operational systems and is deliberately messy: text keys, placeholder rows, duplicated attributes and wide-format tables.

| Domain | Sheets |
|---|---|
| Customers | `CUST_MASTER`, `customer_contacts`, `user_details`, `Address`, `cities`, `regions` |
| Products | `products`, `subcategories` |
| Orders | `ORDERS_2025`, `ORDERS_2026`, `order_line_items` |
| Fulfilment and payment | `shipments`, `INVOICES`, `invoice_lines`, `payments` |
| Marketing | `CAMPAIGN_LOG`, `campaign_skus` |
| Planning and stock | `sales_targets`, `inventory` |
| Reference and security | `exchange_rates`, `security` |

`channels` is a small lookup table embedded directly in Power Query. `regions`, `exchange_rates` and `invoice_lines` are staged but not yet used by the model.

## Data preparation (Power Query)

Queries follow two layers. All raw sheets sit in a `01_Stage` group with types set and no business logic. The model tables then reference the staging queries, so a source change is fixed in one place.

| Model table | What the query does |
|---|---|
| `dim_customer` | Joins customer master to primary contact email (primary contacts only), credit limit and phone, then street, city and region through address and city lookups. Removes placeholder customer `9999` and technical columns (`hash_key`, `source_id`). |
| `dim_product` | Removes placeholder product code `ZZZ-000` and rows with no brand. Splits the `Category\|Subcategory` text to derive `category`, and adds a `product_key` surrogate key. |
| `dim_geo` | Distinct cities with region and a `geo_key` surrogate key. |
| `dim_order_flags` | Combines `ORDERS_2025` and `ORDERS_2026`, keeps only channel, status and priority, removes duplicates, adds `flag_key` and looks up the channel name. |
| `dim_order` | Distinct order IDs. |
| `dim_campaign` | Distinct campaign attributes (name, channel, dates, budget) from the campaign log, with `campaign_key`. |
| `fact_sales` | One row per order line. Replaces text fields with surrogate keys for customer, product, flag, ship-to city and bill-to city. |
| `fact_order_process` | One row per order. Joins shipments, invoices and payments, then groups to first ship date, last delivery date, invoice date and first and last payment dates. |
| `fact_inventory` | Unpivots monthly columns into product-by-month rows and looks up `product_key`. |
| `fact_campaign_spend` | Daily impressions, clicks and spend per campaign. |
| `fact_promotion_coverage` | Splits the comma-separated promoted SKU list into one row per campaign and product. |
| `fact_sales_targets` | Monthly revenue targets, renamed to the model's naming standard. |
| `security` | User email to region mapping, loaded directly. |

Column names are standardised to `snake_case`.

### Table grain

| Table | Grain | Rows |
|---|---|---|
| `fact_sales` | One order line | 200 |
| `fact_order_process` | One order (accumulating snapshot of milestone dates) | 80 |
| `fact_inventory` | Product by month (2025) | 288 |
| `fact_campaign_spend` | Campaign by day | 140 |
| `fact_promotion_coverage` | Campaign by promoted product (bridge) | 30 |
| `fact_sales_targets` | Month | 20 |
| `dim_customer` | Customer | 60 |
| `dim_product` | Product | 60 |
| `dim_geo` | City | 20 |
| `dim_order` | Order | 80 |
| `dim_order_flags` | Channel, status and priority combination | 14 |
| `dim_campaign` | Campaign | 6 |
| `dim_date` | Day (`CALENDARAUTO()`) | 730 |

### Relationships

| From (many side) | To (one side) | Active |
|---|---|---|
| `fact_sales[customer_id]` | `dim_customer[customer_id]` | Yes |
| `fact_sales[product_key]` | `dim_product[product_key]` | Yes |
| `fact_sales[flag_key]` | `dim_order_flags[flag_key]` | Yes |
| `fact_sales[order_id]` | `dim_order[order_id]` | Yes |
| `fact_sales[order_date]` | `dim_date[Date]` | Yes |
| `fact_sales[ship_to_city_key]` | `dim_geo[geo_key]` | Yes |
| `fact_sales[bill_to_city_key]` | `dim_geo[geo_key]` | No |
| `fact_order_process[order_id]` | `dim_order[order_id]` | Yes |
| `fact_order_process[customer_id]` | `dim_customer[customer_id]` | Yes |
| `fact_order_process[flag_key]` | `dim_order_flags[flag_key]` | Yes |
| `fact_order_process[order_date]` | `dim_date[Date]` | Yes |
| `fact_order_process[ship_date]`, `[delivery_date]`, `[invoice_date]`, `[last_pay_date]` | `dim_date[Date]` | No (four relationships) |
| `fact_inventory[product_key]` | `dim_product[product_key]` | Yes |
| `fact_inventory[month]` | `dim_date[Date]` | Yes |
| `fact_campaign_spend[campaign_key]` | `dim_campaign[campaign_key]` | Yes |
| `fact_promotion_coverage[campaign_key]` | `dim_campaign[campaign_key]` | Yes |
| `fact_promotion_coverage[product_key]` | `dim_product[product_key]` | Yes |
| `fact_campaign_spend[date]` | `dim_date[Date]` (one-to-one) | Yes |
| `fact_sales_targets[date]` | `dim_date[Date]` (one-to-one) | Yes |
| `dim_customer[region]` | `security[region]` | Yes |

### Modeling patterns used

- **Star schema with several fact tables.** Sales, order process, inventory, marketing and targets share the conformed dimensions `dim_date`, `dim_product`, `dim_customer` and `dim_order_flags`.
- **Junk dimension.** `dim_order_flags` holds the low-cardinality channel, status and priority combinations, keeping them out of the fact tables.
- **Surrogate keys.** Products, cities, flags and campaigns get integer keys, replacing text joins in the facts.
- **Role-playing dimensions.** `fact_order_process` has five dates against one `dim_date`, and `fact_sales` has two cities against one `dim_geo`. One relationship is active and the rest are inactive, activated in DAX with `USERELATIONSHIP`.
- **Accumulating snapshot.** `fact_order_process` holds one row per order with a column for each lifecycle milestone, which supports cycle-time measures.
- **Bridge table.** `fact_promotion_coverage` resolves the many-to-many relationship between campaigns and products.
- **Thin conformed dimension.** `dim_order` links the order-line and order-process facts at order level.
- **Wide-to-long reshaping.** Monthly inventory columns are unpivoted so the fact table can relate to the date table.

### Row-level security

A role filters `dim_customer` with:

```dax
[region] = LOOKUPVALUE(security[region], security[user_email], USERPRINCIPALNAME())
```

Regional filtering flows to `fact_sales` and `fact_order_process` through their customer relationships. Test it with Modeling > View as.

## DAX calculations

**Calculated columns**

| Column | Definition |
|---|---|
| `fact_order_process[order_to_pay]` | `DATEDIFF(order_date, last_pay_date, DAY)` |
| `dim_date[Year]`, `dim_date[Month]` | `YEAR()` and `MONTH()` of `Date` |

**Measures** live in a dedicated `_measures` table and are organised in display folders.

| Folder | Measures |
|---|---|
| Core (5) | `total_sales`, `total_orders`, `total_active_customers`, `base_total_customers`, `avg_order_to_pay` |
| 1 Sales (11) | `units_sold`, `avg_order_value`, `gross_sales_at_list`, `discount_value`, `discount_pct`, `total_cost`, `gross_margin`, `gross_margin_pct`, `sales_ytd`, `sales_prior_year`, `sales_yoy_pct` |
| 2 Targets (4) | `sales_target`, `target_variance`, `target_attainment_pct`, `target_status` |
| 3 Fulfilment (8) | `orders_tracked`, `avg_order_to_ship_days`, `avg_ship_to_delivery_days`, `avg_order_to_invoice_days`, `avg_invoice_to_pay_days`, `unshipped_orders`, `unpaid_orders`, `unpaid_order_pct` |
| 4 Marketing (11) | `total_spend`, `total_impressions`, `total_clicks`, `ctr_pct`, `cost_per_click`, `cost_per_1000_impressions`, `campaign_budget`, `budget_utilization_pct`, `campaigns_running`, `promoted_product_sales`, `promoted_sales_per_spend` |

Examples:

```dax
gross_margin_pct = DIVIDE([total_sales] - [total_cost], [total_sales])

target_attainment_pct = DIVIDE([total_sales], SUM(fact_sales_targets[target_revenue]))

promoted_product_sales =
SUMX(dim_campaign,
    VAR prods = SELECTCOLUMNS(RELATEDTABLE(fact_promotion_coverage), "k", fact_promotion_coverage[product_key])
    RETURN CALCULATE([total_sales],
        TREATAS(prods, dim_product[product_key]),
        DATESBETWEEN(dim_date[Date], dim_campaign[start_date], dim_campaign[end_date])))
```

## How to open the model

1. Open `data_modeling_project.pbix` in Power BI Desktop.
2. The queries point to the original file location. Go to Transform data > Data source settings > Change Source and select your local copy of `dataset.xlsx`.
3. If refresh fails with a privacy firewall error (`Formula.Firewall`), go to File > Options > Current File > Privacy and choose "Ignore the Privacy Levels".
4. Open Model view to see the relationships, and Modeling > View as to test row-level security.

## Possible next steps

- Add charts to the dashboard: monthly sales against target as a line chart, sales by region, and a fulfilment page for shipping, invoicing and payment times.
- Add slicers for year, region and product category so the cards and table can be filtered.
