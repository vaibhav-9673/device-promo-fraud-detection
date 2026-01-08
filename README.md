# device-promo-abuse-detection
SQL, Power BI and Python based fraud detection project to identify promo abuse using user behavior analysis.

## 🔴 Problem Statement 
Online platforms that offer new-user promotions (such as discounts, cashback, or free deliveries) often face significant revenue leakage due to promo abuse. A common abuse pattern occurs when users create multiple accounts to repeatedly claim benefits intended for first-time users.

#### Traditional fraud-prevention checks typically rely on phone numbers, email IDs, to identify unique users. However, these methods are easily bypassed because:
<br> 1. Phone numbers can be changed or obtained easily.
<br> 2. Email addresses can be created in seconds
<br> 3. Users can log out and re-register with minimal effort

As a result, the same individual can appear as multiple “new users” in the system.

#### This creates a challenge for platforms because:
<br> 1. Promo costs increase without genuine customer acquisition
<br> 2. Marketing metrics become inaccurate
<br> 3. Legitimate users are indirectly affected due to stricter promo policies

Therefore, the key challenge is to identify and link accounts that appear different at the user level but originate from similar behavioral or device-level patterns, enabling the platform to detect and prevent promo abuse at scale.

## 🎯 Objective

The objective of this project is to identify and analyze promo abuse and fraudulent user behavior on an online platform where users create multiple accounts to misuse new-user promotional offers.
<br> 1.Detect abnormal promo usage patterns
<br> 2.Identify users with unusually fast signup-to-order behavior
<br> 3.Highlight high-risk user segments using percentage-based metrics
<br> 4.Provide actionable insights for fraud prevention without relying on heavy machine learning models
<br> 5.The solution focuses on rule-based and behavioral analysis, making it transparent, explainable, and scalable for real-world fraud monitoring systems.

1. Database Setup
Database Creation

The project begins by creating a relational database to store user signup and order behavior data, which is required to analyze promotional abuse patterns.

Table Creation

The database contains the following core tables:

users
Stores user-level information such as:

user_id

signup_date

city

orders
Stores transactional data related to user orders:

order_id

user_id

order_date

order_amount

promo_used (boolean flag indicating promo usage)

These tables represent the minimum realistic schema required to analyze new-user promo abuse at scale.

#### SQL Schema Definition
```
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    signup_date DATETIME,
    city VARCHAR(50)
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
2. Data Exploration & Cleaning

Before performing fraud analysis, the data was validated to ensure accuracy and consistency.

🔍 Data Validation Steps

Record Count Validation

Verified total number of records in users and orders

Null Value Checks

Checked for missing values in critical columns such as user_id, signup_date, order_date, and promo_used

Referential Integrity Check

Ensured all orders map to valid users
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

