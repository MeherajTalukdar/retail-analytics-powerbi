# retail-analytics-powerbi


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

## 📐 Foundation Measures

```dax
Total Orders = 
DISTINCTCOUNT(fact_orders[OrderID])

Net Revenue = 
SUMX(
    fact_orders,
    fact_orders[Quantity] * fact_orders[UnitPrice_EUR]
    - fact_orders[DiscountAmt_EUR]
)

Gross Profit = 
SUM(fact_orders[GrossProfit_EUR])

Gross Margin % = 
DIVIDE([Gross Profit], [Net Revenue])
```

---

## 📐 DAX Measures — Problems #01 to #10 (Sales & Revenue)

### #01 — Real-Time Sales Performance Dashboard
```dax
Revenue vs Target = 
VAR target = 150000
RETURN [Net Revenue] - target
```

### #02 — MoM & YoY Revenue Trend
```dax
Prev Month Revenue = 
CALCULATE([Net Revenue], DATEADD(dim_calendar[Date], -1, MONTH))

MoM % Change = 
DIVIDE([Net Revenue] - [Prev Month Revenue], [Prev Month Revenue])

Prev Year Revenue = 
CALCULATE([Net Revenue], SAMEPERIODLASTYEAR(dim_calendar[Date]))

YoY % Change = 
DIVIDE([Net Revenue] - [Prev Year Revenue], [Prev Year Revenue])
```

### #03 — Product Return Rate Dashboard
```dax
Gross Orders = 
CALCULATE([Total Orders], fact_orders[TransactionType] = "Sale")

Return Orders = 
CALCULATE([Total Orders], fact_orders[TransactionType] = "Return")

Return Rate % = 
DIVIDE([Return Orders], [Gross Orders])
```

### #04 — Top N Product Rankings
```dax
Product Rank = 
RANKX(ALL(fact_orders[ProductName]), [Net Revenue], , DESC, Dense)

Top N Filter = 
IF([Product Rank] <= [TopN Value], 1, 0)
```
> 💡 Create a What-If Parameter: **Modeling → New Parameter → TopN**, Min=5, Max=50, Increment=5, Default=10. Add `[Top N Filter] = 1` as a visual-level filter on your bar chart.

### #05 — Revenue by Geography
> No new DAX needed — `[Net Revenue]` works directly. Set the `Country` column **Data Category = Country/Region** in Column tools.

### #06 — Sales Funnel & Conversion Rate
```dax
AOV = 
DIVIDE([Net Revenue], [Total Orders])

Orders per Customer = 
DIVIDE(
    [Total Orders],
    CALCULATE(DISTINCTCOUNT(fact_orders[CustomerID]), fact_orders[CustomerID] <> "GUEST")
)
```

### #07 — Discount & Promotion Impact
```dax
Total Discount = 
SUM(fact_orders[DiscountAmt_EUR])

Discount % of Revenue = 
DIVIDE([Total Discount], [Net Revenue] + [Total Discount])

Net Margin % = 
DIVIDE([Gross Profit], [Net Revenue])
```

### #08 — Revenue Forecasting
```dax
Rolling 3M Revenue = 
CALCULATE(
    [Net Revenue],
    DATESINPERIOD(dim_calendar[Date], LASTDATE(dim_calendar[Date]), -3, MONTH)
)
```
> 💡 For the forecast line: **Analytics Pane → Forecast → on**, set Forecast Length = 6 months, Confidence Interval = 95%.

### #09 — Basket Size & AOV Analysis
```dax
Avg Basket Size = 
DIVIDE(SUM(fact_orders[Quantity]), [Total Orders])
```

### #10 — Channel Mix & Attribution
```dax
Channel Revenue Share % = 
DIVIDE([Net Revenue], CALCULATE([Net Revenue], ALL(fact_orders[AcquisitionChannel])))
```

---

## 📐 DAX Measures — Problems #11 to #20 (Marketing & E-commerce)

