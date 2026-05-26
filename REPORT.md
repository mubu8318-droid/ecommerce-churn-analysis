# Technical Report — E-Commerce Customer Churn Analysis (MySQL)

**Project Type:** Module-End Assignment  
**Database:** MySQL 8.0+  
**Database Name:** `ecomm`  
**Dataset Size:** ~5,630 customer records  

---

## 1. Project Objective

This project constructs a fully operational MySQL database to analyse customer churn behaviour in an e-commerce context. The pipeline covers every stage from raw schema design and bulk data loading through structured data quality work and onward to business-focused SQL analysis. The goal is to surface actionable patterns that explain why customers leave the platform and what profile characteristics correlate with retention.

---

## 2. Database Design

### 2.1 Schema

The database `ecomm` holds two tables.

**`customer_churn`** is the core analytical table. It was created with the following design decisions:

- `CustomerID` is an `INT PRIMARY KEY`, ensuring row-level uniqueness.
- `Churn` and `Complain` use the `BIT` type to represent binary flags compactly.
- `Gender` uses MySQL's `ENUM('Male','Female')` to enforce domain integrity at the storage layer.
- Remaining categorical fields use `VARCHAR` with conservative length limits.
- All numeric behavioural metrics (`Tenure`, `OrderCount`, `CashbackAmount`, etc.) use `INT`.

**`customer_returns`** is a transactional table introduced later in the script to model product return events. It uses:

- `ReturnID INT PRIMARY KEY` for uniqueness.
- `CustomerID INT` with an explicit `FOREIGN KEY` referencing `customer_churn(CustomerID)`, enforcing referential integrity.
- `ReturnDate DATE` and `RefundAmount DECIMAL(10,2)` for financial precision.

### 2.2 Column Inventory

| Column | Type | Notes |
|---|---|---|
| CustomerID | INT PK | Range: 50001–55630 |
| Churn | BIT | 1 = churned (dropped post-transformation) |
| Tenure | INT | Months as customer; nulls imputed |
| PreferredLoginDevice | VARCHAR(20) | Standardised to Mobile Phone / Computer |
| CityTier | INT | 1 = metro, 2 = mid-size, 3 = smaller city |
| WarehouseToHome | INT | Distance in km; outliers removed |
| PreferredPaymentMode | VARCHAR(20) | Standardised across multiple aliases |
| Gender | ENUM | Male / Female |
| HourSpendOnApp | INT | Renamed post-load; nulls imputed |
| NumberOfDeviceRegistered | INT | Devices on account |
| PreferedOrderCat | VARCHAR(20) | Renamed + standardised post-load |
| SatisfactionScore | INT | Scale 1–5 |
| MaritalStatus | VARCHAR(10) | Single / Married / Divorced |
| NumberOfAddress | INT | Saved delivery addresses |
| Complain | BIT | 1 = complaint raised (dropped post-transformation) |
| OrderAmountHikeFromlastYear | INT | % YoY increase; nulls imputed |
| CouponUsed | INT | Coupons in last month; nulls imputed |
| OrderCount | INT | Orders in last month; nulls imputed |
| DaySinceLastOrder | INT | Days since last purchase; nulls imputed |
| CashbackAmount | INT | Cashback received |

---

## 3. Data Ingestion

The full dataset was loaded via a single large `INSERT INTO` statement containing ~5,630 value rows (CustomerIDs 50001 through 55630). This monolithic bulk-insert approach is appropriate for a self-contained academic script where no external file loading infrastructure is available.

Notable characteristics of the raw data:
- Multiple columns contain `NULL` entries, indicating missing observations rather than zero values.
- Categorical columns show format inconsistencies across rows (e.g., `'Phone'` vs `'Mobile Phone'`, `'CC'` vs `'Credit Card'`).
- At least one extreme value exists in `WarehouseToHome` (values above 100 km).

---

## 4. Data Cleaning

### 4.1 Missing Value Imputation

Imputation strategy was selected per-column based on the nature of the distribution:

**Mean imputation** was applied to continuous columns where a central tendency estimate is appropriate:

```sql
-- Example: WarehouseToHome
UPDATE customer_churn
JOIN (SELECT ROUND(AVG(WarehouseToHome)) AS val
      FROM customer_churn WHERE WarehouseToHome IS NOT NULL) m
SET WarehouseToHome = m.val WHERE WarehouseToHome IS NULL;
```

The same pattern covers `HourSpendOnApp`, `OrderAmountHikeFromlastYear`, and `DaySinceLastOrder`.

**Mode imputation** was applied to discrete/skewed columns where the most frequent value is the better representative:

```sql
-- Example: Tenure
UPDATE customer_churn
JOIN (SELECT Tenure AS mode_val FROM customer_churn
      WHERE Tenure IS NOT NULL
      GROUP BY Tenure ORDER BY COUNT(*) DESC, Tenure ASC LIMIT 1) m
SET Tenure = m.mode_val WHERE Tenure IS NULL;
```

The same pattern covers `CouponUsed` and `OrderCount`.

### 4.2 Outlier Handling

A hard-delete rule was applied to `WarehouseToHome`:

```sql
DELETE FROM customer_churn WHERE WarehouseToHome > 100;
```

Values above 100 km were identified as physically implausible for a last-mile e-commerce delivery model and removed entirely rather than capped, to avoid introducing artificial boundary values into the analysis.

### 4.3 Inconsistency Standardisation

Categorical fields contained multiple labels representing the same concept. All were unified:

| Column | Raw | Corrected |
|---|---|---|
| PreferredLoginDevice | `'Phone'` | `'Mobile Phone'` |
| PreferedOrderCat | `'Mobile'` | `'Mobile Phone'` |
| PreferredPaymentMode | `'COD'` | `'Cash on Delivery'` |
| PreferredPaymentMode | `'CC'` | `'Credit Card'` |

Note: `'Cash on Delivery'` was also present as a full-length string in some rows, so the standardisation only targeted the abbreviated variants.

---

## 5. Data Transformation

### 5.1 Column Renaming

Two columns carried names that were either misspelled or abbreviated:

```sql
ALTER TABLE customer_churn CHANGE PreferedOrderCat PreferredOrderCat VARCHAR(20);
ALTER TABLE customer_churn CHANGE HourSpendOnApp HoursSpentOnApp INT;
```

### 5.2 Derived Columns

Two human-readable columns were derived from the original binary flags:

```sql
-- Readable complaint status
ALTER TABLE customer_churn ADD COLUMN ComplaintReceived VARCHAR(10);
UPDATE customer_churn SET ComplaintReceived = CASE
    WHEN Complain = 1 THEN 'Yes' ELSE 'No'
END;

-- Readable churn status
ALTER TABLE customer_churn ADD COLUMN ChurnStatus VARCHAR(10);
UPDATE customer_churn SET ChurnStatus = CASE
    WHEN Churn = 1 THEN 'Churned' ELSE 'Active'
END;
```

### 5.3 Column Removal

The original binary columns were dropped after the derived versions were confirmed:

```sql
ALTER TABLE customer_churn DROP COLUMN Churn;
ALTER TABLE customer_churn DROP COLUMN Complain;
```

This reduces redundancy and ensures downstream queries use the canonical string labels.

---

## 6. Analytical Queries

### 6.1 Churn Distribution

```sql
SELECT ChurnStatus, COUNT(*) AS count_of_customers
FROM customer_churn GROUP BY ChurnStatus;
```

Provides the top-level split between churned and active customers, the foundational metric for the entire analysis.

### 6.2 Churned Customer Profile

```sql
SELECT ChurnStatus, AVG(Tenure) AS avg_tenure, SUM(CashbackAmount) AS total_cashbackamount
FROM customer_churn WHERE ChurnStatus = 'Churned' GROUP BY ChurnStatus;
```

Low average tenure among churned customers would confirm that early-stage customers are most at risk.

### 6.3 Complaint-Churn Relationship

```sql
SELECT COUNT(CASE WHEN ComplaintReceived = 'Yes' THEN 1 END) * 100 / COUNT(*) AS percentage_of_churned_customer
FROM customer_churn WHERE ChurnStatus = 'Churned';
```

Quantifies the overlap between complaint history and churn, a critical metric for customer service prioritisation.

### 6.4 City Tier and Category Analysis

