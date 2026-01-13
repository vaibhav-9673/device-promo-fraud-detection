# device-promo-abuse-detection
SQL, Power BI and Python based fraud detection project to identify promo abuse using user behavior analysis.

**Project Structure**
```
device-level-promo-abuse-detection/
│
├── data/
│   ├── users_1000_rows.csv
│   ├── orders_1000_rows.csv
│
├── sql/
│   ├── 01_table_creation.sql
│   ├── 02_data_loading.sql
│   ├── 03_data_validation.sql
│   ├── 04_fraud_analysis.sql
│   ├── 05_analytical_view.sql
│
├── powerbi/
│   └── fraud_dashboard.pbix
│
├── python/
│   └── fraud_analysis.py
│
├── outputs/
│   └── fraud_user_summary_python.csv
│
└── README.md
```

## 🔴 Problem Statement 
Online platforms that offer new-user promotions (such as discounts, cashback, or free deliveries) often face significant revenue leakage due to promo abuse. A common abuse pattern occurs when users create multiple accounts to repeatedly claim benefits intended for first-time users.

**Traditional fraud-prevention checks typically rely on phone numbers, email IDs, to identify unique users. However, these methods are easily bypassed because:**
<br> 1. Phone numbers can be changed or obtained easily.
<br> 2. Email addresses can be created in seconds
<br> 3. Users can log out and re-register with minimal effort

As a result, the same individual can appear as multiple “new users” in the system.

#### This creates a challenge for platforms because:
<br> 1. Promo costs increase without genuine customer acquisition
<br> 2. Marketing metrics become inaccurate
<br> 3. Legitimate users are indirectly affected due to stricter promo policies 
    <br>Therefore, the key challenge is to identify and link accounts that appear different at the user level but originate from similar behavioral or device-level patterns, enabling the platform to detect and prevent promo abuse at scale.


---
## 🎯 Objective

The objective of this project is to identify and analyze promo abuse and fraudulent user behavior on an online platform where users create multiple accounts to misuse new-user promotional offers.
<br> 1.Detect abnormal promo usage patterns
<br> 2.Identify users with unusually fast signup-to-order behavior
<br> 3.Highlight high-risk user segments using percentage-based metrics
<br> 4.Provide actionable insights for fraud prevention without relying on heavy machine learning models
<br> 5.The solution focuses on rule-based and behavioral analysis, making it transparent, explainable, and scalable for real-world fraud monitoring systems.

----

### 1. Database Setup
**Database Creation**
The project begins by creating a relational database to store user signup and order behavior data, which is required to analyze promotional abuse patterns.

**Table Creation**
The database contains the following core tables:

**Table 01: users**
<br>Stores user-level information such as:
1. user_id
2. signup_date
3. city

**Table 02: devices**
<br>Stores device-level information used to identify multiple accounts originating from the same or similar devices, which is a key indicator of promo abuse.
1. device_id
2. user_id
3. device_model
4. os_version
5. ip_address

**Table 03: orders**
<br>Stores transactional data related to user orders:
1. order_id
2. user_id
3. order_date
4. order_amount
5. promo_used (boolean flag indicating promo usage)

#### SQL Schema Definition
```
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    signup_date DATETIME,
    city VARCHAR(50)
);

CREATE TABLE devices (
    device_id VARCHAR(50),
    user_id INT,
    device_model VARCHAR(50),
    os_version VARCHAR(20),
    ip_address VARCHAR(50),
    PRIMARY KEY (device_id, user_id)
);

CREATE TABLE orders (
    order_id INT PRIMARY KEY,
    user_id INT,
    order_date DATETIME,
    order_amount DECIMAL(10,2),
    promo_used BOOLEAN,
    FOREIGN KEY (user_id) REFERENCES users(user_id)
);
```
### 2. Data Exploration & Cleaning
<br> (Before performing fraud analysis, the data was validated to ensure accuracy and consistency.)

**🔍 Data Validation Steps**
1. Record Count Validation
2. Verified total number of records in users and orders
3. Null Value Checks
4. Checked for missing values in critical columns such as user_id, signup_date, order_date, and promo_used
5. Referential Integrity Check
6. Ensured all orders map to valid users

```
-- Check for NULL values in users
SELECT * FROM users
WHERE user_id IS NULL
   OR signup_date IS NULL
   OR city IS NULL;

-- Check for NULL values in orders

SELECT * FROM orders
WHERE order_id IS NULL
   OR user_id IS NULL
   OR order_date IS NULL
   OR promo_used IS NULL;
```

### 📊 Data Analysis & Findings
<br>This section presents key insights derived from analyzing user behavior, device usage, and order patterns to identify potential new-user promo abuse.

#### 1. Device-Level Analysis
**A) Which devices are linked to multiple users?**
```
SELECT 
    device_id,
    COUNT(DISTINCT user_id) AS total_users
FROM devices
GROUP BY device_id
HAVING total_users > 1;
```
**Finding:**
<br>Several devices were associated with multiple user accounts, indicating a high likelihood of users creating multiple accounts on the same device to exploit new-user promotions.

**2) High-risk devices based on number of linked users**
```
SELECT 
    device_id,
    COUNT(DISTINCT user_id) AS user_count
FROM devices
GROUP BY device_id
HAVING user_count >= 3;
```
**Finding:**
<br>Devices linked to three or more users were classified as high-risk devices, as such patterns rarely occur in genuine usage scenarios.

