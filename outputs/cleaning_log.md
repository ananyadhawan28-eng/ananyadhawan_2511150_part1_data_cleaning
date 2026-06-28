# Data Cleaning Log
## Part 1: Data Cleaning & Quality Report — raw_orders.xlsx

**Analyst:** Business Analyst  
**Raw File:** `data/raw_orders.xlsx` (932 records, 21 columns)  
**Cleaned File:** `data/cleaned_orders.xlsx` (912 records, 32 columns)  
**Tool:** Python (pandas, openpyxl)

---

## 1. Issues Found

### 1.1 Text Field Issues
- **Trailing/leading spaces:** Found in `segment`, `region`, `category`, `ship_mode`, `payment_status`, `order_status`, `customer_name` — e.g., `"  Small Business  "`, `"Paid "`
- **Case inconsistencies:** `"SMALL BUSINESS"`, `"corporate"`, `"STANDARD CLASS"`, `"FIRST CLASS"`, `"storage"`, `"machines"`, `"OFFICE SUPPLIES"` — multiple case variants in same column
- **Double internal spaces:** e.g., `"Small  Business"`, `"Standard  Class"`, `"Office  Supplies"`
- **Category mismatches:** `"  Furniture "`, `"FURNITURE"`, `"Technology "` all represent the same category

### 1.2 Missing Values
| Column | Missing Count | % of Records |
|--------|-------------|-------------|
| region | 26 | 2.79% |
| ship_mode | 22 | 2.36% |
| discount | 18 | 1.93% |

### 1.3 Duplicate Records
- **Exact duplicate rows:** 20 rows where every column matched exactly
- **Conflicting duplicate order_ids:** 12 order_id values appearing more than once with different data (different sales, status, etc.)
- **Total duplicate rows detected:** 63

### 1.4 Discount Issues
- **Negative discount values:** 16 rows (e.g., -0.19, -0.23, -0.14) — discount cannot be negative
- **Discount > 50%:** 7 rows (e.g., 0.55, 0.65) — business rule cap is 50%
- **Percentage-formatted strings:** 8 rows with "70%", "85%" instead of decimal (0.70, 0.85)
- **Missing discount:** 18 rows with blank discount field

### 1.5 Date Issues
- **Multiple date formats in same column:** At least 8 different formats detected — `DD Mon YYYY`, `MM/DD/YYYY`, `YYYY-MM-DD`, `DD-MM-YYYY`, `DD/MM/YYYY`, and combinations
- **Ship date before order date:** 95 records where `ship_date < order_date` — logically invalid

### 1.6 Sales Calculation Mismatches
- **Records where calculated_sales ≠ recorded sales by more than ₹1:** 62 rows
- Root causes: negative discounts applied, missing discounts treated differently, % discount not converted

### 1.7 Order Status Inconsistencies
- Mixed case: `"completed"`, `"COMPLETED"`, `"Completed "`, `"  Completed "`
- Orders marked `Failed` payment but `Completed` status (2 records) — flagged
- `Refunded` payments with `Cancelled` status — treated as refunded

---

## 2. Cleaning Actions Performed

### 2.1 Text Standardization
**Action:** Applied to all text fields: `customer_name`, `segment`, `region`, `state`, `city`, `category`, `sub_category`, `ship_mode`, `payment_status`, `order_status`

Steps applied:
1. `str.strip()` — removed leading/trailing spaces
2. `re.sub(r'\s+', ' ', s)` — collapsed multiple internal spaces into one
3. `.title()` — converted to title case (first letter uppercase, rest lowercase)

**Result:** All text values standardized. "SMALL BUSINESS", "Small  Business", "  small business  " → all become "Small Business"

### 2.2 Missing Value Handling
- **region (26 missing):** Filled with `"Unknown"` as per business rule. Flagged in data_quality_report.xlsx.
- **ship_mode (22 missing):** Filled with `"Unknown"` as per business rule. Flagged in data_quality_report.xlsx.
- **discount (18 missing):** Treated as 0 where all other sales fields (quantity, unit_price, sales, cost, profit) appear valid. Flagged as `"Missing discount - treated as 0"` in `discount_flag` column.

### 2.3 Duplicate Handling
- **Exact duplicates (20 rows):** Removed. Kept first occurrence. These were 100% identical rows — no information lost.
- **Conflicting duplicate order_ids (12 order_ids, 24+ rows):** **NOT deleted.** Flagged with `dup_flag = "Conflicting duplicate - flagged for review"` so business stakeholders can resolve which version is correct.
- Logic: Never silently delete records where business intent is ambiguous.

### 2.4 Date Cleaning
- Implemented multi-format date parser supporting all 8+ formats found in the data
- Used `dayfirst=True` fallback to handle DD-MM-YYYY patterns
- Records where dates cannot be parsed: flagged in `date_flag` column
- Records where `ship_date < order_date`: flagged as `"Ship date before order date"` in `date_flag`; `shipping_delay_days` set to NaN

