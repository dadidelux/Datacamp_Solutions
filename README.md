# FoodYum Grocery Store Sales – SQL Practical Exam

This project contains **SQL solutions** for practical data cleaning and analysis tasks based on the `products` table from FoodYum’s grocery chain data.

---

## Task 1: **Count Missing Year Added**

**Question:**
Last year (2022), some products were added without setting the `year_added` value.
Write a query to determine how many products have the `year_added` value missing.
Output a single column `missing_year`, with the number of missing values.

**SQL Solution:**

```sql
SELECT COUNT(*) AS missing_year
FROM products
WHERE year_added IS NULL;
```

---

## Task 2: **Clean Categorical and Text Data**

**Question:**
Clean the `products` table so that it matches the criteria below. Do not update the original table—just output the cleaned data.

| Column               | Cleaning Rule                                                                                                                  |
| -------------------- | ------------------------------------------------------------------------------------------------------------------------------ |
| product\_id          | Unique identifier. Never missing.                                                                                              |
| product\_type        | If missing or empty, replace with `'Unknown'`. Trim and title case.                                                            |
| brand                | If missing or equals `'-'`, replace with `'Unknown'`. Trim and title case.                                                     |
| weight               | Extract numeric value (e.g. `'602.61 grams'` → `602.61`). If missing/invalid, replace with median weight. Round to 2 decimals. |
| price                | If missing, replace with median price. Round to 2 decimals.                                                                    |
| average\_units\_sold | If missing, replace with `0`.                                                                                                  |
| year\_added          | If missing, replace with `2022`.                                                                                               |
| stock\_location      | If missing or empty, replace with `'Unknown'`. Trim and uppercase.                                                             |

**SQL Solution:**

```sql
WITH ranked_weights AS (
    SELECT
        weight,
        ROW_NUMBER() OVER (ORDER BY weight::NUMERIC) AS rn,
        COUNT(*) OVER() AS cnt
    FROM products
    WHERE
        weight ~ E'^\\d+\\.?\\d*\\s*grams$'
        OR weight ~ E'^\\d+\\.?\\d*$'
),
median_weight AS (
    SELECT ROUND(AVG(weight::NUMERIC)::NUMERIC, 2) AS median_value
    FROM (
        SELECT weight
        FROM ranked_weights
        WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)
    ) AS sub
),
ranked_prices AS (
    SELECT
        price,
        ROW_NUMBER() OVER (ORDER BY price::NUMERIC) AS rn,
        COUNT(*) OVER() AS cnt
    FROM products
    WHERE price IS NOT NULL
),
median_price AS (
    SELECT ROUND(AVG(price::NUMERIC)::NUMERIC, 2) AS median_value
    FROM (
        SELECT price
        FROM ranked_prices
        WHERE rn IN ((cnt + 1) / 2, (cnt + 2) / 2)
    ) AS sub
)
SELECT
    product_id,
    COALESCE(NULLIF(INITCAP(TRIM(product_type)), ''), 'Unknown') AS product_type,
    COALESCE(NULLIF(INITCAP(TRIM(brand)), '-'), 'Unknown') AS brand,
    ROUND(
        COALESCE(
            CASE
                WHEN weight ~ E'^\\d+\\.?\\d*\\s*grams$'
                    THEN CAST(REPLACE(weight, ' grams', '') AS NUMERIC)
                WHEN weight ~ E'^\\d+\\.?\\d*$'
                    THEN CAST(weight AS NUMERIC)
            END,
            (SELECT median_value FROM median_weight LIMIT 1)
        )::NUMERIC, 2
    ) AS weight,
    ROUND(
        COALESCE(price::NUMERIC, (SELECT median_value FROM median_price LIMIT 1))::NUMERIC, 2
    ) AS price,
    COALESCE(average_units_sold::INTEGER, 0) AS average_units_sold,
    COALESCE(year_added::INTEGER, 2022) AS year_added,
    COALESCE(NULLIF(UPPER(TRIM(stock_location)), ''), 'Unknown') AS stock_location
FROM products;
```

---

## Task 3: **Min and Max Price Per Product Type**

**Question:**
Determine the minimum and maximum price for each product type, treating missing `product_type` as `'Unknown'`.

**SQL Solution:**

```sql
SELECT
    COALESCE(NULLIF(product_type, ''), 'Unknown') AS product_type,
    ROUND(MIN(price)::numeric, 2) AS min_price,
    ROUND(MAX(price)::numeric, 2) AS max_price
FROM products
GROUP BY COALESCE(NULLIF(product_type, ''), 'Unknown');
```

---

## Task 4: **Detailed Query for Meat and Dairy with High Sales**

**Question:**
Return the `product_id`, `price`, and `average_units_sold` for all **Meat** and **Dairy** products where `average_units_sold` is greater than 10.

**SQL Solution:**

```sql
SELECT
    product_id,
    price,
    average_units_sold
FROM products
WHERE
    (product_type = 'Meat' OR product_type = 'Dairy')
    AND average_units_sold > 10;
```

---

## Task 5: **(If Applicable: String Manipulation Only)**

**Question:**
Demonstrate string cleaning:

* Remove whitespace from `product_type`, `brand`, `stock_location`.
* Title-case `product_type`, `brand`; uppercase `stock_location`.

**SQL Solution:**

```sql
SELECT
    product_id,
    COALESCE(NULLIF(INITCAP(TRIM(product_type)), ''), 'Unknown') AS product_type,
    COALESCE(NULLIF(INITCAP(TRIM(brand)), '-'), 'Unknown') AS brand,
    COALESCE(NULLIF(UPPER(TRIM(stock_location)), ''), 'Unknown') AS stock_location
FROM products;
```

---

# Notes

* **All queries are written for PostgreSQL.**
* Be sure to test with your actual schema/data and adapt for other SQL dialects as needed.
* Cleaning steps make use of regex, window functions, and built-in string functions to robustly prepare data for analysis.
