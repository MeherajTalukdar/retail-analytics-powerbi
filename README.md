# retail-analytics-powerbi

## E-commerce Analytics in Power BI — Sales, Marketing & Customer Insights
### From Raw Data to Dashboard | German Job Market Focus

Power BI retail analytics project: data cleaning in Power Query, star schema modelling, and executive dashboards covering sales performance, campaign ROI, and customer behaviour.

---

## Project Overview

| Domain | Problems | Focus |
|---|---|---|
| Sales & Revenue | 01 – 10 | Real-time tracking, MoM & YoY trends, return rate, Top N rankings, discount impact, channel attribution |
| Marketing & E-commerce | 11 – 20 | Customer LTV, campaign ROI & ROAS, cohort retention, NPS, seasonal demand, CAC by channel, brand performance |

---

## Data Model

Star schema with 3 core tables.

| Table | Rows | Description |
|---|---|---|
| `orders` | 10,000 | Core fact table — sales, returns, customer and marketing analytics |
| `campaigns` | 300 | Marketing campaign spend and performance |
| `calendar` | — | Date dimension (2022–2024) — powers all time intelligence DAX |

Key columns in `orders`:
`OrderID` · `OrderDate` · `DeliveryDate` · `CustomerID` · `CustomerSegment` · `AcquisitionChannel` · `Country` · `Platform` · `Brand` · `ProductCategory` · `ProductName` · `SKU` · `Quantity` · `UnitPrice_EUR` · `COGS_EUR` · `PromoCode` · `DiscountPct` · `DiscountAmt_EUR` · `GrossRevenue_EUR` · `NetRevenue_EUR` · `GrossProfit_EUR` · `ShippingCost_EUR` · `TransactionType` · `Carrier` · `ShippingDays` · `OnTimeDelivery` · `Warehouse` · `NPS_Score` · `SalesRep`

Key columns in `campaigns`:
`CampaignID` · `CampaignName` · `Platform` · `CampaignType` · `StartDate` · `EndDate` · `TargetSegment` · `Spend_EUR` · `Impressions` · `Clicks` · `EmailOpens` · `Conversions` · `Revenue_EUR` · `NewCustomers` · `Unsubscribes`

---

## Foundation Measures

Everything else depends on these four measures. Create them first.

```dax
Total Orders =
DISTINCTCOUNT(orders[OrderID])

Net Revenue =
SUMX(
    orders,
    orders[Quantity] * orders[UnitPrice_EUR]
    - orders[DiscountAmt_EUR]
)

Gross Profit =
SUM(orders[GrossProfit_EUR])

Gross Margin % =
DIVIDE([Gross Profit], [Net Revenue])
```

---

## DAX Measures — Problems 01 to 10 (Sales & Revenue)

---

### 01 — Real-Time Sales Performance Dashboard

Track total net revenue against a monthly target. KPI cards with a trend line by month.

```dax
Revenue vs Target =
VAR target = 150000
RETURN [Net Revenue] - target
```

Visuals: KPI card (Net Revenue), KPI card (Revenue vs Target), line chart with calendar[YearMonth] on axis and [Net Revenue] as value, slicer on Country and Platform.

---

### 02 — MoM & YoY Revenue Trend

Compare this month and this year's revenue to the prior periods.

```dax
Prev Month Revenue =
CALCULATE(
    [Net Revenue],
    DATEADD(calendar[Date], -1, MONTH)
)

MoM % Change =
DIVIDE(
    [Net Revenue] - [Prev Month Revenue],
    [Prev Month Revenue]
)

Prev Year Revenue =
CALCULATE(
    [Net Revenue],
    SAMEPERIODLASTYEAR(calendar[Date])
)

YoY % Change =
DIVIDE(
    [Net Revenue] - [Prev Year Revenue],
    [Prev Year Revenue]
)
```

Visuals: Line chart with two series ([Net Revenue] and [Prev Year Revenue]), matrix with Year on columns and MonthName on rows, KPI card showing [YoY % Change].

---

### 03 — Product Return Rate Dashboard

Measure return rate per product category and SKU to identify profit-draining items.

```dax
Gross Orders =
CALCULATE(
    [Total Orders],
    orders[TransactionType] = "Sale"
)

Return Orders =
CALCULATE(
    [Total Orders],
    orders[TransactionType] = "Return"
)

Return Rate % =
DIVIDE([Return Orders], [Gross Orders])
```

Visuals: Bar chart (ProductCategory on axis, [Return Rate %] as value), line chart (YearMonth on axis, [Return Rate %] as value). Add drill-through page to SKU level.

---

### 04 — Top N Product Rankings

Rank products by revenue with a dynamic Top N slicer.

