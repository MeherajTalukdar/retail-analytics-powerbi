# retail-analytics-powerbi-

# 📊 Power BI — 20 Real-World Business Problems


## 🗂️ Project Overview

| Domain | Problems | Focus |
|---|---|---|
| 📦 Sales & Revenue | #01 – #10 | GMV, forecasting, LTV, pipeline |

| 📣 Marketing & E-commerce | #11 – #20 | ROI, cohorts, NPS, A/B testing |

---

## 🗃️ Data Model

Star schema with 12 tables built to reflect a real e-commerce analytics environment.

| Table | Rows | Description |
|---|---|---|
| `orders` | 10,000 | Core fact table — sales, returns, customer & marketing analytics |
| `campaigns` | 300 | Marketing campaign spend and performance |
| `calendar` | — | Date dimension (2022–2024) — all time intelligence DAX |


## 📐 DAX Measures — Problems #01 to #10 (Sales & Revenue)

---

### #01 — Monthly Revenue Tracking

**Business question:** Show total net revenue per product category per month, compared to the target — flag red/green.

```dax
Total Net Revenue =
SUM(orders[NetRevenue_EUR])

Total Gross Revenue =
SUM(orders[GrossRevenue_EUR])

Revenue vs Target =
[Total Net Revenue] - SUM(budget[Revenue_Target_EUR])

Revenue vs Target % =
DIVIDE(
    [Revenue vs Target],
    SUM(budget[Revenue_Target_EUR])
)
```

> 💡 Use a Matrix visual with `calendar[MonthName]` on columns and `ProductCategory` on rows. Apply conditional formatting on `Revenue vs Target %` — red below 0, green above.

---

### #02 — Sales Rep Performance Ranking

**Business question:** Rank all sales reps by quarterly revenue and flag the bottom 20%.

```dax
Rep Revenue =
CALCULATE(SUM(orders[NetRevenue_EUR]))

Sales Rep Rank =
RANKX(
    ALL(orders[SalesRep]),
    [Rep Revenue],
    ,
    DESC,
    DENSE
)

Bottom 20% Flag =
VAR TotalReps  = DISTINCTCOUNT(orders[SalesRep])
VAR Threshold  = CEILING(TotalReps * 0.2, 1)
RETURN
    IF(
        [Sales Rep Rank] >= TotalReps - Threshold + 1,
        "⚠️ Bottom 20%",
        "✅ OK"
    )
```

---

### #03 — Price Elasticity Analysis

**Business question:** How did a discount affect sales volume across product categories?

```dax
Avg Unit Price EUR =
AVERAGE(orders[UnitPrice_EUR])

Total Quantity Sold =
SUM(orders[Quantity])

Avg Discount % =
AVERAGE(orders[DiscountPct])

Gross Margin % =
DIVIDE(
    SUM(orders[GrossProfit_EUR]),
    SUM(orders[GrossRevenue_EUR])
)
```

> 💡 Build a scatter chart: `Avg Discount %` on X-axis, `Total Quantity Sold` on Y-axis, `ProductCategory` on legend. Add a trend line to visualise the price-volume relationship.

---

### #04 — Year-over-Year Revenue Growth

**Business question:** YoY revenue growth % per product category for the board presentation.

```dax
Revenue PY =
CALCULATE(
    [Total Net Revenue],
    SAMEPERIODLASTYEAR(calendar[Date])
)

YoY Revenue Growth % =
DIVIDE(
    [Total Net Revenue] - [Revenue PY],
    [Revenue PY]
)

YoY Growth (Safe) =
IF(
    ISBLANK([Revenue PY]),
    BLANK(),
    DIVIDE([Total Net Revenue] - [Revenue PY], [Revenue PY])
)
```

---

### #05 — Revenue Forecast (3-Month Moving Average)

**Business question:** 3-month rolling revenue average for warehouse staffing demand planning.

```dax
Revenue 3M Rolling Avg =
AVERAGEX(
    DATESINPERIOD(
        calendar[Date],
        LASTDATE(calendar[Date]),
        -3,
        MONTH
    ),
    CALCULATE([Total Net Revenue])
)

Revenue Prior Month =
CALCULATE(
    [Total Net Revenue],
    DATEADD(calendar[Date], -1, MONTH)
)

MoM Revenue Change % =
DIVIDE(
    [Total Net Revenue] - [Revenue Prior Month],
    [Revenue Prior Month]
)
```

---

### #06 — Product Return Rate Analysis

**Business question:** Return rate per SKU to identify high-return products draining profit.

