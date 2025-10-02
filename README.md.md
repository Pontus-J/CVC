# Portfolio Monitoring Dashboard

This repository contains the Power BI project for an interactive Portfolio Monitoring Dashboard. The dashboard is designed for investment professionals to analyze the financial and operational performance of portfolio companies, with a focus on comparing actual results against budget.

## Key Features

- **Single Company View:** Select a single portfolio company for a detailed analysis.
- **Dynamic Date Slicer:** A flexible date range slicer allows users to define custom analysis periods, including full-year, YTD, or LTM-style selections.
- **At-a-Glance Financials:** KPI cards summarize key metrics like Revenue, EBITDA, Cash Flow, and Net Debt for the selected period.
- **Budget vs. Actuals:** Comprehensive variance analysis (both absolute and percentage) comparing performance against a seasonally-adjusted budget.
- **Trend Analysis:** Line charts visualize monthly trends for key financial metrics and operational KPIs.
- **Comments Log:** A dedicated panel displays chronological comments from the deal team and operating partners.

## Data Modeling & Key Calculations

The model is built to provide maximum analytical flexibility. Below are some of the key technical implementations.

### Budget Phasing using Seasonal Weights

To enable a meaningful monthly Budget vs. Actuals comparison, the provided annual budget figures are spread across months. This process, handled in Power Query, avoids a simplistic 1/12 split and instead reflects the typical seasonality of the business.

**Methodology:**

1. Historical monthly actuals for the most recent full year (2023) are used as the baseline.
2. For each financial metric (e.g., Revenue, EBITDA, Capex), we calculate each month's percentage contribution to the annual total. This creates a *seasonal weight* for each month.
3. The annual budget for the selected year is then allocated to each month using these calculated seasonal weights.
4. As a fallback for metrics with no historical data, an equal 1/12 allocation is used.

> The following Power Query M code was used to generate the seasonal weights table:

```
let
    // Load Financials_Monthly
    Source = Financials_Monthly,
    
    // Filter to 2023 for baseline seasonal patterns
    Historical2023 = Table.SelectRows(Source, each [Year] = 2023),
    
    // Select key columns and unpivot to get all metrics
    SelectColumns = Table.SelectColumns(Historical2023, {"CompanyID", "Month", "Revenue", "EBITDA", "COGS", "CashFromOps", "Capex"}),
    
    // Unpivot to get one row per company/month/metric
    UnpivotedData = Table.UnpivotOtherColumns(SelectColumns, {"CompanyID", "Month"}, "Metric", "MonthlyValue"),
    
    // Calculate annual totals per company per metric
    AnnualTotals = Table.Group(UnpivotedData, {"CompanyID", "Metric"}, 
        {{"AnnualTotal", each List.Sum([MonthlyValue]), type number}}),
    
    // Join monthly data with annual totals
    JoinedData = Table.NestedJoin(UnpivotedData, {"CompanyID", "Metric"}, AnnualTotals, {"CompanyID", "Metric"}, "Annual", JoinKind.Inner),
    ExpandAnnual = Table.ExpandTableColumn(JoinedData, "Annual", {"AnnualTotal"}),
    
    // Calculate seasonal weight (monthly value / annual total)
    AddSeasonalWeight = Table.AddColumn(ExpandAnnual, "SeasonalWeight", each 
        if [AnnualTotal] = 0 or [AnnualTotal] = null then (1/12) 
        else [MonthlyValue] / [AnnualTotal], type number),
    
    // Clean up and format
    FinalWeights = Table.SelectColumns(AddSeasonalWeight, {"CompanyID", "Month", "Metric", "SeasonalWeight"}),
    
    // Add data types
    TypedTable = Table.TransformColumnTypes(FinalWeights, {
        {"CompanyID", type text},
        {"Month", Int64.Type},
        {"Metric", type text},
        {"SeasonalWeight", type number}
    })
in
    TypedTable
```

### Advanced Time Intelligence with DAX

A significant amount of DAX was written to support the dashboard's flexible time analysis. The primary challenge was to display metrics for the user-defined period from the date slicer while simultaneously showing fixed-period comparisons like *Last Month* and *LTM (Last Twelve Months)*.

This was achieved by:
- **Context Manipulation:** Using `CALCULATE` in conjunction with filter modifiers like `REMOVEFILTERS` and `ALL` to create measures that ignore the primary date slicer's context.
- **Dual Calendar Approach:** The visual for the date slicer utilizes a secondary, disconnected date table. This allows the user's selection to be captured and used to filter the main visuals, while DAX measures can independently calculate metrics for different time frames (e.g., LTM average) relative to the end date of the user's selection.

## Dashboard Design Philosophy

The dashboard is intentionally designed as a single, information-dense page. This approach is tailored for professional finance users who value having a complete, holistic view of a company's performance without needing to navigate between multiple pages. The goal is to provide *maximum analytical flexibility* and empower users to draw connections between different data points—from high-level financials to operational KPIs and qualitative comments—all within a single view.

## Limitations & Future Enhancements

While the current single-page design offers density and flexibility, a future iteration could adopt a more guided, multi-page structure to cater to different analytical workflows.

Potential next steps include:
- **Introductory Page:** A landing page to provide a high-level company overview and a more prominent view of the comments log.
- **Financial Deep Dives:** Splitting the analysis into two dedicated reports:
    - A **P&L Deep Dive** with more granular margin analysis.
    - A **Cash Flow Deep Dive** focusing on cash conversion cycles and capital structure.
- **Forecasting:** Incorporating financial forecasts for 2025-2026 to provide a forward-looking perspective.
- **Expanded KPI Analysis:** Creating dedicated sections or visuals for more detailed breakouts of company-specific operational KPIs.