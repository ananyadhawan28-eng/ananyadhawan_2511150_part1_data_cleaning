# Part 1: Data Cleaning & Quality Analysis
## Retail Order Data — raw_orders.xlsx

---

## Problem Summary

A retail company exported order-level sales data from multiple internal systems. The raw dataset (932 records, 21 columns) contains significant data quality issues: inconsistent text formatting across multiple fields, at least 8 different date formats in the same columns, duplicate records (exact and conflicting), missing values, negative/invalid discounts, and sales calculation mismatches. The goal is to produce a clean, validated, analysis-ready dataset with full documentation of every decision.

---

## Dataset Description

**File:** `data/raw_orders.xlsx`  
**Records:** 932 rows, 21 columns  
**Coverage:** Retail orders (2024–2025), multiple regions across India

| Column | Type | Description |
|--------|------|-------------|
| order_id | String | Unique order identifier |
| order_date | String (mixed formats) | Date order was placed |
| ship_date | String (mixed formats) | Date order was shipped |
| customer_id | String | Customer identifier |
| customer_name | String | Customer full name |
| segment | Categorical | Consumer / Corporate / Home Office / Small Business |
| region | Categorical | North / South / East / West (26 missing) |
| state | String | Indian state name |
| city | String | City name |
| category | Categorical | Furniture / Office Supplies / Technology |
| sub_category | Categorical | 13 sub-categories |
| product_name | String | Product name |
| ship_mode | Categorical | Standard Class / First Class / Second Class / Same Day (22 missing) |
| quantity | Integer | Units ordered |
| unit_price | Float | Price per unit (₹) |
| discount | Mixed | Discount rate — contains blanks, negatives, % strings |
| sales | Float | Recorded order revenue |
| cost | Float | Cost of goods |
| profit | Float | Recorded profit |
| payment_status | Categorical | Paid / Pending / Failed / Refunded |
| order_status | Categorical | Completed / Cancelled / Returned |

---

## Tools Used

- **Python 3.12** — pandas, numpy, openpyxl, dateutil
- **pandas** — data loading, cleaning, groupby/pivot operations
- **openpyxl** — formatted Excel output (cleaned_orders.xlsx, data_quality_report.xlsx, pivot_summary.xlsx)
- **matplotlib** — screenshot generation

---

## Cleaning Steps Performed

### Step 1: Preserve Raw Data
Original `raw_orders.xlsx` is kept unchanged in `data/`. All work done in a separate `cleaned_orders.xlsx`.

### Step 2: Text Field Standardization
Applied to: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`
- `str.strip()` — removes leading/trailing whitespace
- `re.sub(r'\s+', ' ', s)` — collapses internal multiple spaces to one
- `.title()` — normalizes case (Title Case throughout)

**Example:** `"  SMALL BUSINESS "` → `"Small Business"`

### Step 3: Date Parsing & Validation
- Multi-format parser handles 8+ date formats found in the data
- `shipping_delay_days` computed as `ship_date − order_date`
- 95 records with `ship_date < order_date` flagged as invalid
- `order_month` and `order_year` extracted for trend analysis

### Step 4: Duplicate Handling
- **20 exact duplicate rows** removed (kept first occurrence)
- **12 conflicting duplicate order_ids** flagged with `dup_flag` — NOT deleted; require business review

### Step 5: Business Rules Applied
- Missing `region` → filled `"Unknown"`, flagged
- Missing `ship_mode` → filled `"Unknown"`, flagged
- Missing `discount` → treated as 0, flagged
- Negative discount → flagged as invalid, clipped to 0 in calculations
- Discount > 50% → flagged as invalid, capped at 50% in calculations
- Percentage-format strings (70%, 85%) → converted to decimal

### Step 6: Calculated Columns Added
7 new columns created in `cleaned_orders.xlsx`: `cleaned_discount`, `calculated_sales`, `calculated_profit`, `profit_margin`, `shipping_delay_days`, `order_month`, `order_year`, plus `data_quality_flag`, `date_flag`, `discount_flag`, `dup_flag`.

---

## Business Rules Applied

| Rule | Action |
|------|--------|
| Missing region | Filled "Unknown", flagged |
| Missing ship_mode | Filled "Unknown", flagged |
| Missing discount | Treated as 0 if other fields valid |
| Negative discount | Flagged invalid |
| Discount > 50% | Flagged invalid; capped at 50% |
| Cancelled orders | Excluded from completed sales pivots |
| Failed payments | Excluded from completed sales pivots |
| Refunded orders | Summarized separately |
| Ship date < order date | Flagged; delay = NaN |
| Exact duplicates | Removed (20 rows) |
| Conflicting duplicate order_ids | Flagged, retained for review |

---

## Summary of Data Quality Issues Found

| Issue | Count |
|-------|-------|
| Exact duplicate rows | 20 |
| Conflicting duplicate order_ids | 12 order_ids (~24 rows) |
| Missing region | 26 |
| Missing ship_mode | 22 |
| Missing discount | 18 |
| Negative discount | 16 |
| Discount > 50% | 7 |
| Non-numeric discount (% strings) | 8 |
| Ship date before order date | 95 |
| Sales calculation mismatches | 62 |
| Total records with any flag | 301 |
| Clean records | 611 |

---

## Summary of Pivot Reports

| Pivot | Contents |
|-------|---------|
| Pivot 1: Sales by Region | Sales, cost, profit, margin % by region — sorted by sales (South leads) |
| Pivot 2: Sales by Category | Sales and profit by category and sub-category — sorted within each category |
| Pivot 3: Orders by Ship Mode | Order count, % share, sales, avg delivery days by ship mode |
| Pivot 4: Profit Margin by Segment | Segment performance sorted by margin (highest first) |
| Pivot 5: Problem Orders by Region | Cancelled/Returned/Failed order counts by region with risk flags |
| Pivot 6: Monthly Sales Trend | Month-by-month sales and profit for 2024 and 2025 |

---

## Key Business Insights

1. **South Region leads in sales** for completed + paid orders — largest revenue contributor
2. **Technology sub-categories** (Copiers, Accessories, Phones) deliver the highest sales volumes
3. **Standard Class** is the most used ship mode by far (majority of orders)
4. **95 shipping date errors** — ship dates recorded before order dates; requires source system audit
5. **Discount policy issues:** 16 negative discounts + 7 above 50% suggest data entry errors or system bugs
6. **62 sales calculation mismatches** — recorded `sales` values do not match `quantity × unit_price × (1-discount)` formula
7. **Monthly trend** shows clear seasonality; some months significantly outperform others
8. **Cancelled and Failed orders** are concentrated in specific regions — risk signal for operations teams

---

## Assumptions and Limitations

1. Date format ambiguity — `dayfirst=True` assumed as default (Indian convention DD/MM/YYYY)
2. Maximum valid discount cap assumed at 50%
3. Negative discounts treated as data entry errors, not surcharges
4. Conflicting duplicate records cannot be resolved without source system access
5. Sales mismatches may have legitimate explanations (price overrides, bulk deals) not captured in dataset
6. No master data tables (customers, products, regions) available for referential integrity checks

---

## Screenshots Included

| File | Contents |
|------|---------|
| `screenshots/raw_data_preview.png` | Text issues, date format problems, missing values, duplicate summary from raw data |
| `screenshots/cleaned_data_preview.png` | Quality flag distribution, new calculated columns, discount distribution, order status breakdown |
| `screenshots/pivot_summary_1.png` | Sales & profit by region and category, monthly trend, profit margin by segment |
| `screenshots/pivot_summary_2.png` | Sub-category sales ranking, ship mode distribution, problem orders by region, final quality summary |
```