```dax
Total Sales Orders =
CALCULATE(
    COUNTROWS(orders),
    orders[TransactionType] = "Sale"
)

Total Returns =
CALCULATE(
    COUNTROWS(orders),
    orders[TransactionType] = "Return"
)

Return Rate % =
DIVIDE(
    [Total Returns],
    [Total Sales Orders] + [Total Returns]
)

Return Revenue Lost =
CALCULATE(
    SUM(orders[NetRevenue_EUR]),
    orders[TransactionType] = "Return"
)
```

---

### #07 — Customer Lifetime Value (LTV) Segmentation

**Business question:** Segment customers into Bronze / Silver / Gold / Platinum tiers by total spend.

```dax
Customer Total Spend =
CALCULATE(
    SUM(orders[NetRevenue_EUR]),
    ALLEXCEPT(orders, orders[CustomerID])
)

LTV Tier =
VAR Spend = [Customer Total Spend]
RETURN
    SWITCH(
        TRUE(),
        Spend >= 1000, "🥇 Platinum",
        Spend >= 500,  "🥈 Gold",
        Spend >= 200,  "🥉 Silver",
        "🔵 Bronze"
    )

Customers per Tier =
CALCULATE(
    DISTINCTCOUNT(orders[CustomerID]),
    ALLEXCEPT(orders, orders[CustomerSegment])
)
```

---

### #08 — Cross-Sell / Upsell Opportunity

**Business question:** Find customers who bought Shoes but never bought Clothing — target for cross-sell campaign.

```dax
Shoes Customers =
CALCULATETABLE(
    VALUES(orders[CustomerID]),
    orders[ProductCategory] = "Shoes"
)

Clothing Customers =
CALCULATETABLE(
    VALUES(orders[CustomerID]),
    orders[ProductCategory] = "Clothing"
)

Cross-Sell Opportunity Count =
VAR ShoesBuyers    = CALCULATETABLE(VALUES(orders[CustomerID]), orders[ProductCategory] = "Shoes")
VAR ClothingBuyers = CALCULATETABLE(VALUES(orders[CustomerID]), orders[ProductCategory] = "Clothing")
RETURN
    COUNTROWS(EXCEPT(ShoesBuyers, ClothingBuyers))

Avg Categories per Customer =
DIVIDE(
    COUNTROWS(orders),
    DISTINCTCOUNT(orders[CustomerID])
)
```

---

### #09 — Sales Pipeline Conversion Rate

**Business question:** Track how each acquisition channel converts — which channel brings the highest-value orders?

```dax
Orders by Channel =
CALCULATE(
    COUNTROWS(orders),
    ALLEXCEPT(orders, orders[AcquisitionChannel])
)

Channel Revenue Share % =
DIVIDE(
    CALCULATE(SUM(orders[NetRevenue_EUR])),
    CALCULATE(SUM(orders[NetRevenue_EUR]), ALL(orders[AcquisitionChannel]))
)

Avg Order Value by Channel =
DIVIDE(
    SUM(orders[NetRevenue_EUR]),
    COUNTROWS(orders)
)
```

---

### #10 — Break-Even Analysis

**Business question:** How many units must be sold to cover fixed + variable costs before a new product launch?

```dax
Avg Selling Price =
AVERAGE(orders[UnitPrice_EUR])

Avg COGS per Unit =
DIVIDE(
    SUM(orders[COGS_EUR]),
    SUM(orders[Quantity])
)

Contribution Margin per Unit =
[Avg Selling Price] - [Avg COGS per Unit]

Contribution Margin % =
DIVIDE(
    [Contribution Margin per Unit],
    [Avg Selling Price]
)

Break Even Units =
DIVIDE(
    'Fixed Cost Parameter'[Fixed Cost Parameter Value],
    [Contribution Margin per Unit]
)

Break Even Revenue =
[Break Even Units] * [Avg Selling Price]
```

> 💡 `Fixed Cost Parameter` is a What-If Parameter slicer — users drag it to model different cost scenarios interactively without changing any data.

---

## 📐 DAX Measures — Problems #11 to #20 (Marketing & E-commerce)

> These measures use both the `orders` table and the `campaigns` table.

---

### #11 — Campaign ROI Analysis

**Business question:** Measure ROI for each paid campaign and rank best performers across Google, Meta, TikTok.

```dax
Campaign ROI % =
DIVIDE(
    SUM(campaigns[Revenue_EUR]) - SUM(campaigns[Spend_EUR]),
    SUM(campaigns[Spend_EUR])
)

Campaign Profit =
SUM(campaigns[Revenue_EUR]) - SUM(campaigns[Spend_EUR])

Campaign Rank by ROI =
RANKX(
    ALL(campaigns[CampaignName]),
    [Campaign ROI %],
    ,
    DESC,
    DENSE
)

Revenue per EUR Spent =
DIVIDE(
    SUM(campaigns[Revenue_EUR]),
    SUM(campaigns[Spend_EUR])
)
```

