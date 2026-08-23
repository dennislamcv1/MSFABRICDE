# Analytics-Ready Dataset — Microsoft Fabric Lakehouse Lab

Companion guide for `customers.csv` and `sales_raw.csv`.

The two files match the published schema exactly — no extra columns. The data is
deliberately imperfect: five classes of defect are planted so the *inspect and
validate* phase of the lab produces real findings rather than a clean pass.

Generated with `generate_dataset.py` (seed `20260822`). Re-running the script
reproduces the identical files.

---

## 1. Files

| File | Rows (incl. header) | Purpose |
|---|---|---|
| `customers.csv` | 516 | Customer dimension |
| `sales_raw.csv` | 5,026 | Transaction fact table |
| `generate_dataset.py` | — | Regenerates both files deterministically |
| `verify_dataset.py` | — | Runs every validation query and asserts the counts below |
| `defect_manifest.json` | — | Machine-readable answer key |

Date coverage: **2024-01-01 to 2025-12-31** (two full years, so year-over-year
comparisons work).

---

## 2. Data dictionary

### `sales_raw`

| Column | Declared type | Actual type on CSV load | Notes |
|---|---|---|---|
| `transaction_id` | STRING | STRING | `TXN-000001` … `TXN-005000`. **Not unique — 25 ids repeat.** |
| `customer_id` | STRING | STRING | Foreign key to `customers`. **Format is not consistent.** |
| `amount` | DECIMAL | **STRING** | Contains blanks, negatives and currency text, so schema inference will not give you a numeric column. |
| `transaction_date` | DATE | **STRING** | Mostly `yyyy-MM-dd`, with 35 rows in `dd/MM/yyyy` and 15 blanks. |
| `region` | STRING | STRING | 31 distinct literals for what are really 5 regions. |

The gap between the *declared* and *actual* types is intentional. Discovering
that `sales_raw` cannot be aggregated until it is cast is the first finding of
the lab.

### `customers`

| Column | Declared type | Actual type on CSV load | Notes |
|---|---|---|---|
| `customer_id` | STRING | STRING | `CUST-00001` … `CUST-00500`. **Not unique — 15 ids appear twice.** |
| `customer_name` | STRING | STRING | 15 rows blank. |
| `customer_segment` | STRING | STRING | Consumer / Corporate / Small Business / Enterprise. 18 rows blank. |
| `region` | STRING | STRING | 11 rows blank; 22 rows carry case or whitespace variants. |

---

## 3. Loading into the Lakehouse

1. Create a workspace on a Fabric-enabled capacity, then **New → Lakehouse**
   (e.g. `lh_retail_ops`).
2. In the Lakehouse explorer, expand **Files**, create a subfolder `landing`,
   and upload both CSVs.
3. Right-click each file → **Load to Tables → New table**. Accept the inferred
   schema. Name them `sales_raw` and `customers`.
4. Confirm both appear under **Tables** as Delta tables, then query them from
   the SQL analytics endpoint or a notebook.

Loading with inferred types is the point — it is what produces the string
`amount` column that the validation step has to catch. If you would rather teach
explicit schema control, load via a notebook with an explicit `StructType`
instead and let students compare the two outcomes.

---

## 4. Answer key — what is planted, and what it does

All counts below were produced by `verify_dataset.py` against the shipped files.

### 4.1 Duplicate joins

15 `customer_id` values appear **twice** in `customers`, as if the customer
re-registered: same id, drifted name, segment, and sometimes region. Nine of the
fifteen are high-volume buyers, so the fan-out is large.

| Measure | Value |
|---|---|
| Rows in `customers` | 515 |
| Distinct `customer_id` | 500 |
| Ids appearing more than once | 15 |
| Rows in `sales_raw` | 5,025 |
| **Rows returned by a naive inner join** | **5,295** |

A join that returns more rows than the fact table is the tell. Revenue per
segment is silently overstated for the affected customers because their
transactions are counted twice.

### 4.2 Inconsistent keys

175 rows in `sales_raw` (3.5%) fail an exact match against `customers`:

| Variant | Rows | Example |
|---|---|---|
| Lowercased | 60 | `cust-00042` |
| Leading / trailing whitespace | 40 | `CUST-00042 ` |
| Zero padding lost | 30 | `CUST-42` |
| Orphan — no such customer | 45 | `CUST-93817` |

Only the last 45 are genuinely unmatchable. The other 130 rows are recoverable
by normalising the key — which is exactly the distinction students should reach.
An inner join silently drops all 175; a left join exposes them.

### 4.3 Incomplete customer records

| Issue | Rows |
|---|---|
| Blank `customer_name` | 15 |
| Blank `customer_segment` | 18 |
| Blank `region` | 11 |

Blank segments matter most: `GROUP BY customer_segment` produces an unlabelled
bucket, and segment percentages will not sum to the reported total.

### 4.4 Invalid aggregations

| Issue | Rows | Effect |
|---|---|---|
| Blank `amount` | 40 | `COUNT(*)` = 5,025 but `COUNT(amount)` = 4,960 — averages disagree |
| Negative `amount` | 30 | Refunds not flagged; netted into revenue without comment |
| `amount` stored as text | 25 | `$340.00`, `1,250.06`, `780.00 USD` — `TRY_CAST` returns NULL |
| Extreme outliers | 6 | Up to `999999.99`; six rows contribute ~1.37M |
| Duplicated `transaction_id` | 25 | Double counting |

Headline figures (Spark cast semantics):

| Query | Result |
|---|---|
| `SUM(amount)` as loaded | **2,903,501.82** |
| `SUM` through the naive join | **3,031,919.72** |
| `SUM` after dedupe, cast and outlier removal | **1,533,214.74** |

The naive number overstates revenue by roughly **90%**, and the joined number by
**98%**. That gap is the deliverable of the validation phase.

### 4.5 Inconsistent filtering behaviour

`region` in `sales_raw` has **31 distinct literals** for 5 real regions:

- Case variants — `North`, `north`, `NORTH`
- Whitespace — `" North"`, `"North "`
- Single-letter codes — `N`, `S`, `E`, `W`, `C` (86 rows)
- Blank — 30 rows

`WHERE region = 'North'` returns a materially different row set than
`WHERE UPPER(TRIM(region)) = 'NORTH'`, and both differ from filtering on the
customer's assigned region. After normalisation, **319 joined rows** have a
transaction region that disagrees with the customer's home region — a real
modelling question (is region an attribute of the sale or of the customer?)
rather than a defect to be scrubbed. The lab should force that decision to be
made explicitly.

### 4.6 Date handling

| Issue | Rows |
|---|---|
| `dd/MM/yyyy` instead of `yyyy-MM-dd` | 35 |
| Blank | 15 |

A single-format `to_date()` nulls the 35 slash-formatted rows, quietly dropping
them from any period filter.

---

## 5. Validation queries

Run these against the SQL analytics endpoint or in a notebook with `%%sql`.
Expected results are in section 4.

