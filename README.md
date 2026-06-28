# Orders Data Cleaning & Analysis Project

## Problem Summary

The raw dataset `raw_orders.xlsx` contained 932 order records with significant data quality issues: inconsistent text formatting, mixed date formats, invalid discount values, missing fields, and duplicate records. This project performs a full end-to-end cleaning pipeline, applies business rules, generates calculated columns, and produces data quality and pivot summary reports.

---

## Dataset Description

| Property | Value |
|---|---|
| Source File | `data/raw_orders.xlsx` |
| Sheet | `raw_orders` |
| Rows | 932 |
| Columns | 21 |
| Date Range | 2024–2025 |
| Geography | India (regions: North, South, East, West) |

### Columns in Raw Data

`order_id`, `order_date`, `ship_date`, `customer_id`, `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `product_name`, `ship_mode`, `quantity`, `unit_price`, `discount`, `sales`, `cost`, `profit`, `payment_status`, `order_status`

---

## Tools Used

- **Python 3** — data processing engine
- **pandas** — data manipulation and analysis
- **openpyxl** — Excel file creation and formatting
- **matplotlib** — screenshot/chart generation
- **numpy** — numerical calculations

---

## Cleaning Steps Performed

### 1. Raw Data Preserved
The original file `data/raw_orders.xlsx` was not modified. All cleaning was performed in a separate file: `data/cleaned_orders.xlsx`.

### 2. Text Field Standardization
- Stripped leading/trailing whitespace from all text fields
- Collapsed multiple internal spaces to single space
- Applied Title Case uniformly
- Removed special characters (kept: alphanumeric, space, `-`, `.`, `,`, `&`, `()`)
- Manually mapped edge cases: `"Small  Business"` → `"Small Business"`, `"Office  Supplies"` → `"Office Supplies"`, `"Standard  Class"` → `"Standard Class"`

### 3. Date Parsing & Validation
- Parsed 7 date formats: `DD Mon YYYY`, `MM/DD/YYYY`, `YYYY-MM-DD`, `DD-MM-YYYY`, `YYYY/MM/DD`, `DD/MM/YYYY`, `DD-Mon-YYYY`
- Created `shipping_delay_days` = ship_date − order_date
- Extracted `order_month` and `order_year`
- Flagged: missing dates, unparseable dates, ship date before order date (21 records)

### 4. Duplicate Handling
- Identified **20 exact duplicate rows** → removed (kept first occurrence)
- Identified **12 order IDs** with conflicting information → flagged with `DUPLICATE_ORDER_ID_FLAGGED`, kept for human review (not silently deleted)

### 5. Missing Value Treatment
- `region` (26 missing) → filled as `"Unknown"`, flagged in report
- `ship_mode` (22 missing) → filled as `"Unknown"`, flagged in report
- `discount` (18 missing) → treated as `0` where all sales fields were valid

### 6. Calculated Columns Added
| Column | Formula |
|---|---|
| `cleaned_discount` | Parsed from raw discount, `"70%"` → `0.70`, negatives flagged |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` in days |
| `order_month` | Extracted month number |
| `order_year` | Extracted year |
| `data_quality_flag` | `CLEAN` / `WARNING` / `INVALID` |
| `quality_notes` | Human-readable issue description |

---

## Business Rules Applied

| Rule | Action |
|---|---|
| Missing region | Filled `Unknown`, flagged |
| Missing ship_mode | Filled `Unknown`, flagged |
| Missing discount | Treated as 0 if sales fields valid |
| Negative discount | Flagged `DISCOUNT_ISSUE` |
| Discount > 90% | Flagged `DISCOUNT_ISSUE` |
| Cancelled orders | Excluded from sales summaries |
| Failed payments | Excluded from sales summaries |
| Returned orders | Separately summarized |
| Ship date < order date | Flagged `INVALID` |

---

## Summary of Data Quality Issues Found

| Issue | Count |
|---|---|
| Exact duplicate rows | 20 |
| Conflicting order IDs (flagged) | 24 records (12 unique IDs × 2) |
| Missing region values | 26 |
| Missing ship_mode values | 22 |
| Missing discount values | 18 |
| Negative discount values | Multiple (flagged) |
| Discount > 90% | Multiple (flagged) |
| Ship date before order date | 21 |
| Text fields with case/space issues | All 10 text fields affected |
| Mixed date formats | Both date columns |

### Final Record Counts (Cleaned Dataset: 912 rows)
| Flag | Count |
|---|---|
| CLEAN | 854 |
| WARNING | 37 |
| INVALID | 21 |

---

## Summary of Pivot Reports (`outputs/pivot_summary.xlsx`)

| Sheet | Summary |
|---|---|
| **Sales by Region** | East region leads in total sales; sorted descending by sales |
| **Sales by Category** | Furniture and Technology have highest average order values; Office Supplies highest volume |
| **Orders by Ship Mode** | Standard Class is the most used mode; Same Day has smallest share |
| **Profit by Segment** | Corporate segment shows highest profit margin; Home Office has smallest order count |
| **Issues by Region** | Issues distributed relatively evenly; flagged cancellations and returns by region |
| **Monthly Sales Trend** | Sales show seasonal peaks; trend visible across 2024–2025 |

---

## Key Business Insights

1. **Completed orders dominate** (~85%) but Cancelled and Returned orders represent meaningful revenue leakage that should be tracked.
2. **Negative discounts** in the raw data indicate a data entry issue in the source system — the sales team should be alerted.
3. **Ship-before-order anomalies** (21 records) suggest either timezone issues or data entry errors in the order management system.
4. **East and South regions** perform competitively in sales; West shows lower volume but comparable margins.
5. **Technology sub-categories** (Copiers, Machines) drive higher average order values but lower volume.
6. **Shipping delays** average 4–6 days across modes; Same Day orders still show occasional multi-day delays (data issue or business exception).

---

## Assumptions & Limitations

- Date format `MM/DD/YYYY` was assumed as the default for ambiguous numeric dates
- Discount range `0–0.9` (0%–90%) defined as valid; this is an assumption
- `"70%"` and `"85%"` string discounts were treated as legitimate values
- Conflicting duplicate records were **not** automatically resolved — human review required
- No external product catalog or customer master was available for cross-validation
- City/State/Region combinations were not geographically validated

---

## Screenshots

| File | Description |
|---|---|
| `outputs/raw_data_preview.png` | First 10 rows of raw data showing mixed formats and issues |
| `outputs/cleaned_data_preview.png` | First 10 rows of cleaned data with calculated columns and quality flags |
| `outputs/pivot_summary_1.png` | Sales & Profit by Region bar charts |
| `outputs/pivot_summary_2.png` | Monthly Sales Trend with dual-axis chart |

---

## Output Files

```
data/
  raw_orders.xlsx          ← Original data (unchanged)
  cleaned_orders.xlsx      ← Cleaned data with calculated columns

outputs/
  data_quality_report.xlsx ← 8-sheet quality report
  pivot_summary.xlsx       ← 6-sheet pivot analysis
  cleaning_log.md          ← Detailed cleaning documentation
  raw_data_preview.png     ← Screenshot: raw data
  cleaned_data_preview.png ← Screenshot: cleaned data
  pivot_summary_1.png      ← Screenshot: regional pivot
  pivot_summary_2.png      ← Screenshot: monthly trend

README.md                  ← This file
```