---

### #12 — Email Campaign Performance Dashboard

**Business question:** Open rate, CTR, conversion rate, and revenue per email per customer segment.

```dax
Email Open Rate % =
DIVIDE(
    SUM(campaigns[EmailOpens]),
    SUM(campaigns[Impressions])
)

Click Through Rate % =
DIVIDE(
    SUM(campaigns[Clicks]),
    SUM(campaigns[EmailOpens])
)

Campaign Conversion Rate % =
DIVIDE(
    SUM(campaigns[Conversions]),
    SUM(campaigns[Clicks])
)

Revenue per Conversion =
DIVIDE(
    SUM(campaigns[Revenue_EUR]),
    SUM(campaigns[Conversions])
)

Cost per Conversion =
DIVIDE(
    SUM(campaigns[Spend_EUR]),
    SUM(campaigns[Conversions])
)
```

---

### #13 — Customer Cohort Retention Analysis

**Business question:** Of customers acquired in January, what % are still buying in Feb, Mar, Apr…?

```dax
First Order Date =
CALCULATE(
    MIN(orders[OrderDate]),
    ALLEXCEPT(orders, orders[CustomerID])
)

Acquisition Month =
FORMAT(CALCULATE(MIN(orders[OrderDate]), ALLEXCEPT(orders, orders[CustomerID])), "YYYY-MM")

Active Customers =
DISTINCTCOUNT(orders[CustomerID])

Cohort Retention % =
DIVIDE(
    DISTINCTCOUNT(orders[CustomerID]),
    CALCULATE(
        DISTINCTCOUNT(orders[CustomerID]),
        FILTER(
            ALL(calendar),
            calendar[YearMonth] = MIN(calendar[YearMonth])
        )
    )
)
```

> 💡 Build a cohort grid in a Matrix visual — acquisition month on rows, activity month on columns, `Cohort Retention %` as values. Apply a heat map via conditional formatting (green = high retention, red = churn).

---

### #14 — Promo Code Usage & Discount Impact

**Business question:** How many orders used a promo code, how much discount was given, and what was the net revenue impact?

```dax
Promo Orders =
CALCULATE(
    COUNTROWS(orders),
    orders[PromoCode] <> "NONE"
)

Promo Order Rate % =
DIVIDE(
    [Promo Orders],
    COUNTROWS(orders)
)

Total Discount Given =
SUM(orders[DiscountAmt_EUR])

Discount as % of Gross Revenue =
DIVIDE(
    SUM(orders[DiscountAmt_EUR]),
    SUM(orders[GrossRevenue_EUR])
)

Revenue Retained After Discount =
DIVIDE(
    SUM(orders[NetRevenue_EUR]),
    SUM(orders[GrossRevenue_EUR])
)
```

---

### #15 — Product Category Mix Analysis

**Business question:** What % of revenue comes from each category, and how has the mix shifted year-over-year?

```dax
Category Revenue Share % =
DIVIDE(
    SUM(orders[NetRevenue_EUR]),
    CALCULATE(SUM(orders[NetRevenue_EUR]), ALL(orders[ProductCategory]))
)

Category Revenue PY =
CALCULATE(
    SUM(orders[NetRevenue_EUR]),
    SAMEPERIODLASTYEAR(calendar[Date])
)

Category Mix Shift YoY =
[Category Revenue Share %] -
CALCULATE(
    [Category Revenue Share %],
    SAMEPERIODLASTYEAR(calendar[Date])
)
```

---

### #16 — Customer Acquisition Cost (CAC) by Channel

**Business question:** How much does it cost to acquire one new customer per marketing channel?

```dax
Total New Customers =
SUM(campaigns[NewCustomers])

CAC by Channel =
DIVIDE(
    SUM(campaigns[Spend_EUR]),
    SUM(campaigns[NewCustomers])
)

LTV to CAC Ratio =
DIVIDE(
    [Customer Total Spend],
    [CAC by Channel]
)
```

> 💡 LTV:CAC ratio above 3× is considered healthy in e-commerce. Use a KPI card with conditional formatting to flag channels below this threshold.

---

### #17 — A/B Test Results Analysis

**Business question:** Was the conversion rate difference between two campaign variants statistically meaningful?