```sql
-- 1. Grain check: is transaction_id actually unique?
SELECT COUNT(*)                        AS total_rows,
       COUNT(DISTINCT transaction_id)  AS distinct_ids,
       COUNT(*) - COUNT(DISTINCT transaction_id) AS surplus_rows
FROM   sales_raw;

-- 2. Grain check on the dimension: duplicate customer keys
SELECT customer_id, COUNT(*) AS row_count
FROM   customers
GROUP  BY customer_id
HAVING COUNT(*) > 1
ORDER  BY row_count DESC, customer_id;

-- 3. Fan-out proof: does the join inflate the row count?
SELECT (SELECT COUNT(*) FROM sales_raw) AS fact_rows,
       COUNT(*)                         AS joined_rows
FROM   sales_raw s
JOIN   customers c ON s.customer_id = c.customer_id;

-- 4. Referential integrity: transactions with no matching customer
SELECT COUNT(*) AS unmatched_rows
FROM   sales_raw s
LEFT   JOIN customers c ON s.customer_id = c.customer_id
WHERE  c.customer_id IS NULL;

-- 5. Classify the key problems
SELECT CASE
         WHEN customer_id <> TRIM(customer_id)                    THEN 'whitespace'
         WHEN customer_id = LOWER(customer_id)                    THEN 'lowercase'
         WHEN LENGTH(TRIM(customer_id)) < 10                      THEN 'padding lost'
         ELSE 'other'
       END AS key_issue,
       COUNT(*) AS rows
FROM   sales_raw s
WHERE  NOT EXISTS (SELECT 1 FROM customers c
                   WHERE c.customer_id = s.customer_id)
GROUP  BY 1
ORDER  BY rows DESC;

-- 6. True orphans, after normalising the key
WITH s AS (
  SELECT CONCAT('CUST-',
           LPAD(SPLIT(UPPER(TRIM(customer_id)), '-')[1], 5, '0')) AS ck
  FROM sales_raw
)
SELECT COUNT(*) AS true_orphans
FROM   s
WHERE  NOT EXISTS (SELECT 1 FROM customers c
                   WHERE UPPER(TRIM(c.customer_id)) = s.ck);

-- 7. Completeness profile of the dimension
SELECT SUM(CASE WHEN customer_name    IS NULL OR customer_name    = '' THEN 1 ELSE 0 END) AS missing_name,
       SUM(CASE WHEN customer_segment IS NULL OR customer_segment = '' THEN 1 ELSE 0 END) AS missing_segment,
       SUM(CASE WHEN region           IS NULL OR region           = '' THEN 1 ELSE 0 END) AS missing_region
FROM   customers;

-- 8. Why amount cannot be aggregated as loaded
SELECT COUNT(*)                                         AS total_rows,
       SUM(CASE WHEN amount = '' THEN 1 ELSE 0 END)     AS blank_amount,
       SUM(CASE WHEN TRY_CAST(amount AS DECIMAL(12,2)) IS NULL
                 AND amount <> '' THEN 1 ELSE 0 END)    AS uncastable_text,
       SUM(CASE WHEN TRY_CAST(amount AS DECIMAL(12,2)) < 0 THEN 1 ELSE 0 END) AS negatives,
       MAX(TRY_CAST(amount AS DECIMAL(12,2)))           AS max_amount
FROM   sales_raw;

-- 9. The three revenue numbers
SELECT SUM(TRY_CAST(amount AS DECIMAL(12,2))) AS revenue_as_loaded
FROM   sales_raw;

SELECT SUM(TRY_CAST(s.amount AS DECIMAL(12,2))) AS revenue_through_naive_join
FROM   sales_raw s
JOIN   customers c ON s.customer_id = c.customer_id;

WITH deduped AS (
  SELECT DISTINCT transaction_id, amount FROM sales_raw
)
SELECT ROUND(SUM(TRY_CAST(amount AS DECIMAL(12,2))), 2) AS revenue_cleaned
FROM   deduped
WHERE  TRY_CAST(amount AS DECIMAL(12,2)) BETWEEN 0 AND 40000;

-- 10. Region cardinality: 31 literals for 5 regions
SELECT region, COUNT(*) AS rows
FROM   sales_raw
GROUP  BY region
ORDER  BY rows DESC;

-- 11. Filtering behaviour changes with normalisation
SELECT SUM(CASE WHEN region = 'North' THEN 1 ELSE 0 END)              AS exact_match,
       SUM(CASE WHEN UPPER(TRIM(region)) = 'NORTH' THEN 1 ELSE 0 END) AS trimmed_match,
       SUM(CASE WHEN UPPER(TRIM(region)) IN ('NORTH','N') THEN 1 ELSE 0 END) AS with_codes
FROM   sales_raw;

-- 12. Fact region vs dimension region
SELECT COUNT(*) AS disagreeing_rows
FROM   sales_raw s
JOIN   customers c
  ON   UPPER(TRIM(s.customer_id)) = UPPER(TRIM(c.customer_id))
WHERE  UPPER(TRIM(s.region)) <> UPPER(TRIM(c.region))
  AND  TRIM(s.region) <> '' AND TRIM(c.region) <> ''
  AND  LENGTH(TRIM(s.region)) > 1;

-- 13. Date parseability
SELECT SUM(CASE WHEN transaction_date = '' THEN 1 ELSE 0 END)      AS blank_dates,
       SUM(CASE WHEN transaction_date LIKE '%/%' THEN 1 ELSE 0 END) AS slash_format,
       MIN(COALESCE(TO_DATE(transaction_date, 'yyyy-MM-dd'),
                    TO_DATE(transaction_date, 'dd/MM/yyyy')))       AS min_date,
       MAX(COALESCE(TO_DATE(transaction_date, 'yyyy-MM-dd'),
                    TO_DATE(transaction_date, 'dd/MM/yyyy')))       AS max_date
FROM   sales_raw;
```

