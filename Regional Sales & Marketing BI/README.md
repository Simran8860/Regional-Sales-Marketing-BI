# Regional Sales & Marketing BI Dashboard

A 4-page Power BI report analysing sales, marketing and operations for an Indian e-commerce business: **100K orders, ~25K customers, 151 marketing campaigns, Jan 2023 – Dec 2025**.

> **Dataset:** synthetic, generated for portfolio use, and **not included in this repository**. All findings describe this dataset, not a real company.

## Report pages

| Page | What it answers |
|---|---|
| **Summary** | Scope, key takeaways, definitions, page navigation |
| **Executive Overview** | Revenue, orders, AOV, profit, margin, high-ticket share, YoY growth, revenue by state and category |
| **Marketing & Customers** | Spend vs revenue, ROAS by channel, new vs returning customers, top campaigns |
| **Operations & Experience** | Delivery time, on-time %, return rate, ratings, refunds, payment mix |

A hidden tooltip page defines High-Ticket Share on hover.

## Headline numbers (delivered orders)

| Metric | Value |
|---|---|
| Revenue | ₹1.98bn |
| Orders | 80.7K |
| AOV | ₹24.6K |
| Profit / margin | ₹158.6M / 7.99% |
| Marketing spend | ₹187.1M |
| ROAS (campaign-attributed) | 1.93 |
| New customers | 21.8K |
| Repeat customer rate | 78.3% |
| On-time delivery | 57.1% |
| Return rate (orders) | 10.4% |
| Avg rating | 3.69 |

## Key findings

- **Growth came in 2025.** Delivered revenue was roughly flat in 2024 (about -2% vs 2023), then rose about 68% in 2025.
- **Q4 2025 is a volume surge.** Delivered orders went from about 2,800 in September to about 6,400 in December while AOV stayed flat (₹25–27K). New and returning customers both rose roughly 60–70%.
- **Marketing spend does not fully explain the surge.** October orders jumped with only ₹4.9M of spend, while August had ₹14.8M of spend and no matching jump.
- **ROAS varies widely by channel.** Direct/App (5.1), Organic Search (4.4) and Referral (4.1) lead. Social Media – Facebook (0.9) is below break-even, yet it and Paid Search are the two largest spend lines (₹37.9M and ₹36.3M).
- **Deliveries are steady but often late.** About 57% of shipments arrive on time, with no change during the Q4 surge.
- **Returns are concentrated.** About 10% of delivered orders have a return. At item level the overall rate is 4.7%, highest in Footwear (8.5%) and Fashion (7.4%).
- **Payment failures are small.** 5.1% of payments failed; success rate among attempted payments is 94.2%.

## Data model

Star-style model around `orders`, with a `DimDate` calendar and a dedicated `_Measures` table.

| Table | Rows | Grain |
|---|---|---|
| `orders` | 100,000 | One row per order |
| `order_items` | 254,331 | One row per order line |
| `customers` | 25,000 | One row per customer |
| `products` | 550 | One row per product |
| `payments` | 100,000 | One row per order payment |
| `returns` | 12,075 | One row per returned item |
| `shipments` | 89,681 | One row per shipment |
| `customer_reviews` | 77,530 | One row per review |
| `marketing_campaigns` | 151 | One row per campaign |

The source CSVs (about 70 MB) are not published. The measures in [`DAX/measures.md`](DAX/measures.md) show how each metric is calculated.

## Technical highlights

- **YoY with a base guard.** Months where the prior-year base is too small return blank, so a ramp-up month does not show +400%.
- **Inactive relationship + `USERELATIONSHIP`.** `marketing_campaigns` → `DimDate` would create an ambiguous path through `orders`, so the date link is inactive and activated inside the spend measure.
- **First-delivered-order logic** for new vs returning customers, as a calculated column plus `KEEPFILTERS`.
- **Item-level return rate** by category (returned items ÷ sold items), so category rates are true rates.
- **`KEEPFILTERS` on status filters**, so slicers still behave sensibly.
- Custom theme, page navigator, report-page tooltip, consistent slicer and card styling.

## Data checks performed

Measures were reconciled against the source CSVs outside Power BI. Notable catches:

- Headline cards originally counted cancelled, failed and returned orders (₹2.45bn gross vs ₹1.98bn delivered).
- `delayed_flag` matches the delivery dates in every row, but "Cancelled in Transit" shipments were being counted as on time.
- Return rate by category used all orders as the denominator, which understated every category.
- Payment success was reported as 82% because refunded and not-charged payments counted as non-success.

## Known limitations

- Marketing spend is counted in each campaign's start month, so spend is lumpy.
- Attributed revenue covers campaign-linked orders only (about 17% of orders) and uses order value including tax and shipping, while headline revenue excludes tax and shipping.
- Blended CAC divides all marketing spend by all new customers, including organic ones.
- `returns` contains rows for some orders that are not marked Returned (including cancelled and failed orders), which is a quirk of the synthetic data.
- The `customers.total_orders` column does not match the orders table and is not used.

## Repository structure

```
├── README.md
├── Regional Sales & Marketing BI.pdf    PDF export of the report
├── DAX/                                 measures and calculated columns
├── Report/                              Power BI report file
├── Screenshots/                         one image per page
└── .gitignore
```

## Tools

Power BI Desktop, DAX, Python (pandas) for reconciliation checks.

## Author

Simran Tyagi