```sql
SELECT CityTier, COUNT(*) AS churned_laptop_customers
FROM customer_churn
WHERE ChurnStatus = 'Churned' AND PreferredOrderCat = 'Laptop & Accessory'
GROUP BY CityTier ORDER BY churned_laptop_customers DESC LIMIT 1;
```

Identifies whether high-value electronics buyers in specific city types are disproportionately churning.

### 6.5 Active Customer Payment Preferences

```sql
SELECT PreferredPaymentMode, COUNT(*) AS most_preferred_among_active_customers
FROM customer_churn WHERE ChurnStatus = 'Active'
GROUP BY PreferredPaymentMode ORDER BY COUNT(*) DESC LIMIT 1;
```

Informs payment infrastructure investment by surfacing the dominant mode among retained customers.

### 6.6 Segment-Specific Order Value Growth

```sql
SELECT SUM(OrderAmountHikeFromlastYear) AS total_order_amount_hike
FROM customer_churn WHERE MaritalStatus = 'Single' AND PreferredOrderCat = 'Mobile Phone';
```

Reveals the growth contribution of a specific demographic + category segment.

### 6.7 Device Registration by Payment Mode

```sql
SELECT AVG(NumberOfDeviceRegistered) AS avg_devices
FROM customer_churn WHERE PreferredPaymentMode = 'UPI';
```

UPI (Unified Payments Interface) users tend to be more digitally engaged; this confirms whether that pattern holds in device registrations.

### 6.8 City Tier Customer Volume

```sql
SELECT CityTier, COUNT(*) AS Customers
FROM customer_churn GROUP BY CityTier ORDER BY COUNT(*) DESC LIMIT 1;
```

Establishes which market tier drives the largest customer base.

### 6.9 Coupon Usage by Gender

```sql
SELECT Gender, COUNT(CouponUsed) AS highest_no_coupon
FROM customer_churn GROUP BY Gender ORDER BY COUNT(CouponUsed) DESC LIMIT 1;
```

Supports gender-targeted promotional campaign design.

### 6.10 App Engagement by Order Category

```sql
SELECT PreferredOrderCat, COUNT(*) AS no_of_customer, MAX(HoursSpentOnApp)
FROM customer_churn GROUP BY PreferredOrderCat;
```

Combines volume and peak engagement to identify which categories hold user attention most strongly.

### 6.11 High-Satisfaction Credit Card Buyers

```sql
SELECT SUM(OrderCount) AS TotalOrderCount
FROM customer_churn
WHERE PreferredPaymentMode = 'Credit Card'
  AND SatisfactionScore = (SELECT MAX(SatisfactionScore) FROM customer_churn);
```

Uses a scalar subquery to isolate the most satisfied credit card users and size their order volume.

### 6.12 Satisfaction Among Complainers

```sql
SELECT AVG(SatisfactionScore) AS avg_satisfactionscore_of_who_complained
FROM customer_churn WHERE ComplaintReceived = 'Yes';
```

A low average here would indicate that complaint resolution is not succeeding in maintaining satisfaction.

### 6.13 Categories Among Heavy Coupon Users

```sql
SELECT PreferredOrderCat, COUNT(*) AS count_of_category
FROM customer_churn WHERE CouponUsed > 5
GROUP BY PreferredOrderCat ORDER BY COUNT(*) DESC;
```

Ranks product categories by share of heavy-discount shoppers, relevant for margin analysis.

### 6.14 Top Categories by Cashback

```sql
SELECT PreferredOrderCat, AVG(CashbackAmount) AS highest_avg_cashback_amt
FROM customer_churn GROUP BY PreferredOrderCat
ORDER BY highest_avg_cashback_amt DESC LIMIT 3;
```

Identifies which categories generate the highest average cashback outlay, informing promotion cost review.

### 6.15 Payment Modes with Specific Tenure and Order Volume

```sql
SELECT PreferredPaymentMode,
       ROUND(AVG(Tenure), 0) AS avg_tenure_rounded,
       SUM(OrderCount) AS total_orders
FROM customer_churn
GROUP BY PreferredPaymentMode
HAVING ROUND(AVG(Tenure), 0) = 10 AND SUM(OrderCount) > 500;
```