```dax
Variant A Conversion Rate % =
CALCULATE(
    DIVIDE(SUM(campaigns[Conversions]), SUM(campaigns[Clicks])),
    campaigns[CampaignType] = "Performance"
)

Variant B Conversion Rate % =
CALCULATE(
    DIVIDE(SUM(campaigns[Conversions]), SUM(campaigns[Clicks])),
    campaigns[CampaignType] = "Influencer"
)

Conversion Rate Lift % =
DIVIDE(
    [Variant B Conversion Rate %] - [Variant A Conversion Rate %],
    [Variant A Conversion Rate %]
)
```

> 💡 Statistical significance testing (Z-test) is best completed in Excel or Python before importing results. In Power BI, focus on visualising the lift, confidence interval, and sample size per variant.

---

### #18 — Seasonal Sales Pattern Analysis

**Business question:** Which product categories peak in which months — used to plan stock buying cycles.

```dax
Monthly Category Revenue =
CALCULATE(
    SUM(orders[NetRevenue_EUR]),
    ALLEXCEPT(orders, orders[ProductCategory], calendar[Month], calendar[Year])
)

Annual Avg Monthly Revenue =
CALCULATE(
    AVERAGEX(
        VALUES(calendar[Month]),
        CALCULATE(SUM(orders[NetRevenue_EUR]))
    ),
    ALL(calendar[Month])
)

Seasonal Index =
DIVIDE(
    [Monthly Category Revenue],
    [Annual Avg Monthly Revenue]
)
```

> 💡 A seasonal index above 1.0 means that month outperforms the annual average. Build a heat map matrix — categories on rows, months on columns — and apply conditional formatting. A score above 1.2 signals a clear seasonal peak.

---

### #19 — Customer Complaint / NPS Analysis

**Business question:** Calculate NPS score and detractor % by product category.

```dax
NPS Promoters =
CALCULATE(
    COUNTROWS(orders),
    orders[NPS_Score] >= 9,
    NOT(ISBLANK(orders[NPS_Score]))
)

NPS Passives =
CALCULATE(
    COUNTROWS(orders),
    orders[NPS_Score] >= 7,
    orders[NPS_Score] <= 8,
    NOT(ISBLANK(orders[NPS_Score]))
)

NPS Detractors =
CALCULATE(
    COUNTROWS(orders),
    orders[NPS_Score] <= 6,
    NOT(ISBLANK(orders[NPS_Score]))
)

Total NPS Responses =
[NPS Promoters] + [NPS Passives] + [NPS Detractors]

NPS Score =
DIVIDE([NPS Promoters] - [NPS Detractors], [Total NPS Responses]) * 100

Detractor Rate % =
DIVIDE([NPS Detractors], [Total NPS Responses])
```

---

### #20 — Influencer / Affiliate Performance Tracking

**Business question:** Revenue, orders, and effective cost per acquisition per campaign platform.

```dax
Influencer Revenue =
CALCULATE(
    SUM(campaigns[Revenue_EUR]),
    campaigns[CampaignType] = "Influencer"
)

Influencer Spend =
CALCULATE(
    SUM(campaigns[Spend_EUR]),
    campaigns[CampaignType] = "Influencer"
)

Influencer ROI % =
DIVIDE(
    [Influencer Revenue] - [Influencer Spend],
    [Influencer Spend]
)

Cost per New Customer by Platform =
DIVIDE(
    SUM(campaigns[Spend_EUR]),
    SUM(campaigns[NewCustomers])
)

Platform Rank by Revenue =
RANKX(
    ALL(campaigns[Platform]),
    SUM(campaigns[Revenue_EUR]),
    ,
    DESC,
    DENSE
)
```

---

## 🛠️ DAX Functions Reference

| Function | Used In |
|---|---|
| `RANKX` | #02, #41, #50 — ranking reps, campaigns, platforms |
| `SAMEPERIODLASTYEAR` | #04, #45 — YoY growth and category mix shift |
| `DATESINPERIOD` | #05 — 3-month rolling average |
| `DATEADD` | #05 — month-over-month comparison |
| `ALLEXCEPT` | #07, #43 — customer-level aggregation while keeping one filter |
| `EXCEPT` | #08 — cross-sell gap identification |
| `SWITCH(TRUE())` | #07 — multi-condition tier logic |
| `CALCULATETABLE` | #08 — virtual table filtering |
| `DIVIDE` | Throughout — safe division with blank handling |
| `ISBLANK` | #04, #19 — null-safe calculations |
| What-If Parameter | #10 — interactive break-even modelling |

---


```

---

## 👤 About

**Meheraj Talukdar** |  Junior Business Data Analyst | Düsseldorf, Germany
🔗 GitHub: [MeherajTalukdar](https://github.com/MeherajTalukdar) · 📊 Power BI · SQL · Excel · DAX
🎯 Target roles: Data Analyst · BI Analyst · Reporting Analyst · Controlling