### 2.5 Discount Cleaning
- **Percentage strings (70%, 85%):** Converted to decimal (0.70, 0.85). These are above 50% so still flagged as invalid after conversion.
- **Negative discounts:** Retained original value in `discount` column but `cleaned_discount` uses absolute value for calculated_sales; flagged in `discount_flag`.
- **Discounts > 50%:** Capped at 50% for `calculated_sales` formula. Flagged as invalid.
- **cleaned_discount column:** Contains standardized, usable discount value. Negative and >50% values are flagged but calculation uses `clip(0, 0.5)`.

### 2.6 Calculated Columns Added
| Column | Formula |
|--------|---------|
| `cleaned_discount` | Parsed from raw discount; % strings converted to decimal |
| `calculated_sales` | `quantity × unit_price × (1 − cleaned_discount.clip(0, 0.5))` |
| `calculated_profit` | `calculated_sales − cost` |
| `profit_margin` | `calculated_profit / calculated_sales` |
| `shipping_delay_days` | `ship_date_clean − order_date_clean` in days; NaN for invalid date pairs |
| `order_month` | Month number extracted from `order_date_clean` |
| `order_year` | Year extracted from `order_date_clean` |
| `data_quality_flag` | Multi-condition flag: `Clean` or `Warning: [reasons]` |
| `date_flag` | Specific date issue description |
| `discount_flag` | Specific discount issue description |
| `dup_flag` | Duplicate conflict status |

---

## 3. Business Rules Applied

| Rule | Action |
|------|--------|
| Missing region | Filled with "Unknown"; flagged in quality report |
| Missing ship_mode | Filled with "Unknown"; flagged in quality report |
| Missing discount | Treated as 0 if other sales fields valid; flagged |
| Negative discount | Flagged as invalid; clipped to 0 for calculations |
| Discount > 50% | Flagged as invalid; capped at 50% for calculations |
| Cancelled orders | Excluded from completed sales pivot summary |
| Failed payments | Excluded from completed sales pivot summary |
| Refunded orders | Summarized separately in pivot; not in completed sales |
| Ship date before order date | Flagged; shipping_delay_days = NaN |
| Exact duplicate rows | Removed (20 rows); kept first occurrence |
| Conflicting duplicate order_ids | Flagged for review; not deleted |

---

## 4. Assumptions Made

1. **Date ambiguity:** Where date format is ambiguous (e.g., 06/08/2024 could be June 8 or August 6), the `dayfirst=True` parser was used as the primary interpretation, consistent with the majority Indian date format (DD/MM/YYYY). Some dates may still be misinterpreted.

2. **Discount cap at 50%:** Business rule assumed maximum valid discount is 50% (0.5). Any discount above this is flagged as invalid. This cap is not stated explicitly in the data but follows standard retail practice.

3. **Negative discounts are errors:** Negative discount values (-0.19, -0.23 etc.) are treated as data entry errors, not surcharges. The absolute value is not used — they are flagged and clipped to 0 in calculations.

4. **Missing discount = 0:** Where discount is blank, it is treated as "no discount applied" only when quantity, unit_price, sales, cost, and profit all appear valid. If the sales figure appears miscalculated, the mismatch is flagged separately.

5. **calculated_sales vs. recorded sales:** Where these differ by more than ₹1, both values are preserved. The `calculated_sales` column uses the standardized formula; the original `sales` column is kept unchanged.

6. **Conflicting duplicate resolution:** Conflicting order_ids with different sales or status values cannot be resolved by the cleaning process — they require business stakeholder input. Both records are retained and flagged.

7. **Order status normalization:** After title-casing, `"Completed"`, `"COMPLETED"`, `"completed"`, `"Completed "` all become `"Completed"`. Same for Cancelled and Returned.

---

## 5. Records Removed

| Reason | Count |
|--------|-------|
| Exact duplicate rows (all columns identical) | 20 |
| **Total records removed** | **20** |

No records were removed for any other reason. All other problematic records are retained with flags.

---

## 6. Records Flagged (Not Removed)

| Flag Type | Count |
|-----------|-------|
| Conflicting duplicate order_ids | ~24 rows across 12 order_ids |
| Ship date before order date | 95 |
| Invalid discount (negative or >50%) | 23 |
| Non-numeric discount (% strings) | 8 |
| Missing discount (treated as 0) | 18 |
| Sales calculation mismatch | 62 |
| Cancelled orders | 88 |
| Failed payment orders | 58 (approx) |
| Returned orders | ~70 |

---

## 7. Limitations of the Cleaning Process

1. **Date ambiguity cannot be fully resolved** without knowing the source system's date format. Records with ambiguous formats (DD/MM vs MM/DD) may have been parsed incorrectly.

2. **95 ship-before-order records could not be corrected** — it is impossible to determine which date is wrong without source system data. Both dates are preserved; the flag indicates the issue.

3. **Conflicting duplicate records require human resolution** — the cleaning process can identify but not resolve conflicting order_ids.

4. **Sales mismatches may have legitimate explanations** (price adjustments, manual overrides, bulk discounts not captured in the discount column) that cannot be determined from this dataset alone.

5. **Customer segments are not validated against a master list** — only text standardization was applied. "Small Business" may or may not be a valid segment in the company's CRM.

6. **No referential integrity checks** — customer_id, product_name, and state/city combinations are not validated against master tables since those are not provided.