```dax
Product Rank =
RANKX(
    ALL(orders[ProductName]),
    [Net Revenue],
    ,
    DESC,
    Dense
)

Top N Filter =
IF([Product Rank] <= [TopN Value], 1, 0)
```

Setup: Modeling → New Parameter → name it TopN, Min = 5, Max = 50, Increment = 5, Default = 10. Add [Top N Filter] = 1 as a visual-level filter on your bar chart.

Visuals: Bar chart (ProductName on axis, [Net Revenue] as value, filtered to [Top N Filter] = 1), slicer connected to TopN parameter.

---

### 05 — Revenue by Geography

No new DAX needed — [Net Revenue] works directly.

Setup: Set the Country column Data Category = Country/Region in Column tools (Column tools tab → Data category).

Visuals: Filled map (Country as Location, [Net Revenue] as Bubble size), bar chart below sorted by [Net Revenue] descending.

---

### 06 — Sales Funnel & Conversion Rate

Measure average order value and purchase frequency by platform and customer type.

```dax
AOV =
DIVIDE([Net Revenue], [Total Orders])

Orders per Customer =
DIVIDE(
    [Total Orders],
    CALCULATE(
        DISTINCTCOUNT(orders[CustomerID]),
        orders[CustomerID] <> "GUEST"
    )
)
```

Visuals: Bar chart (Platform on axis, [Total Orders] and [Net Revenue] as values), line chart (YearMonth, [AOV]).

---

### 07 — Discount & Promotion Impact

Quantify how much revenue is being given away through discounts and what it costs in margin.

```dax
Total Discount =
SUM(orders[DiscountAmt_EUR])

Discount % of Revenue =
DIVIDE(
    [Total Discount],
    [Net Revenue] + [Total Discount]
)

Net Margin % =
DIVIDE([Gross Profit], [Net Revenue])
```

Visuals: Matrix (PromoCode on rows, columns = [Total Orders], [Total Discount], [Net Revenue], [Net Margin %]), bar chart (ProductCategory on axis, [Total Discount] as value).

---

### 08 — Revenue Forecasting

3-month rolling average plus a built-in Power BI forecast line.

```dax
Rolling 3M Revenue =
CALCULATE(
    [Net Revenue],
    DATESINPERIOD(
        calendar[Date],
        LASTDATE(calendar[Date]),
        -3,
        MONTH
    )
)
```

Forecast setup: Build a line chart with YearMonth on axis and [Net Revenue] as value. Then Analytics Pane → Forecast → turn on → Forecast Length = 6 months, Confidence Interval = 95%.

---

### 09 — Basket Size & AOV Analysis

Understand how many items customers buy per order and how that varies by segment.

```dax
Avg Basket Size =
DIVIDE(
    SUM(orders[Quantity]),
    [Total Orders]
)
```

AOV is already defined in 06.

Visuals: Bar chart (CustomerSegment on axis, [AOV] as value), scatter chart (X = [Avg Basket Size], Y = [AOV], Legend = ProductCategory).

---

### 10 — Channel Mix & Attribution

See what share of revenue comes from each acquisition channel and how it shifts over time.

```dax
Channel Revenue Share % =
DIVIDE(
    [Net Revenue],
    CALCULATE([Net Revenue], ALL(orders[AcquisitionChannel]))
)
```

Visuals: 100% stacked bar (YearMonth on axis, AcquisitionChannel as legend, [Net Revenue] as value), donut chart (AcquisitionChannel as legend, [Net Revenue] as value).

---

## DAX Measures — Problems 11 to 20 (Marketing & E-commerce)

Before writing any DAX for problems 11–20, set up these relationships in Model view:

- `campaigns[StartDate]` → `calendar[Date]` (active relationship)
- `orders[OrderDate]` → `calendar[Date]` (already done)
- `campaigns` and `orders` stay unrelated to each other — they are separate fact tables sharing only the calendar dimension. This is correct star schema design.

---

### 11 — Customer Segmentation & LTV Dashboard

Segment customers by value and measure how long they stay active.

```dax
Active Customers =
CALCULATE(
    DISTINCTCOUNT(orders[CustomerID]),
    orders[CustomerID] <> "GUEST"
)

Customer LTV =
DIVIDE([Net Revenue], [Active Customers])

Avg Orders per Customer =
DIVIDE([Total Orders], [Active Customers])

Recency Days =
DATEDIFF(
    CALCULATE(MAX(orders[OrderDate]), ALL(calendar)),
    TODAY(),
    DAY
)
```

Visuals: Scatter chart (X = [Avg Orders per Customer], Y = [Customer LTV], Legend = CustomerSegment), treemap (CustomerSegment as group, [Net Revenue] as values), KPI cards per segment using a CustomerSegment slicer.