`TRY_CAST` and `SPLIT` require the Spark SQL dialect (notebook `%%sql` cell or a
Lakehouse SQL endpoint on Fabric Runtime 1.2+). If you are on the warehouse
T-SQL endpoint instead, substitute `TRY_CONVERT` and `PARSENAME` / `STRING_SPLIT`.

---

## 6. Suggested curated layer

The natural conclusion of the lab: build `dim_customer` and `fact_sales` that
resolve every issue above, then re-run query 9 and confirm the three revenue
numbers converge.

```sql
CREATE OR REPLACE TABLE dim_customer AS
SELECT UPPER(TRIM(customer_id))                                   AS customer_key,
       MAX(NULLIF(TRIM(customer_name), ''))                       AS customer_name,
       COALESCE(MAX(NULLIF(TRIM(customer_segment), '')), 'Unknown') AS customer_segment,
       COALESCE(INITCAP(LOWER(MAX(NULLIF(TRIM(region), '')))), 'Unknown') AS home_region
FROM   customers
GROUP  BY UPPER(TRIM(customer_id));   -- collapses the 15 duplicate keys

CREATE OR REPLACE TABLE fact_sales AS
WITH deduped AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY transaction_id ORDER BY amount DESC) AS rn
  FROM   sales_raw
),
typed AS (
  SELECT transaction_id,
         CONCAT('CUST-', LPAD(SPLIT(UPPER(TRIM(customer_id)), '-')[1], 5, '0')) AS customer_key,
         TRY_CAST(amount AS DECIMAL(12,2)) AS amount,
         COALESCE(TO_DATE(transaction_date, 'yyyy-MM-dd'),
                  TO_DATE(transaction_date, 'dd/MM/yyyy')) AS transaction_date,
         CASE UPPER(TRIM(region))
           WHEN 'N' THEN 'North'  WHEN 'S' THEN 'South'
           WHEN 'E' THEN 'East'   WHEN 'W' THEN 'West'
           WHEN 'C' THEN 'Central'
           WHEN ''  THEN 'Unknown'
           ELSE INITCAP(LOWER(TRIM(region)))
         END AS sale_region
  FROM   deduped
  WHERE  rn = 1
)
SELECT t.*,
       CASE WHEN d.customer_key IS NULL THEN TRUE ELSE FALSE END AS is_orphan,
       CASE WHEN t.amount < 0           THEN TRUE ELSE FALSE END AS is_refund,
       CASE WHEN t.amount > 40000       THEN TRUE ELSE FALSE END AS is_outlier
FROM   typed t
LEFT   JOIN dim_customer d ON t.customer_key = d.customer_key;
```

Flagging rather than deleting the orphans, refunds and outliers keeps the
lineage auditable — a defensible choice students should be asked to justify.

---

## 7. Suggested marking checkpoints

1. Lakehouse created; both tables visible under **Tables** as Delta.
2. Student reports that `amount` did **not** load as a numeric type, and says why.
3. `transaction_id` identified as non-unique — 25 duplicates found.
4. `customer_id` identified as non-unique in the dimension — 15 duplicates found,
   and the join fan-out (5,295 > 5,025) demonstrated.
5. The 175 unmatched keys found, and split into the 130 recoverable and 45 true orphans.
6. Region cardinality of 31 reported, and reduced to 5.
7. Three revenue figures produced, with the ~90% overstatement explained.
8. A curated layer built, with a written justification for each cleaning decision.