#### 3. Promo Usage Analysis
**A) What percentage of orders used promo codes?**
```
SELECT 
    ROUND(SUM(promo_used)/COUNT(*)*100, 2) AS promo_usage_percentage
FROM orders;
```
**Finding:**
<br>A significant percentage of orders were placed using promotional offers, suggesting potential overuse and abuse of new-user promotions.

**B) Which users repeatedly used promo codes?**
```
SELECT 
    user_id,
    COUNT(*) AS total_orders,
    SUM(promo_used) AS promo_orders
FROM orders
GROUP BY user_id
HAVING promo_orders >= 2;
```
**Finding:**
<br>Multiple users repeatedly applied promo codes across different orders, which is inconsistent with typical one-time new-user promo behavior.

#### 3️⃣ Signup-to-First-Order Behavior
**A) How quickly do users place their first order after signup?**
```
SELECT 
    u.user_id,
    TIMESTAMPDIFF(
        MINUTE,
        u.signup_date,
        MIN(o.order_date)
    ) AS minutes_to_first_order
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id;
```
**Finding:**
<br>A subset of users placed their first order within minutes of account creation, a common characteristic of fraudulent accounts aiming to quickly redeem promotional benefits.

#### 4️⃣ Combined Fraud Indicators
**A) Identification of high-risk users**
1. Users were flagged as high-risk if they satisfied all of the following conditions:
2. Promo usage percentage > 70%
3. Signup-to-first-order time < 15 minutes
4. More than one order placed
```
SELECT 
    u.user_id,
    COUNT(o.order_id) AS total_orders,
    ROUND(SUM(o.promo_used)/COUNT(o.order_id)*100,2) AS promo_pct,
    TIMESTAMPDIFF(
        MINUTE,
        u.signup_date,
        MIN(o.order_date)
    ) AS minutes_to_first_order
FROM users u
JOIN orders o ON u.user_id = o.user_id
GROUP BY u.user_id
HAVING promo_pct > 70
   AND minutes_to_first_order < 15
   AND total_orders >= 2;
```
**Finding:**
<br>Combining device sharing, promo usage, and behavioral timing significantly improves fraud detection accuracy compared to using a single indicator.

#### 5️⃣ Device + User Behavior Correlation
**A) Promo abuse concentration on shared devices**
```
SELECT 
    d.device_id,
    COUNT(o.order_id) AS total_orders,
    SUM(o.promo_used) AS promo_orders,
    ROUND(SUM(o.promo_used)/COUNT(o.order_id)*100,2) AS promo_pct
FROM devices d
JOIN orders o ON d.user_id = o.user_id
GROUP BY d.device_id
HAVING promo_pct > 60;
```
**Finding:**
<br>Devices shared across multiple users also showed higher promo usage percentages, reinforcing the hypothesis of coordinated promo abuse.

#### 6️⃣ Analytical Summary View
<br>To support dashboarding and further analysis, a consolidated analytical view was created combining all fraud indicators.
```
CREATE VIEW fraud_user_summary AS
SELECT 
    u.user_id,
    u.city,
    COUNT(o.order_id) AS total_orders,
    SUM(o.promo_used) AS promo_orders,
    ROUND(SUM(o.promo_used)/COUNT(o.order_id)*100,2) AS promo_pct,
    TIMESTAMPDIFF(
        MINUTE,
        u.signup_date,
        MIN(o.order_date)
    ) AS minutes_to_first_order,
    COUNT(DISTINCT d.device_id) AS device_count
FROM users u
JOIN orders o ON u.user_id = o.user_id
LEFT JOIN devices d ON u.user_id = d.user_id
GROUP BY u.user_id, u.city;
```

#### 🔍 Key Insights Summary
1. Multiple accounts linked to the same device are a strong indicator of promo abuse
2. Fraudulent users tend to place orders immediately after signup
3. Promo abuse is highly concentrated among users sharing devices
4. Combining multiple behavioral signals yields more reliable fraud detection

### ✅ Conclusion
This project demonstrates how device-level analysis combined with user and order behavior can effectively identify new-user promo abuse using data analytics techniques. By linking multiple user accounts through shared device information and analyzing promo usage patterns, the project highlights how fraudulent behavior can be detected without relying on complex machine learning models.

**Key outcomes of the analysis include:**
1. Identification of high-risk devices associated with multiple user accounts
2. Detection of users with abnormally high promo usage and rapid signup-to-order behavior
3. Use of rule-based and percentage-driven metrics to ensure explainable and unbiased fraud detection
4. Creation of analysis-ready datasets that support dashboarding, reporting, and future model development
    <br>Overall, this project showcases an end-to-end fraud analytics workflow using SQL, Power BI, and Python, demonstrating how structured data analysis can help organizations reduce promo misuse, protect revenue, and support data-driven decision-making. The solution is scalable and can be further enhanced by integrating AI/ML models for automated, real-time fraud prevention.

### Author: Vaibhav Gade
This project is part of my data analytics portfolio and demonstrates practical skills in **MySQL, Power BI,** and **Python** applied to fraud analysis and business-oriented data insights, relevant to Data Analyst and Business Analyst roles.

