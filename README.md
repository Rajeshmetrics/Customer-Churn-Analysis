# Customer Churn Analysis

A Jupyter Notebook project that ingests raw customer/subscription/support data from Excel, stages it in SQLite, cleans and merges it with pandas, computes churn KPIs, and produces exploratory visualizations to understand **why customers churn**.

---

## 📁 Project Structure

```
churn-analysis/
├── churnAnalysis.ipynb              # Main analysis notebook
├── customer_churn_data_raw.xlsx     # Raw input workbook (multi-sheet)
├── customer_churn.db                # SQLite DB generated from the Excel sheets
├── exported_churn_data.csv          # Final cleaned & merged dataset (output)
├── requirements.txt                 # Python dependencies
├── assets/
│   └── workflow_diagram.png         # Workflow diagram (see below)
└── README.md
```

---

## 🔄 Workflow

The notebook follows a linear **ingest → clean → merge → analyze → visualize** pipeline:


| Step | Description |
|---|---|
| **1. Data Ingestion** | `customer_churn_data_raw.xlsx` is read sheet-by-sheet with `pandas.ExcelFile` and written into a local SQLite database (`customer_churn.db`), one table per sheet. |
| **2. Data Extraction** | Table names are queried from `sqlite_master`, and each table is loaded back into a pandas DataFrame (`df_db_customer`, `df_db_subscription`, `df_db_support`), with schemas inspected via `PRAGMA table_info`. |
| **3. Data Cleaning** | Column renaming (`name` → `customer_name`), dropping unused columns (`interests`, `pincode`, `col_1`, `comment`), datetime conversion (`dob`, `subscription_start_date`, `renewal_date`, `cancellation_date`, `complaint_date`), gender standardization (`Men/Women` → `Male/Female`), and filling missing `country` values via a `state → country` lookup. |
| **4. Feature Engineering** | Derives `churn_flag` (1 if `cancellation_date` is present), aggregates `complaint_count` per customer, and deduplicates the support table to one row per customer (latest complaint kept). |
| **5. Merge Datasets** | Left-joins `subscription` + `customer` + `support` on `customerid` into a single analysis-ready DataFrame `df`. |
| **6. Export** | The merged dataset is saved to `exported_churn_data.csv`. |
| **7. KPI Calculation** | Computes churn rate, retention rate, ARPU, average tenure (days), revenue at risk, escalation rate, average complaints per user, and the correlation between escalations and churn. |
| **8. Risk Segmentation** | Buckets `churn_score` into `churn_risk` categories (`low` < 50, `med` 50–69, `high` ≥ 70). |
| **9. EDA & Visualization** | Monthly churn trend, churn rate by plan type and by state, a correlation heatmap (encoded categorical + numeric features), a pairplot, a `seaborn.catplot` of charges by plan/gender/risk, and pivot tables summarizing churn and revenue by plan. |

---

## 📊 Key Metrics (KPIs) Computed

- **Churn Rate** — % of subscriptions with a `churn_flag` of 1
- **Retention Rate** — `100 − churn rate`
- **Churn Rate by Plan Type** — grouped mean of `churn_flag`
- **ARPU** (Average Revenue Per User) — mean of `monthly_charges`
- **Average Tenure (days)** — days from `subscription_start_date` to `cancellation_date` (or today, if still active)
- **Revenue at Risk** — sum of `monthly_charges` for churned customers
- **Escalation Rate** — % of records flagged `escalations == 'Y'`
- **Avg Complaints per User** — total complaints ÷ unique customers
- **Escalation–Churn Correlation** — Pearson correlation between `escalations` and `churn_flag`

---

## 🧰 Requirements

- **Python** 3.9+
- **Jupyter Notebook** or JupyterLab

### Python packages

| Package | Purpose |
|---|---|
| `pandas` | Data loading, cleaning, merging, pivot tables |
| `numpy` | Numeric operations, conditional logic (`np.where`, `np.select`) |
| `matplotlib` | Line/bar charts, custom heatmap |
| `seaborn` | Heatmap, pairplot, catplot |
| `openpyxl` | Reading the `.xlsx` source workbook (pandas Excel engine) |
| `sqlite3` | Built into the Python standard library — staging database |

Install everything with:

```bash
pip install -r requirements.txt
```

`requirements.txt`:
```
pandas
numpy
matplotlib
seaborn
openpyxl
jupyter
```

---

## ▶️ How to Run

1. Place the raw workbook `customer_churn_data_raw.xlsx` in the project root (must contain `customer`, `subscription`, and `support` sheets — column names are referenced explicitly in the cleaning steps).
2. Install dependencies: `pip install -r requirements.txt`
3. Launch the notebook: `jupyter notebook churnAnalysis.ipynb`
4. Run cells top to bottom. The notebook will:
   - Create `customer_churn.db`
   - Print the discovered tables and their columns
   - Produce `exported_churn_data.csv`
   - Print all KPIs to output
   - Render each chart inline

---

## 📝 Notes / Known Issues in the Current Notebook

A few things worth cleaning up if you plan to re-run or hand this off:

- **Ingestion cell has a stray statement** — the cell that writes Excel sheets to SQLite ends with a bare `p`, which will raise a `NameError` on execution; remove that line.
- **`SettingWithCopyWarning` risk** — the categorical-encoding cells (`df_encoded[col] = ...`) assign into a slice of `df_visual`; use `.copy()` when creating `df_encoded` to silence/avoid this.
- **Truncated cell** — the manual correlation-heatmap-annotation cell (using `ax.imshow`) is cut off mid-loop and won't run as-is; either complete the loop or rely on the earlier `sns.heatmap` cell, which already covers this.
- **Unrelated demo cells at the end** — the final few cells create a `test_database.sqlite` with unrelated `users` sample data; these aren't part of the churn pipeline and can be removed from a production version.
- **Duplicate merge** — the merge step (`df = subscription.merge(...)`) is run twice in the notebook (before and after support-table deduplication); only the second run (post-deduplication) is used downstream, so the first can be deleted.

---

## 📌 Data Sources (expected input schema)

| Table | Key columns used |
|---|---|
| `customer` | `customerid`, `name`, `dob`, `gender`, `state`, `country`, `interests`, `pincode` |
| `subscription` | `customerid`, `plan_type`, `contract_type`, `monthly_charges`, `subscription_start_date`, `renewal_date`, `cancellation_date`, `churn_score` |
| `support` | `customerid`, `complaint_date`, `escalations`, `col_1`, `comment` |