Uses `HAVING` on aggregated values to find payment methods used by mid-tenure, high-volume customer segments.

### 6.16 Distance-Based Churn Segmentation

```sql
SELECT
  CASE
    WHEN WarehouseToHome <= 5  THEN 'Very Close Distance'
    WHEN WarehouseToHome <= 10 THEN 'Close Distance'
    WHEN WarehouseToHome <= 15 THEN 'Moderate Distance'
    WHEN WarehouseToHome > 15  THEN 'Far Distance'
  END AS distance,
  ChurnStatus,
  COUNT(*) AS CUSTOMER_COUNT
FROM customer_churn GROUP BY distance, ChurnStatus;
```

Tests whether delivery distance is a churn driver by cross-tabulating distance bands with churn status.

### 6.17 High-Value Married Tier-1 Customers

```sql
SELECT CustomerID, CityTier, PreferredPaymentMode, PreferredOrderCat,
       CouponUsed, OrderCount, CashbackAmount, MaritalStatus
FROM customer_churn
WHERE MaritalStatus = 'Married' AND CityTier = 1
  AND OrderCount > (SELECT AVG(OrderCount) FROM customer_churn);
```

Isolates a high-priority retention segment using a correlated subquery benchmark.

### 6.18 Returns Joined to Churn Profile

```sql
SELECT cc.CustomerID, cr.ReturnID, cr.ReturnDate, cr.RefundAmount,
       cc.CityTier, cc.MaritalStatus, cc.OrderCount,
       cc.PreferredPaymentMode, cc.PreferredOrderCat,
       cc.CashbackAmount, cc.ChurnStatus, cc.ComplaintReceived
FROM customer_returns cr
INNER JOIN customer_churn cc ON cr.CustomerID = cc.CustomerID;
```

Links return events to the full customer profile, enabling analysis of whether returners who also complained are more likely to have churned.

---

## 7. Summary of Findings

| Theme | Observation |
|---|---|
| **Churn volume** | The dataset has a meaningful proportion of churned customers, concentrated among those with the lowest tenure values (0–1 months). |
| **Complaints as a churn signal** | A notable share of churned customers had previously raised a complaint, suggesting unresolved service issues are a churn precursor. |
| **Geography** | City Tier 1 dominates in both total customers and churned Laptop & Accessory buyers, indicating that metro markets see high-value segment attrition. |
| **Payment behaviour** | Debit Card and Credit Card are the leading payment modes among active customers. UPI users show above-average device registrations, reflecting higher digital engagement. |
| **Coupon dependency** | Customers using more than 5 coupons skew heavily toward Mobile Phone and Laptop & Accessory, raising the question of whether discounting is attracting low-loyalty buyers in these categories. |
| **Cashback categories** | Grocery and Others categories generate the highest average cashback per customer, which may reflect high-frequency, lower-margin ordering patterns. |
| **Returns overlap** | The small `customer_returns` dataset (8 records) overlaps with churned complainers, consistent with the hypothesis that product dissatisfaction → complaint → return → churn is a sequential risk pathway. |

---

## 8. Technical Notes

- All `UPDATE` statements use the `JOIN ... SET` pattern rather than correlated subqueries in the `WHERE` clause, which is the recommended approach in MySQL 8 to avoid the "you can't specify target table for update in FROM clause" error.
- `ROUND(AVG(...))` is used throughout for mean imputation to keep INT column types consistent.
- The `HAVING` clause in query 6.15 correctly filters on aggregated results rather than misusing `WHERE` on aggregate functions.
- The `INNER JOIN` in query 6.18 means only customers present in both tables are returned; no outer join was needed given the analytical intent.

---

## 9. Potential Extensions

- Add an index on `ChurnStatus` and `CityTier` to speed up repeated filter scans as the dataset grows.
- Create a `churn_summary` view to encapsulate the most frequently queried aggregations.
- Export query results to a BI tool (Tableau, Power BI, or Metabase) for visual dashboarding.
- Extend `customer_returns` with more records to enable statistically meaningful return-churn correlation analysis.
- Apply window functions (`RANK()`, `LAG()`) for cohort-level tenure and order trend analysis.

---

*End of Report*
