# B2B CRM Data Report — Power BI Project

A Power BI report built on two noisy real-world-style datasets covering B2B companies and their employees. The project covers end-to-end data cleaning in Power Query (M), DAX measure development, and an interactive dashboard.

---

## Datasets

| Table | Rows | Description |
|---|---|---|
| `companies_noisy_734.csv` | 734 | Company-level data: revenue, marketing spend, leads, contracts |
| `employees_noisy_5234.csv` | 5,234 | Employee-level data: job titles, departments, contact info, scores |

---

## Data Cleaning — Companies Table

All cleaning was done in Power Query using M code.

**1. Type casting & renaming**
Initial types were assigned to all 19 columns and snake_case column names were renamed to readable labels.

**2. Trim all text columns**
`Text.Trim` applied to all text columns to remove leading/trailing whitespace.

**3. Value normalization — categorical columns**
Typos, casing inconsistencies, and encoding errors were fixed across 7 columns using `List.Accumulate` + `Table.ReplaceValue` (exact match) and lookup-based replacement lists:

- `Industry` — 21 replacements (e.g. `"Aerospac"` → `"Aerospace"`, `"OIL & GAS"` → `"Oil & gas"`)
- `Company Size` — normalized casing (`"large"` / `"SMALL"` → proper case)
- `Campaign Type` — 17 replacements (e.g. `"EMAIL"`, `"emal"`, `"gEM"` → `"Email"`)
- `Region` — 21 replacements (e.g. `"Ankaa"` → `"Ankara"`, `"dana"` → `"Adana"`)
- `District` — 21 replacements (e.g. `"ebze"` → `"Gebze"`, `"Ihmit"` → `"Izmit"`)
- `Contract Status` — 6 replacements (casing only)
- `Payment Behavior` — 5 replacements (e.g. `"on-time"` / `"ON-TIME"` → `"On-Time"`)
- `Preferred Channel` — 6 replacements (casing and typos)

**4. Marketing Spend scale fix**
Some values were stored in full units instead of thousands. The `"K"` suffix was stripped, the column cast to integer, and values truncated to 2 digits to normalize scale.

**5. Days Since Last Purchase**
`"N/A"` text values replaced with empty before casting the column to `Int64`.

**6. Missing value imputation**

| Column | Strategy |
|---|---|
| Annual Revenue | Median |
| Marketing Spend | Median |
| Leads Generated | Mean (rounded) |
| Conversion Rate | Median |
| Days Since Last Purchase | Median |
| Total Purchases Last Year | Median |
| District | Mode |
| Payment Behavior | Mode |
| Preferred Channel | Mode |

---

## Data Cleaning — Employees Table

**1. Type casting, trimming & renaming**
All 21 columns typed, trimmed, and renamed. `Campaign Response Rate (%)` temporarily cast to text for trimming then re-cast to decimal.

**2. Null standardization**
`"unknown"` replaced with `null` in date columns. `"N/A"` replaced with `null` in `Tenure Years`, `Event Attendance`, and `Influence Score` before numeric casting.

**3. Campaign Response Rate fix**
Values >= 1 were divided by 100 to normalize percentage scale (e.g. `73` → `0.73`).

**4. Value normalization — categorical columns**
Used record-based single-pass lookup (`Record.HasFields` + `Record.Field`) for performance on large replacement lists:

- `Department` — 100+ replacements via `List.Accumulate` (e.g. `"ales"` → `"Sales"`, `"ENGINEERING"` → `"Engineering"`)
- `Job Title` — 130+ replacements via record lookup (e.g. `"accont manager"` → `"Account Manager"`, `"wnfrastructure Engineer"` → `"Infrastructure Engineer"`)
- `Seniority Level` — 9 replacements (casing normalization)
- `Work Location` — casing and typo fixes
- `Education Level` — casing normalization
- `Preferred Contact Method` — typo and casing fixes
- `Language` — typo fixes
- `Data Source` — 80+ replacements normalizing CRM variants, Webinar, LinkedIn, Email Campaign, Fair

**5. Missing value imputation**

| Column | Strategy |
|---|---|
| Campaign Response Rate | Median |
| Tenure Years | Median |
| Event Attendance | Median |
| Influence Score | Median |
| Education Level | Mode |
| Newsletter Subscription | Mode |
| Preferred Contact Method | Mode |
| Language | Mode |
| Data Source | Mode |

