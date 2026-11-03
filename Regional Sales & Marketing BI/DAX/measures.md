# DAX measures and calculated columns

All measures live in the `_Measures` table unless noted. Revenue, profit and orders count **delivered orders only**.

## Date table (DimDate, calculated column)

```dax
Month Label = FORMAT(DimDate[Date], "MMM YY")
-- Sort by column: YearMonthSort
```

## Core sales

```dax
Total Revenue =
CALCULATE(
    SUM(order_items[item_revenue]),
    KEEPFILTERS(orders[order_status] = "Delivered")
)

Total Profit =
CALCULATE(
    SUM(order_items[profit]),
    KEEPFILTERS(orders[order_status] = "Delivered")
)

Total Orders =
CALCULATE(
    DISTINCTCOUNT(orders[order_id]),
    KEEPFILTERS(orders[order_status] = "Delivered")
)

AOV =
DIVIDE([Total Revenue], [Total Orders], 0)

Profit Margin =
DIVIDE([Total Profit], [Total Revenue])
```

## Growth (delivered order value)

```dax
Delivered Revenue =
CALCULATE(
    SUM(orders[final_amount]),
    orders[order_status] = "Delivered"
)

Delivered Item Revenue =
CALCULATE(
    SUM(order_items[item_revenue]),
    orders[order_status] = "Delivered"
)

Delivered Revenue PY =
CALCULATE(
    [Delivered Revenue],
    DATEADD(DimDate[Date], -1, YEAR)
)

-- Blank when the prior-year base is too small to compare (2023 ramp-up months)
Delivered Revenue YoY % =
VAR _PY = [Delivered Revenue PY]
RETURN
    IF(
        _PY < 30000000,
        BLANK(),
        DIVIDE([Delivered Revenue] - _PY, _PY)
    )
```

## High-ticket

```dax
-- TODO: paste your High-Ticket Threshold (P75) and High-Ticket Share measures here
```

## Marketing

The `marketing_campaigns` → `DimDate` relationship is inactive (an active one would create an ambiguous path via `orders`), so it is activated inside the measure.

```dax
Marketing Spend =
CALCULATE(
    SUM(marketing_campaigns[campaign_cost]),
    USERELATIONSHIP(marketing_campaigns[campaign_start_date], DimDate[Date])
)

Attributed Revenue =
CALCULATE(
    SUM(orders[final_amount]),
    orders[order_status] = "Delivered",
    FILTER(
        orders,
        NOT ISBLANK(orders[campaign_id])
            && orders[campaign_id] <> ""
    )
)

ROAS =
DIVIDE([Attributed Revenue], [Marketing Spend])

-- Blended: all marketing spend / all new customers (not only campaign-acquired)
Blended CAC =
DIVIDE([Marketing Spend], [New Customers])
```

## Customers

Calculated column on `orders`:

```dax
first_delivered_date =
CALCULATE(
    MIN(orders[order_date]),
    ALLEXCEPT(orders, orders[customer_id]),
    orders[order_status] = "Delivered"
)
```

Measures:

```dax
New Customers =
CALCULATE(
    DISTINCTCOUNT(orders[customer_id]),
    KEEPFILTERS(orders[order_status] = "Delivered"),
    FILTER(
        orders,
        orders[order_date] = orders[first_delivered_date]
    )
)

Purchasing Customers =
CALCULATE(
    DISTINCTCOUNT(orders[customer_id]),
    KEEPFILTERS(orders[order_status] = "Delivered")
)

-- Period-based: customers active in the period who were acquired earlier
Returning Customers =
[Purchasing Customers] - [New Customers]

-- Customers with 2+ delivered orders in the current filter
Repeat Customers =
CALCULATE(
    COUNTROWS(
        FILTER(
            ADDCOLUMNS(
                VALUES(orders[customer_id]),
                "OrderCount", CALCULATE(DISTINCTCOUNT(orders[order_id]))
            ),
            [OrderCount] > 1
        )
    ),
    KEEPFILTERS(orders[order_status] = "Delivered")
)

Repeat Customer Rate =
DIVIDE([Repeat Customers], [Purchasing Customers])
```

## Operations

```dax
Avg Delivery Days =
AVERAGE(shipments[delivery_days])

-- Excludes shipments cancelled in transit (never delivered)
On-Time Delivery % =
DIVIDE(
    CALCULATE(COUNTROWS(shipments),
        shipments[delayed_flag] = 0,
        shipments[delivery_status] <> "Cancelled in Transit"),
    CALCULATE(COUNTROWS(shipments),
        shipments[delivery_status] <> "Cancelled in Transit")
)

-- Order level: orders with a return / delivered + returned orders
Return Rate =
DIVIDE(
    CALCULATE(DISTINCTCOUNT(returns[order_id]),
        KEEPFILTERS(orders[order_status] IN {"Delivered", "Returned"})),
    CALCULATE(DISTINCTCOUNT(orders[order_id]),
        KEEPFILTERS(orders[order_status] IN {"Delivered", "Returned"}))
)

-- Item level: used for return rate by category
Return Rate (Items) =
DIVIDE(
    COUNTROWS(returns),
    COUNTROWS(order_items)
)

-- Full refunds on cancelled and returned orders (payments table)
Order Refunds =
SUM(payments[refund_amount])

-- Success / (Success + Failed); refunded, pending and not-charged are excluded
Payment Success % =
DIVIDE(
    CALCULATE(COUNTROWS(payments), payments[payment_status] = "Success"),
    CALCULATE(COUNTROWS(payments),
        payments[payment_status] IN {"Success", "Failed"})
)

Avg Rating =
AVERAGE(customer_reviews[rating])
```