> Before writing any DAX for #11–20, connect in Model view:
> - `fact_campaigns[StartDate]` → `dim_calendar[Date]` (active)
> - `fact_orders[OrderDate]` → `dim_calendar[Date]` (already done)
> - `fact_campaigns` and `fact_orders` remain unrelated to each other — correct star schema design.

### #11 — Customer Segmentation & LTV Dashboard
```dax
Active Customers = 
CALCULATE(DISTINCTCOUNT(fact_orders[CustomerID]), fact_orders[CustomerID] <> "GUEST")

Customer LTV = 
DIVIDE([Net Revenue], [Active Customers])

Avg Orders per Customer = 
DIVIDE([Total Orders], [Active Customers])

Recency Days = 
DATEDIFF(
    CALCULATE(MAX(fact_orders[OrderDate]), ALL(dim_calendar)),
    TODAY(),
    DAY
)
```

### #12 — Campaign Performance & ROI Dashboard
```dax
Total Campaign Spend = 
SUM(fact_campaigns[Spend_EUR])

Total Campaign Revenue = 
SUM(fact_campaigns[Revenue_EUR])

ROAS = 
DIVIDE([Total Campaign Revenue], [Total Campaign Spend])

CPA = 
DIVIDE([Total Campaign Spend], SUM(fact_campaigns[Conversions]))

Campaign ROI % = 
DIVIDE([Total Campaign Revenue] - [Total Campaign Spend], [Total Campaign Spend])

CTR % = 
DIVIDE(SUM(fact_campaigns[Clicks]), SUM(fact_campaigns[Impressions]))
```

### #13 — Customer Cohort Retention Dashboard
> Step 1 — Add this as a **calculated column** on `fact_orders` in Data view:
```dax
CohortMonth = 
FORMAT(
    CALCULATE(MIN(fact_orders[OrderDate]), ALLEXCEPT(fact_orders, fact_orders[CustomerID])),
    "YYYY-MM"
)
```
> Step 2 — Measures:
```dax
Cohort Size = 
CALCULATE(DISTINCTCOUNT(fact_orders[CustomerID]), fact_orders[CohortMonth] = MAX(fact_orders[CohortMonth]))

Retained Customers = 
DISTINCTCOUNT(fact_orders[CustomerID])

Retention Rate % = 
DIVIDE([Retained Customers], [Cohort Size])
```
> 💡 Matrix visual: `CohortMonth` on rows, `dim_calendar[YearMonth]` on columns, `[Retention Rate %]` as values. Apply conditional formatting → background colour scale (white → green) for the classic cohort heat map.

### #14 — Web & App Analytics (Platform Mix)
```dax
Mobile App Orders = 
CALCULATE([Total Orders], fact_orders[Platform] = "Mobile App")

Mobile App Share % = 
DIVIDE([Mobile App Orders], [Total Orders])
```

### #15 — Email & CRM Campaign Performance
```dax
Email Open Rate % = 
CALCULATE(
    DIVIDE(SUM(fact_campaigns[EmailOpens]), SUM(fact_campaigns[Impressions])),
    fact_campaigns[Platform] = "Email"
)

Email CTR % = 
CALCULATE(
    DIVIDE(SUM(fact_campaigns[Clicks]), SUM(fact_campaigns[EmailOpens])),
    fact_campaigns[Platform] = "Email"
)

Email Conversion Rate % = 
CALCULATE(
    DIVIDE(SUM(fact_campaigns[Conversions]), SUM(fact_campaigns[Clicks])),
    fact_campaigns[Platform] = "Email"
)

Email Unsubscribe Rate % = 
CALCULATE(
    DIVIDE(SUM(fact_campaigns[Unsubscribes]), SUM(fact_campaigns[Impressions])),
    fact_campaigns[Platform] = "Email"
)
```

