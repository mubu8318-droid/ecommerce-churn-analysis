# 🛒 E-Commerce Customer Churn Analysis — MySQL

A complete end-to-end MySQL project that models, cleans, transforms, and analyzes customer churn data for an e-commerce platform. The project covers database design, data quality engineering, and exploratory SQL analysis across ~5,630 customer records.

---

## 📁 Repository Structure

```
ecommerce-churn-analysis/
│
├── MODULE_END_ASSIGNMENT_MYSQL.sql   # Full SQL script (schema → data → analysis)
├── README.md                         # Project overview and usage guide
└── REPORT.md                         # Detailed technical report with findings
```

---

## 🗄️ Database Overview

| Item | Detail |
|---|---|
| **Database** | `ecomm` |
| **Primary Table** | `customer_churn` |
| **Secondary Table** | `customer_returns` |
| **Total Records** | ~5,630 customers |
| **Customer ID Range** | 50001 – 55630 |

---

## 🔄 Project Workflow

```
1. Schema Design & Table Creation
        ↓
2. Raw Data Insertion (~5,630 rows)
        ↓
3. Data Cleaning (nulls, outliers, inconsistencies)
        ↓
4. Data Transformation (renaming, derived columns)
        ↓
5. Exploratory Data Analysis (15+ queries)
        ↓
6. Multi-table JOIN Analysis (customer_returns)
```

---

## 🧱 Schema

### `customer_churn` (Primary Table)

| Column | Type | Description |
|---|---|---|
| `CustomerID` | INT (PK) | Unique customer identifier |
| `Churn` | BIT | Raw churn flag (1 = churned) |
| `Tenure` | INT | Months as a customer |
| `PreferredLoginDevice` | VARCHAR(20) | Device used to log in |
| `CityTier` | INT | City classification (1, 2, or 3) |
| `WarehouseToHome` | INT | Distance from warehouse to home (km) |
| `PreferredPaymentMode` | VARCHAR(20) | Payment method |
| `Gender` | ENUM | Male / Female |
| `HourSpendOnApp` | INT | Hours spent on the app per month |
| `NumberOfDeviceRegistered` | INT | Devices linked to account |
| `PreferedOrderCat` | VARCHAR(20) | Most ordered product category |
| `SatisfactionScore` | INT | Customer satisfaction (1–5) |
| `MaritalStatus` | VARCHAR(10) | Single / Married / Divorced |
| `NumberOfAddress` | INT | Saved delivery addresses |
| `Complain` | BIT | Raw complaint flag (1 = complaint raised) |
| `OrderAmountHikeFromlastYear` | INT | % order value increase YoY |
| `CouponUsed` | INT | Coupons used in last month |
| `OrderCount` | INT | Orders placed in last month |
| `DaySinceLastOrder` | INT | Days since most recent order |
| `CashbackAmount` | INT | Cashback received (currency units) |

> **Post-transformation additions:** `ChurnStatus` (VARCHAR), `ComplaintReceived` (VARCHAR). Original `Churn` and `Complain` columns are dropped.

### `customer_returns` (Secondary Table)

| Column | Type | Description |
|---|---|---|
| `ReturnID` | INT (PK) | Unique return identifier |
| `CustomerID` | INT (FK) | Links to `customer_churn` |
| `ReturnDate` | DATE | Date of return |
| `RefundAmount` | DECIMAL(10,2) | Refund issued |

---

## 🧹 Data Cleaning Steps

**Missing Value Imputation**

| Column | Strategy | Reason |
|---|---|---|
| `WarehouseToHome` | Mean | Continuous, roughly normal |
| `HourSpendOnApp` | Mean | Continuous, roughly normal |
| `OrderAmountHikeFromlastYear` | Mean | Continuous, roughly normal |
| `DaySinceLastOrder` | Mean | Continuous, roughly normal |
| `Tenure` | Mode | Skewed / discrete |
| `CouponUsed` | Mode | Skewed / discrete |
| `OrderCount` | Mode | Skewed / discrete |

**Outlier Removal**
- Rows where `WarehouseToHome > 100` are deleted as physically implausible.

**Inconsistency Fixes**

| Column | Raw Value | Standardised To |
|---|---|---|
| `PreferredLoginDevice` | `'Phone'` | `'Mobile Phone'` |
| `PreferredOrderCat` | `'Mobile'` | `'Mobile Phone'` |
| `PreferredPaymentMode` | `'COD'` | `'Cash on Delivery'` |
| `PreferredPaymentMode` | `'CC'` | `'Credit Card'` |

---

## 🔁 Data Transformation

| Action | Detail |
|---|---|
| Column rename | `PreferedOrderCat` → `PreferredOrderCat` |
| Column rename | `HourSpendOnApp` → `HoursSpentOnApp` |
| New derived column | `ComplaintReceived` — `'Yes'` / `'No'` from `Complain` |
| New derived column | `ChurnStatus` — `'Churned'` / `'Active'` from `Churn` |
| Column drop | `Churn` and `Complain` removed after derivation |

---

## 📊 Analysis Queries

The script includes 15+ analytical queries covering:

- Count of churned vs active customers
- Average tenure and total cashback for churned customers
- Percentage of churned customers who lodged a complaint
- City tier with the most churned Laptop & Accessory buyers
- Most preferred payment mode among active customers
- Total order amount hike for single customers ordering mobile phones
- Average devices registered per UPI user
- City tier with the highest customer count
- Gender with the highest coupon usage
- Customer count and peak app hours per order category
- Total order count for credit card users with maximum satisfaction score
- Average satisfaction score among customers who complained
- Most popular order categories for heavy coupon users (> 5 used)
- Top 3 order categories by average cashback amount
- Payment modes with average tenure ≈ 10 months and > 500 combined orders
- Warehouse-to-home distance segmentation with churn breakdown
- High-value married Tier-1 customers with above-average order count
- Return details joined to churn + complaint profile

---

## ⚙️ How to Run

### Prerequisites
- MySQL Server 8.0+ (or MariaDB 10.5+)
- MySQL Workbench, DBeaver, or any compatible SQL client

### Steps

```sql
-- 1. Open MODULE_END_ASSIGNMENT_MYSQL.sql in your SQL client

-- 2. The script self-contained — run it top to bottom:
--    It creates the database, tables, inserts data,
--    performs all cleaning/transformation, then runs analyses.

-- 3. Verify setup:
USE ecomm;
SELECT COUNT(*) FROM customer_churn;   -- ~5,630 rows
SELECT COUNT(*) FROM customer_returns; -- 8 rows
```

> **Note:** Run the full script sequentially. Some statements (ALTER TABLE, UPDATE) depend on earlier steps completing first.

---

## 🔑 Key Takeaways

- Customers with **short tenure (0–1 months)** show the highest churn rates.
- **City Tier 1** has the largest customer base and the most churned Laptop & Accessory buyers.
- A significant share of churned customers had previously raised a complaint, suggesting complaint resolution is a churn driver.
- **Debit Card** and **Credit Card** are the dominant payment modes among active customers.
- Customers who used more than 5 coupons skew toward **Mobile Phone** and **Laptop & Accessory** categories.

---

## 🛠️ Tech Stack

![MySQL](https://img.shields.io/badge/MySQL-8.0-blue?logo=mysql&logoColor=white)

---

## 📄 License

This project is submitted as an academic module-end assignment. Feel free to reference or adapt the SQL patterns for learning purposes.
