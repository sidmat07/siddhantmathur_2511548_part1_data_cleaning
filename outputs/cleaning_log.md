# Data Cleaning Log — raw_orders.xlsx

**Date:** June 2026  
**Dataset:** raw_orders.xlsx (932 rows × 21 columns)  
**Cleaned Output:** data/cleaned_orders.xlsx (912 rows × 28 columns)

---

## 1. Issues Found

### Text Field Issues
- **segment**: 16 unique variants due to extra spaces, mixed case, and double spaces (e.g., `"  Small Business "`, `"SMALL BUSINESS"`, `"Small  Business"`)
- **region**: 16 variants including extra spaces and mixed case (e.g., `"  North "`, `"NORTH"`, `"west"`) + **26 missing values**
- **category**: 13 variants due to leading/trailing spaces, all-caps, and double spaces (e.g., `"OFFICE SUPPLIES"`, `"Office  Supplies"`)
- **sub_category**: 23 variants — same issues as above
- **ship_mode**: 12 variants + **22 missing values**
- **payment_status**: 8 variants (case and space issues)
- **order_status**: 10 variants (case and space issues)

### Date Issues
- Mixed date formats across `order_date` and `ship_date`:
  - `21 Jul 2024` (text with month name)
  - `08/31/2024` (MM/DD/YYYY)
  - `2024-05-24` (ISO format)
  - `28-11-2024` (DD-MM-YYYY)
  - `15 Jun 2024` (text with month name)
- **21 records** had ship dates earlier than order dates → flagged as INVALID
- Some date strings could not be reliably parsed → flagged as missing

### Discount Issues
- **18 missing** discount values
- **Negative discounts** found (e.g., -0.19, -0.23): clearly data entry errors
- **Percentage strings** (e.g., `"70%"`, `"85%"`): non-standard format; converted to decimals
- Discounts above 90% threshold flagged as suspicious

### Duplicate Issues
- **20 exact duplicate rows** identified (all 21 columns identical)
- **12 unique order IDs** appeared more than once with differing data (conflicting records)

### Missing Values
- `region`: 26 missing
- `ship_mode`: 22 missing
- `discount`: 18 missing

---

## 2. Cleaning Actions Performed

### Text Cleaning
1. Applied `.strip()` to remove leading/trailing spaces
2. Collapsed multiple internal spaces to single space using regex `\s+` → `' '`
3. Applied `.title()` for consistent Title Case
4. Removed unwanted special characters (kept alphanumeric, space, hyphen, period, comma, ampersand, parentheses)
5. Applied explicit normalization for edge cases:
   - `"Small  Business"` → `"Small Business"`
   - `"Office  Supplies"` → `"Office Supplies"`
   - `"Standard  Class"` → `"Standard Class"`

### Date Cleaning
1. Attempted parsing with 7 date format patterns in priority order
2. Converted all parseable dates to Python `datetime` objects
3. Stored as ISO-format date values in `order_date` and `ship_date` columns
4. Created `shipping_delay_days` = ship_date − order_date
5. Extracted `order_month` and `order_year` from cleaned order_date
6. Populated `date_flag` column with human-readable issue descriptions

### Discount Cleaning
1. Parsed `"70%"` and `"85%"` strings → 0.70 and 0.85 decimal values
2. Stored in `cleaned_discount` column
3. Negative discounts flagged in `discount_flag` (not zeroed or removed)
4. Discounts > 0.90 flagged as unusually high
5. Missing discounts → treated as 0.0 where all sales fields were valid

### Duplicate Handling
1. Exact duplicates (all 21 columns match): **removed**, kept first occurrence
2. Same order_id with conflicting data: **flagged** with `DUPLICATE_ORDER_ID_FLAGGED` in `quality_notes`, kept all records for human review
3. No silent deletions of conflicting records

### Missing Value Handling
- `region` missing → filled as `"Unknown"`, flagged in quality report
- `ship_mode` missing → filled as `"Unknown"`, flagged in quality report
- `discount` missing → treated as 0 in `cleaned_discount` where sales fields were valid

---

## 3. Business Rules Applied

| Rule Area | Action Taken |
|---|---|
| Missing region | Filled as `Unknown`, noted in quality report |
| Missing ship_mode | Filled as `Unknown`, noted in quality report |
| Missing discount | Treated as 0 if qty, unit_price, and sales all valid |
| Negative discount | Flagged as `DISCOUNT_ISSUE` in `data_quality_flag` |
| Discount above 90% | Flagged as `DISCOUNT_ISSUE` in `data_quality_flag` |
| Cancelled orders | Excluded from completed sales pivot summaries |
| Failed payments | Excluded from completed sales pivot summaries |
| Returned orders | Separately summarized in `pivot_summary.xlsx` → Issues by Region sheet |
| Ship date before order date | Flagged as `INVALID` in `data_quality_flag` |

---

## 4. Calculated Columns Added

| Column | Calculation |
|---|---|
| `cleaned_discount` | Standardized from raw `discount` (parsed %, negatives flagged) |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount)` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` |
| `shipping_delay_days` | `ship_date − order_date` in days |
| `order_month` | Month number extracted from `order_date` |
| `order_year` | Year extracted from `order_date` |
| `data_quality_flag` | `CLEAN`, `WARNING`, or `INVALID` |
| `quality_notes` | Human-readable description of issues per record |

---

## 5. Records Summary

| Category | Count |
|---|---|
| Raw records | 932 |
| Exact duplicates removed | 20 |
| Records in cleaned dataset | 912 |
| CLEAN records | 854 |
| WARNING records | 37 |
| INVALID records | 21 |

---

## 6. Assumptions Made

1. Date format `MM/DD/YYYY` was assumed for ambiguous formats (e.g., `08/31/2024` can only be Aug 31, not day 8 of month 31 — unambiguous here, but used MM/DD as default US format)
2. Discount values between 0 and 0.9 were considered valid business range
3. `"70%"` and `"85%"` format discounts were treated as legitimate percentage entries, converted to 0.70 and 0.85
4. For profit calculation, negative discounts were excluded from `cleaned_discount_calc` (treated as NaN) to avoid inflating calculated sales
5. `calculated_sales` was computed fresh; discrepancies with the original `sales` column were noted in the quality report but the original column was preserved
6. `order_status` cleaning used `.title()` so `"COMPLETED"` → `"Completed"` — this is the canonical form used throughout

---

## 7. Limitations

1. **No external reference for customer data**: Cannot validate customer_id ↔ customer_name consistency
2. **Unit price validation**: No product catalog to verify unit prices are reasonable
3. **City/State validation**: No geolocation lookup performed; city-state-region combinations are not cross-validated
4. **Partial date parsing**: Some date strings in unusual formats (e.g., localized text dates) may have silently failed to parse
5. **Conflicting duplicate records**: The 12 conflicting order IDs were flagged but not resolved — a human reviewer must decide which record is authoritative
6. **Discount ceiling**: The 90% threshold was chosen as a reasonable business rule; it may not match the actual discount policy
