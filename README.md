# Sales, Fulfilment & Marketing Data Model (Power BI)

A multi-fact star schema built in Power BI from a raw Excel workbook. This document covers the **data modeling** only: source data, Power Query preparation, the semantic model, relationships, row-level security and DAX calculations.

**Tools:** Power BI Desktop · Power Query (M) · DAX · Excel

## Business problem

Sales, fulfilment and marketing data sits in separate operational sheets with text keys, placeholder rows and duplicated attributes. Leadership cannot answer cross-functional questions from the raw sheets, such as whether sales are on target, where orders get stuck between shipping and payment, or whether marketing spend is paying off. This project integrates the sheets into one governed model so those questions can be answered consistently, with each regional manager seeing only their own region.

## Business questions

Each question is answered by measures that already exist in the model.

| # | Business question | Measures | Fact table |
|---|---|---|---|
| 1 | How are sales and margin performing, and how do they compare with last year? | `total_sales`, `gross_margin`, `gross_margin_pct`, `sales_ytd`, `sales_prior_year`, `sales_yoy_pct` | `fact_sales` |
| 2 | How much revenue is being given away in discounts? | `gross_sales_at_list`, `discount_value`, `discount_pct` | `fact_sales` |
| 3 | Are we hitting our sales targets? | `sales_target`, `target_variance`, `target_attainment_pct`, `target_status` | `fact_sales_targets` |
| 4 | How quickly does an order move from order to cash, and where are the delays? | `avg_order_to_ship_days`, `avg_ship_to_delivery_days`, `avg_order_to_invoice_days`, `avg_invoice_to_pay_days`, `unshipped_orders`, `unpaid_orders`, `unpaid_order_pct` | `fact_order_process` |
| 5 | How efficient is marketing spend, and do promoted products sell? | `ctr_pct`, `cost_per_click`, `cost_per_1000_impressions`, `budget_utilization_pct`, `promoted_product_sales`, `promoted_sales_per_spend` | `fact_campaign_spend`, `fact_promotion_coverage` |

## Key insights

Values below come from the model's own measures, evaluated with no filters on the sample dataset.

1. **Sales are slightly below target.** Total sales are 526,644 across 80 orders and 1,095 units, with an average order value of 6,583. Against a target of 552,000, attainment is 95.4% (a variance of -25,356), and `target_status` returns "At risk". Gross margin is 194,440, or 36.9%.
2. **Sales are spread across regions, with Europe in front.** Europe leads at 167,364 (27 orders), followed by Middle East (117,005), Asia Pacific (103,847), North America (97,456) and Latin America (40,972). Gross margin is similar everywhere, between 35.5% and 37.9%. `total_active_customers` is 47 against a base of 60 customers.
3. **Shipping is fast, but billing and collection are slow.** Orders ship in 2.6 days on average and arrive 6.0 days later. Invoicing takes 11.2 days from the order and payment takes another 21.6 days after the invoice, so an order takes 32.9 days to be paid in full.
4. **A quarter of orders are unpaid, concentrated in the Middle East.** 20 of 80 orders (25.0%) have no payment recorded. In the Middle East the figure is 56.3% of orders, against 13.3% to 18.5% in the other regions.
5. **Marketing spend is large compared with the sales tied to promoted products.** Spend is 78,841 (97.3% of the 81,000 budget) for 7.0 million impressions and 223,744 clicks, a click-through rate of 3.19% at 0.35 per click. Promoted products sold 5,231 during their campaign windows, which is 0.01 per unit of spend. Only 3 of the 6 campaigns show promoted-product sales at all.
6. **Cost per click varies widely by campaign.** The Black Friday paid search campaign cost 1.84 per click and took the largest share of spend (28,065). The other five campaigns cost between 0.17 and 0.32 per click.

## Recommendations

1. **Focus on collections, not shipping.** Fulfilment is quick, while invoice-to-payment takes 21.6 days and 25.0% of orders are unpaid. Review payment follow-up, starting with the Middle East.
2. **Review marketing effectiveness.** Compare promoted-product sales with spend per campaign and revisit campaigns with no measurable promoted sales. Paid search click costs deserve a closer look.
3. **Track target attainment by month and region.** The overall result is "At risk", so use the target measures to find where the 25,356 shortfall comes from.

## Data notes

- The workbook is sample data built to mimic messy operational systems, so these values illustrate the model and are not real business results.
- `discount_value` and `discount_pct` (4.7%) assume recorded discounts. Because `line_total` equals quantity times unit price on every line, `total_sales` may be before discount. Confirm this with the data owner.
- `promoted_product_sales` shows correlation, not proven attribution, and overlapping campaigns can double count sales.

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

## Opening the model

1. Open `data_modeling_project.pbix` in Power BI Desktop.
2. The queries point to the original file location. Go to Transform data > Data source settings > Change Source and select your local copy of `dataset.xlsx`.
3. If refresh fails with a privacy firewall error (`Formula.Firewall`), go to File > Options > Current File > Privacy and choose "Ignore the Privacy Levels".
4. Open Model view to see the relationships, and Modeling > View as to test row-level security.

## Possible next steps

- Build report pages on top of the model, one per business question above.
- Use `exchange_rates` to add currency conversion. It is staged but not yet used by the model.
- Use `invoice_lines` for product-level billing analysis. It is staged but not yet used by the model.