---

### 12 — Campaign Performance & ROI Dashboard

All measures in this section use the `campaigns` table — no relationship to `orders` needed.

```dax
Total Campaign Spend =
SUM(campaigns[Spend_EUR])

Total Campaign Revenue =
SUM(campaigns[Revenue_EUR])

ROAS =
DIVIDE([Total Campaign Revenue], [Total Campaign Spend])

CPA =
DIVIDE([Total Campaign Spend], SUM(campaigns[Conversions]))

Campaign ROI % =
DIVIDE(
    [Total Campaign Revenue] - [Total Campaign Spend],
    [Total Campaign Spend]
)

CTR % =
DIVIDE(SUM(campaigns[Clicks]), SUM(campaigns[Impressions]))
```

Visuals: Matrix (CampaignName on rows, columns = [Total Campaign Spend], [ROAS], [CPA], [Campaign ROI %]), bar chart (Platform on axis, [ROAS] as value), scatter (X = [Total Campaign Spend], Y = [Total Campaign Revenue], Legend = Platform).

---

### 13 — Customer Cohort Retention Dashboard

Track what percentage of customers acquired in month X are still buying in later months.

Step 1 — Add this as a calculated column on `orders` in Data view:

```dax
CohortMonth =
FORMAT(
    CALCULATE(
        MIN(orders[OrderDate]),
        ALLEXCEPT(orders, orders[CustomerID])
    ),
    "YYYY-MM"
)
```

Step 2 — Measures:

```dax
Cohort Size =
CALCULATE(
    DISTINCTCOUNT(orders[CustomerID]),
    orders[CohortMonth] = MAX(orders[CohortMonth])
)

Retained Customers =
DISTINCTCOUNT(orders[CustomerID])

Retention Rate % =
DIVIDE([Retained Customers], [Cohort Size])
```

Visuals: Matrix (CohortMonth on rows, calendar[YearMonth] on columns, [Retention Rate %] as values). Apply conditional formatting → background colour scale (white → green). This creates the classic cohort heatmap.

---

### 14 — Web & App Analytics (Platform Mix)

Compare order volume and revenue across Own Website, Mobile App, and Marketplace.

```dax
Mobile App Orders =
CALCULATE(
    [Total Orders],
    orders[Platform] = "Mobile App"
)

Mobile App Share % =
DIVIDE([Mobile App Orders], [Total Orders])
```

Visuals: Donut chart (Platform as legend, [Total Orders] as values), bar chart (Platform on axis, [Net Revenue] as value), line chart (YearMonth, [Total Orders] split by Platform legend).

---

### 15 — Email & CRM Campaign Performance

Filter campaign data to Email platform rows and measure funnel performance at each stage.

```dax
Email Open Rate % =
CALCULATE(
    DIVIDE(SUM(campaigns[EmailOpens]), SUM(campaigns[Impressions])),
    campaigns[Platform] = "Email"
)

Email CTR % =
CALCULATE(
    DIVIDE(SUM(campaigns[Clicks]), SUM(campaigns[EmailOpens])),
    campaigns[Platform] = "Email"
)

Email Conversion Rate % =
CALCULATE(
    DIVIDE(SUM(campaigns[Conversions]), SUM(campaigns[Clicks])),
    campaigns[Platform] = "Email"
)

Email Unsubscribe Rate % =
CALCULATE(
    DIVIDE(SUM(campaigns[Unsubscribes]), SUM(campaigns[Impressions])),
    campaigns[Platform] = "Email"
)
```

Visuals: Matrix (CampaignName on rows, all 4 rate measures as columns), line chart (StartDate on axis, [Email Open Rate %] and [Email CTR %] as values), slicer on TargetSegment.

---

### 16 — Customer Complaint & NPS Dashboard

Calculate the Net Promoter Score and break it down by product category.

```dax
NPS Promoters =
CALCULATE(
    COUNTROWS(orders),
    orders[NPS_Score] >= 9,
    NOT ISBLANK(orders[NPS_Score])
)

NPS Detractors =
CALCULATE(
    COUNTROWS(orders),
    orders[NPS_Score] <= 6,
    NOT ISBLANK(orders[NPS_Score])
)

NPS Responses =
CALCULATE(
    COUNTROWS(orders),
    NOT ISBLANK(orders[NPS_Score])
)

NPS Score =
DIVIDE([NPS Promoters], [NPS Responses])
- DIVIDE([NPS Detractors], [NPS Responses])

Avg NPS Score =
CALCULATE(
    AVERAGE(orders[NPS_Score]),
    NOT ISBLANK(orders[NPS_Score])
)
```