### #16 — Customer Complaint & NPS Dashboard
```dax
NPS Promoters = 
CALCULATE(COUNTROWS(fact_orders), fact_orders[NPS_Score] >= 9, NOT ISBLANK(fact_orders[NPS_Score]))

NPS Detractors = 
CALCULATE(COUNTROWS(fact_orders), fact_orders[NPS_Score] <= 6, NOT ISBLANK(fact_orders[NPS_Score]))

NPS Responses = 
CALCULATE(COUNTROWS(fact_orders), NOT ISBLANK(fact_orders[NPS_Score]))

NPS Score = 
DIVIDE([NPS Promoters], [NPS Responses]) - DIVIDE([NPS Detractors], [NPS Responses])

Avg NPS Score = 
CALCULATE(AVERAGE(fact_orders[NPS_Score]), NOT ISBLANK(fact_orders[NPS_Score]))
```
> 💡 `NPS Score` runs −1 to +1. Multiply by 100 for the standard −100 to +100 scale.

### #17 — Seasonal Demand Pattern Dashboard
```dax
Seasonal Index = 
DIVIDE(
    [Net Revenue],
    CALCULATE([Net Revenue], ALL(dim_calendar[Month]), ALL(dim_calendar[MonthName])) / 12
)
```
> 💡 Index > 1.0 = above average month. Build a Matrix (ProductCategory on rows, MonthName on columns) with conditional formatting → colour scale (red → white → green).

### #18 — Customer Acquisition Cost (CAC) by Channel
```dax
New Customers = 
SUM(fact_campaigns[NewCustomers])

CAC = 
DIVIDE([Total Campaign Spend], [New Customers])

LTV to CAC Ratio = 
DIVIDE([Customer LTV], [CAC])
```
> 💡 LTV:CAC above 3× is healthy. Below 1× means you're losing money acquiring customers.

### #19 — Product Category Affinity & Cross-Sell
```dax
Shoes + Clothing Customers = 
CALCULATE(
    DISTINCTCOUNT(fact_orders[CustomerID]),
    FILTER(
        VALUES(fact_orders[CustomerID]),
        CALCULATE(COUNTROWS(fact_orders), fact_orders[ProductCategory] = "Shoes") > 0
        &&
        CALCULATE(COUNTROWS(fact_orders), fact_orders[ProductCategory] = "Clothing") > 0
    )
)
```

### #20 — Brand Partner Performance Dashboard
```dax
Brand Rank = 
RANKX(ALL(fact_orders[Brand]), [Net Revenue], , DESC, Dense)

Brand Return Rate % = 
CALCULATE([Return Rate %], ALLEXCEPT(fact_orders, fact_orders[Brand]))

Brand AOV = 
CALCULATE([AOV], ALLEXCEPT(fact_orders, fact_orders[Brand]))
```
> 💡 Scatter chart (X = `[Return Rate %]`, Y = `[Net Revenue]`, Legend = Brand) — brands in the top-left (high revenue, low returns) are your best partners.
```

---

## 🛠️ DAX Functions Reference

| Function | Used In |
|---|---|
| `RANKX` |  ranking reps, campaigns, platforms |
| `SAMEPERIODLASTYEAR` |  YoY growth and category mix shift |
| `DATESINPERIOD` |  2-month rolling average |
| `DATEADD` |  month-over-month comparison |
| `ALLEXCEPT` | customer-level aggregation while keeping one filter |
| `EXCEPT` |  cross-sell gap identification |
| `SWITCH(TRUE())` |  multi-condition tier logic |
| `CALCULATETABLE` |  virtual table filtering |
| `DIVIDE` | Throughout — safe division with blank handling |
| `ISBLANK` |  null-safe calculations |


---

## 👤 About

**Meheraj Talukdar** |  Junior Business Data Analyst | Düsseldorf, Germany
🔗 GitHub: [MeherajTalukdar](https://github.com/MeherajTalukdar) · 📊 Power BI · SQL · Excel · DAX
🎯 Target roles: Data Analyst · BI Analyst · Reporting Analyst · Controlling