**6. Date column imputation**
`Last Contact Date` and `Next Followup Date` were not sequential so forward/backward fill was not appropriate. Strategy used:
- If `Next Followup Date` is null → estimated as `Last Contact Date + 30 days`
- If `Last Contact Date` is null → estimated as `Next Followup Date Fixed - 30 days`
- Both fixed columns added as new columns; originals removed

**7. Column removal**
`Name`, `Owner Rep`, original date columns removed from the final output.

---

## DAX Measures

| Measure | Description |
|---|---|
| `Contacts` | Total row count of employees |
| `Decision Makers` | Count of contacts where `Decision Maker Flag = "Yes"` |
| `Decision Maker Rate` | `Decision Makers / Contacts` |
| `Decision Maker Rate LM` | Decision Maker Rate for previous month |
| `High Influence Contacts` | Contacts with Influence Score > 1.2× average |
| `High Influence Contacts Rate` | `High Influence Contacts / Contacts` |
| `High Influence Contacts Rate LM` | High Influence Rate for previous month |
| `AVG Campaign Response Rate` | Average of `Campaign Response Rate (%)` |
| `AVG Campaign Response Rate LM` | AVG Campaign Response Rate for previous month |
| `AVG Conversion Rate` | Average of `Conversion Rate (%)` |
| `AVG Influence Score` | Average of `Influence Score` |
| `Revenue` | Sum of `Annual Revenue (M₺)` |
| `Marketing Spend` | Sum of `Marketing Spend (K₺)` |
| `Marketing ROI` | `Revenue / (Marketing Spend / 1000)` |
| `Leads Generated` | Sum of `Leads Generated` |
| `Active Contracts` | Distinct company count where `Contract Status = "Active"` |
| `Expired Contracts` | Distinct company count where `Contract Status = "Expired"` |
| `Total Tenure Years` | Sum of `Tenure Years` |

---

## Dashboard

The report is structured across 4 pages, each serving a distinct analytical purpose.

---

### Page 1 — Executive View

The main summary page giving a high-level overview of business performance. Includes a date range slicer to filter all visuals by time period.

- **3 KPI cards** — High Influence Contacts %, AVG Campaign Response Rate %, and Decision Maker Rate %, each showing the current value, a goal benchmark, and month-over-month change with a directional indicator
- **Donut chart** — Breakdown of contacts by Preferred Contact Method (Email, Phone, LinkedIn, Events)
- **Horizontal bar chart** — AVG Campaign Response % ranked by Campaign Type (Webinar, LinkedIn Ads, SEM, Content Marketing, Trade Show, Email)
- **6 summary metric cards** — Revenue, Marketing Spend, Leads Generated, Contacts, Marketing ROI, AVG Conversion Rate
- **Bar chart** — AVG Conversion Rate % across all industries
- **Line chart** — Active vs Expired Contracts tracked over time with a trend line

---

### Page 2 — Decision Maker Key Influencers

An analytical page using Power BI's built-in Key Influencers visual to explore what drives a contact to be flagged as a decision maker.

- Identifies the top factors that increase the likelihood of `Decision Maker Flag = "Yes"`
- Top influencers found include Seniority Level being Senior (2.01×), Department being Sales (1.81×), Management (1.78×), and Procurement (1.77×), and Seniority Level being Director (1.58×)
- The **Top Segments** tab groups contacts into segments with the highest concentration of decision makers
- Includes a supporting bar chart showing Decision Maker % broken down by Seniority Level, highlighting the gap between Senior/Director and Mid/Junior levels

---

### Page 3 — Revenue Decomposition Tree

An interactive decomposition tree allowing users to drill into Revenue from the top level down through multiple dimensions.

- Starts from total Revenue and breaks down by Industry, then Region, then Campaign Type, then Company Size, then Preferred Channel
- Allows the user to explore which specific combinations of industry, region, and campaign type drive the most revenue
- Example path: Utilities → Sakarya → Trade Show → Medium company size = 44.10M₺

---

### Page 4 — Leads Generated Decomposition Tree

Same decomposition structure applied to Leads Generated, giving a demand generation perspective alongside the revenue view.

- Drills from total Leads Generated down through Campaign Type, Region, Industry, and Preferred Channel
- Useful for identifying which campaign and region combinations generate the highest lead volumes
- Example path: Webinar → Kayseri → Aerospace → Sales Rep channel = 10 leads