Note: [NPS Score] runs from -1 to +1. Multiply by 100 for the standard -100 to +100 scale.

Visuals: KPI card ([NPS Score]), gauge chart ([Avg NPS Score], min = 0, max = 10), bar chart (ProductCategory on axis, [Avg NPS Score] as value), line chart (YearMonth, [NPS Score] trend).

---

### 17 — Seasonal Demand Pattern Dashboard

Index each month's revenue against the annual average to find seasonal peaks and troughs.

```dax
Seasonal Index =
DIVIDE(
    [Net Revenue],
    CALCULATE(
        [Net Revenue],
        ALL(calendar[Month]),
        ALL(calendar[MonthName])
    ) / 12
)
```

A Seasonal Index above 1.0 means that month outperforms the annual average. Index below 1.0 means it underperforms.

Visuals: Matrix (ProductCategory on rows, MonthName on columns, [Seasonal Index] as values). Apply conditional formatting → colour scale (red → white → green). This creates the seasonal heatmap. Line chart (MonthName on axis, [Net Revenue] split by ProductCategory legend).

---

### 18 — Customer Acquisition Cost (CAC) by Channel

Measure how much it costs to acquire one new customer per marketing channel, and compare it to LTV.

```dax
New Customers =
SUM(campaigns[NewCustomers])

CAC =
DIVIDE([Total Campaign Spend], [New Customers])

LTV to CAC Ratio =
DIVIDE([Customer LTV], [CAC])
```

LTV:CAC above 3x is considered healthy in e-commerce. Below 1x means you are losing money on every new customer acquired.

Visuals: Bar chart (Platform on axis, [CAC] as value), scatter (X = [CAC], Y = [Customer LTV], Legend = Platform), KPI card ([LTV to CAC Ratio]) with target annotation at 3x.

---

### 19 — Product Category Affinity & Cross-Sell

Find customers who bought from one category but not another — the core cross-sell audience.

```dax
Shoes + Clothing Customers =
CALCULATE(
    DISTINCTCOUNT(orders[CustomerID]),
    FILTER(
        VALUES(orders[CustomerID]),
        CALCULATE(
            COUNTROWS(orders),
            orders[ProductCategory] = "Shoes"
        ) > 0
        &&
        CALCULATE(
            COUNTROWS(orders),
            orders[ProductCategory] = "Clothing"
        ) > 0
    )
)
```

Visuals: Matrix (ProductCategory on both rows and columns, [Net Revenue] as values) — shows which categories drive revenue when filtered together. Use cross-filter interaction between two bar charts (one per category) for the interactive affinity experience.

---

### 20 — Brand Partner Performance Dashboard

Rank brand partners by revenue, return rate, and average order value to identify your best and worst performers.

```dax
Brand Rank =
RANKX(
    ALL(orders[Brand]),
    [Net Revenue],
    ,
    DESC,
    Dense
)

Brand Return Rate % =
CALCULATE(
    [Return Rate %],
    ALLEXCEPT(orders, orders[Brand])
)

Brand AOV =
CALCULATE(
    [AOV],
    ALLEXCEPT(orders, orders[Brand])
)
```

Visuals: Matrix (Brand on rows, columns = [Net Revenue], [Brand Rank], [Brand Return Rate %], [Brand AOV], [Total Orders]), bar chart (Brand on axis, [Net Revenue] sorted descending), scatter (X = [Brand Return Rate %], Y = [Net Revenue], Legend = Brand). Brands in the top-left quadrant — high revenue, low returns — are your strongest partners.

---

## DAX Functions Used

| Function | Problems |
|---|---|
| `RANKX` | 04, 12, 20 — product, campaign, and brand rankings |
| `SAMEPERIODLASTYEAR` | 02 — year-over-year revenue comparison |
| `DATEADD` | 02 — month-over-month comparison |
| `DATESINPERIOD` | 08 — 3-month rolling average |
| `ALLEXCEPT` | 13, 20 — customer and brand-level aggregation |
| `FILTER` + `VALUES` | 19 — cross-sell customer identification |
| `DIVIDE` | Throughout — safe division with blank handling |
| `ISBLANK` | 16 — null-safe NPS calculation |
| `FORMAT` | 13 — cohort month calculated column |
| `DATEDIFF` | 11 — customer recency in days |

---

## Repository Contents

```
retail-analytics-powerbi/
    Power Bi Analysis and dashboards.pbix
    retail-analytics Dataset.xlsx
    README.md
```

---

## About

Meheraj Talukdar — Junior Data Analyst — Dusseldorf, Germany

GitHub: github.com/MeherajTalukdar
Stack: Power BI · SQL · Excel · DAX
Target roles: Data Analyst · BI Analyst · Reporting Analyst · Controlling
