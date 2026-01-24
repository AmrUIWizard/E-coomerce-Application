# E-Commerce Database System

This project documents the database design for a relational e-commerce system.
It includes the schema definition, entity relationships, analytical SQL queries,
and utility functions for generating and managing test data.

---

## Table of Contents
- [1. Project Overview](#1-project-overview)
- [2. Database Schema](#2-database-schema)
- [3. Entity Relationships](#3-entity-relationships)
- [4. ERD Diagram](#4-erd-diagram)
- [5 Analytical SQL Queries](#1-analytical-sql-queries)
	- [5.1 Daily Revenue Report](#51-daily-revenue-report)
	- [5.2 Top-Selling Products in a Month](#52-top-selling-products-in-a-month)
	- [5.3 Customers Who Spent More Than 500 in a Month](#53-customers-who-spent-more-than-500-in-a-month)
	- [5.4 Search for Products Containing the Word camera](#54-search-for-products-containing-the-word-camera)
	- [5.5 Suggest Popular Products From the Same Category](#55-suggest-popular-products-from-the-same-category)
- [6 Data Generation](#6-data-generation)
	- [6.1 Customer Data Generation Function](#61-customer-data-generation-function)
	- [6.2 Categories Data Generation Function](#62-categories-data-generation-function)
	- [6.3 Products Data Generation Function](#63-products-data-generation-function)
	- [6.4 Orders and Order Details Generation Function](#64-orders-and-order-details-generation-function)




---

## 1 Project Overview

This database is designed to support a basic e-commerce platform.
It handles customer management, product cataloging, order processing,
and analytical reporting.

The schema follows relational database best practices while allowing
selective denormalization for reporting performance.

---

## 2 Database Schema

This section defines the core tables used to store and manage
e-commerce data. The schema is designed for data integrity,
scalability, and analytical querying.

```sql
CREATE TABLE customer (
customer_id SERIAL PRIMARY KEY,
email VARCHAR(100) NOT NULL UNIQUE,
password TEXT NOT NULL,
first_name VARCHAR(100) NOT NULL,
last_name VARCHAR(100) NOT NULL
);

CREATE TABLE category (
category_id SERIAL PRIMARY KEY,
category_name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE product (
product_id SERIAL PRIMARY KEY,
category_id INTEGER NOT NULL REFERENCES category(category_id) ON DELETE CASCADE,
name VARCHAR(100) NOT NULL,
description TEXT,
price NUMERIC(10, 2) NOT NULL,
stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0)
);

CREATE TABLE orders (
order_id SERIAL PRIMARY KEY,
customer_id INTEGER NOT NULL REFERENCES customer(customer_id) ON DELETE CASCADE,
order_date TIMESTAMP NOT NULL DEFAULT NOW(),
total_amount NUMERIC(10, 2) NOT NULL CHECK (total_amount >= 0),
customer_name VARCHAR(200)
);

CREATE TABLE order_details (
order_detail_id SERIAL PRIMARY KEY,
order_id INTEGER NOT NULL REFERENCES orders(order_id) ON DELETE CASCADE,
product_id INTEGER NOT NULL REFERENCES product(product_id) ON DELETE CASCADE,
unit_price NUMERIC(10, 2) NOT NULL,
quantity INTEGER NOT NULL DEFAULT 1 CHECK (quantity >= 1)
);
```

---

## 3 Entity Relationships

The following table describes how entities are connected and
how data flows between them.

| Parent   | Child         | Relationship | Foreign Key              |
| -------- | ------------- | ------------ | ------------------------ |
| customer | order         | 1 : M        | order.customer_id        |
| category | product       | 1 : M        | product.category_id      |
| order    | order_details | 1 : M        | order_details.order_id   |
| product  | order_details | 1 : M        | order_details.product_id |

---

## 4 ERD Diagram

The Entity Relationship Diagram (ERD) below visually represents
the database structure and the relationships between entities.

![ERD Diagram](erd_diagram.png)

---

## 5 Analytical SQL Queries

This section provides example SQL queries used for reporting
and business insights, such as revenue analysis, customer value,
and product performance.

### 5.1 Daily Revenue Report

This query calculates the total revenue generated on a specific date.
It is useful for daily sales tracking and financial reporting.

```sql
SELECT
    DATE(order_date) AS report_date,
    SUM(total_amount) AS total_revenue
FROM orders
WHERE DATE(order_date) = DATE '2025-01-18'
GROUP BY DATE(order_date);
```

### 5.2 Top-Selling Products in a Month

This query identifies the top-selling products for a given month based on total revenue.
It helps evaluate product performance and supports inventory and marketing decisions.

```sql
SELECT
    p.product_id,
    p.name AS product_name,
    SUM(od.unit_price * od.quantity) AS total_revenue
FROM order_details od
JOIN orders o ON od.order_id = o.order_id
JOIN products p ON od.product_id = p.product_id
WHERE DATE_TRUNC('month', o.order_date) = DATE_TRUNC('month', DATE '2025-01-01')
GROUP BY p.product_id, p.name
ORDER BY total_revenue DESC
LIMIT 3;
```

### 5.3 Customers Who Spent More Than 500 in a Month

This query finds high-value customers whose total spending exceeds a defined threshold within a specific month.
It can be used for customer segmentation, loyalty programs, or targeted promotions.

```sql
SELECT
    c.customer_id,
    CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON o.customer_id = c.customer_id
WHERE DATE_TRUNC('month', o.order_date) = DATE_TRUNC('month', DATE '2025-01-01')
GROUP BY c.customer_id, customer_name
HAVING SUM(o.total_amount) > 500
ORDER BY total_spent DESC;
```

### 5.4 Search for Products Containing the Word camera

This query searches for products whose name or description contains a specific keyword.
It supports product search functionality commonly used in e-commerce platforms.

```sql
SELECT *
FROM products
WHERE name ILIKE '%camera%'
   OR description ILIKE '%camera%';
```

### 5.5 Suggest Popular Products From the Same Category

This query suggests popular products from the same category while excluding a selected product.
It can be used to implement product recommendations such as “You may also like”.

```sql
SELECT *
FROM products
WHERE category_id = <category_id>
  AND product_id != <product_id>
ORDER BY stock_quantity DESC
LIMIT 5;
```

---

## 6 Data Generation

This project includes helper SQL functions and scripts used to generate realistic test data and simplify database operations. These utilities are intended for development, testing, and demos.

### 6.1 Customer Data Generation Function

Generates sample customers (default 1 million rows) with unique emails, passwords, and placeholder names for testing and analytics.
```sql
CREATE OR REPLACE FUNCTION generate_customers_data(num_rows INTEGER DEFAULT 1000000)
RETURNS VOID AS $$
BEGIN
    INSERT INTO customer
        (email, password, first_name, last_name)
    SELECT
        'customer' || gs || '@example.com',
        md5(random()::text),
        'FirstName' || gs,
        'LastName' || gs
    FROM generate_series(1, num_rows) AS gs;
END;
$$ LANGUAGE plpgsql;
```

### 6.2 Categories Data Generation Function

 Generates sample categories (default 100 rows) with names assigns them to categories.
```sql
CREATE OR REPLACE FUNCTION generate_categories_data(num_rows INTEGER DEFAULT 100)
RETURNS VOID AS $$
BEGIN
    INSERT INTO category (category_name)
    SELECT
        'Category name ' || gs
    FROM generate_series(1, num_rows) AS gs;
END;
$$ LANGUAGE plpgsql;
```

### 6.3 Products Data Generation Function

Generates sample products (default 100k rows) with names, descriptions, prices, stock quantities, and assigns them to categories.
```sql
CREATE OR REPLACE FUNCTION generate_products_data(num_rows INTEGER DEFAULT 100000)
RETURNS VOID AS $$
DECLARE
    cat_count INTEGER;
BEGIN
    SELECT COUNT(*) INTO cat_count FROM category;

    INSERT INTO product (category_id, name, description, price, stock_quantity)
    SELECT
        floor(random() * cat_count + 1)::INTEGER,
        'Product ' || gs,
        'Description for product ' || gs,
        (random() * 500 + 5)::DECIMAL(10,2),
        floor(random() * 10 + 1)::INTEGER
    FROM generate_series(1, num_rows) AS gs;
END;
$$ LANGUAGE plpgsql;
```

### 6.4 Orders and Order Details Generation Function

Generates sample orders and associated order details (default 2.5 million rows) with random customers, products, dates, quantities, and prices.
```sql
CREATE OR REPLACE FUNCTION generate_orders_and_details_complete(
    num_orders INTEGER DEFAULT 2500000
)
RETURNS VOID AS $$
DECLARE
    cus_count INTEGER;
    prod_count INTEGER;
BEGIN
    -- Count customers and products
    SELECT COUNT(*) INTO cus_count FROM customer;
    SELECT COUNT(*) INTO prod_count FROM product;

    -- Step 1: Insert orders with total_amount 0
    WITH new_orders AS (
        INSERT INTO orders (customer_id, order_date, customer_name, total_amount)
        SELECT
            floor(random() * cus_count + 1)::INTEGER,
            CURRENT_DATE - floor(random() * 1825)::INTEGER,
            CONCAT(c.first_name, ' ', c.last_name),
            0
        FROM generate_series(1, num_orders) AS gs
        CROSS JOIN LATERAL (
            SELECT first_name, last_name
            FROM customer
            WHERE customer_id = floor(random() * cus_count + 1)::INTEGER
        ) AS c
        RETURNING order_id
    ),
    -- Step 2: Assign exactly one product per order (cycling if needed)
    products_random AS (
        SELECT product_id, price, row_number() OVER (ORDER BY random()) AS rn
        FROM product
    ),
    new_order_details AS (
        SELECT
            o.order_id,
            p.product_id,
            p.price AS unit_price,
            floor(random() * 10 + 1)::INTEGER AS quantity
        FROM new_orders o
        JOIN LATERAL (
            SELECT product_id, price
            FROM products_random
            WHERE rn = ((o.order_id - 1) % prod_count) + 1  -- cycle through products
        ) AS p ON TRUE
    )
    INSERT INTO order_details (order_id, product_id, unit_price, quantity)
    SELECT *
    FROM new_order_details;

    -- Step 3: Update orders.total_amount
    UPDATE orders o
    SET total_amount = od.total
    FROM (
        SELECT order_id, SUM(unit_price * quantity)::DECIMAL(10,2) AS total
        FROM order_details
        GROUP BY order_id
    ) od
    WHERE o.order_id = od.order_id;

END;
$$ LANGUAGE plpgsql;
```

---
## 7 Query Performance Optimization
This section documents SQL query optimization using `EXPLAIN ANALYZE` execution plans, comparing performance before and after optimization for our e-commerce application.

| Description                         | Original Query                                                                                                                                                                                    | Execution Time (Before)                                                                                                                                              | Optimization Technique                                                                                                                                                                               | Rewritten Query                                                                                                                                                                                                                                                     | Execution Time (After)                                                                                                                                                           |
| ----------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Total Products per Category**     | `SELECT C.CATEGORY_ID, C.CATEGORY_NAME, COUNT(P.PRODUCT_ID) FROM CATEGORY C LEFT JOIN PRODUCT P ON C.CATEGORY_ID = P.CATEGORY_ID GROUP BY C.CATEGORY_ID, C.CATEGORY_NAME ORDER BY C.CATEGORY_ID;` | **102.976 ms**  <br>• Seq Scan: 15.7 ms (100k rows)  <br>• Hash Right Join: 32.3 ms  <br>• HashAggregate: 16.2 ms  <br>• Sort: 0.1 ms  <br>• Rows: 100,000 processed | **• Query:** Correlated subquery pattern  <br>**• Index:** Created covering index for index-only scans  <br>**• Impact:** Eliminated sequential scan, enabled index-only processing                  | `CREATE INDEX idx_product_category_covering ON product(category_id, product_id);`  <br>  <br>`SELECT c.category_id, c.category_name, (SELECT COUNT(*) FROM product p WHERE p.category_id = c.category_id) as product_count FROM category c ORDER BY c.category_id;` | **54.430 ms**  <br>• Index Scan (category): 0.5 ms  <br>• SubPlan loops: 100 times  <br>• Index Only Scan: 0.4 ms per loop  <br>• Heap Fetches: 0  <br>• Memory: Minimal per-row |
| **Top 10 Customers by Total Spend** | `SELECT o.customer_id, o.customer_name, SUM(o.total_amount) AS total_spent FROM orders o GROUP BY o.customer_id, o.customer_name ORDER BY total_spent DESC LIMIT 10;`                             | **2913 ms**  <br>• Seq Scan: 355 ms (2.5M rows)  <br>• HashAggregate: 1246 ms  <br>• Disk Spill: 160 MB  <br>• Memory: 8249 kB  <br>• Batches: 161                   | **• Index:** Created covering index `idx_orders_customer_covering`  <br>**• Impact:** Eliminated disk spills, enabled index-only scans  <br>**• Technique:** Same query structure with optimal index | `CREATE INDEX idx_orders_customer_covering ON orders(customer_id, customer_name) INCLUDE (total_amount);`  <br>  <br>Same query structure                                                                                                                           | **1430 ms**  <br>• Index Only Scan: 555 ms  <br>• GroupAggregate: 718 ms  <br>• No Disk Spill: 0 MB  <br>• Memory: In-memory only  <br>• Heap Fetches: 0                         |


---

## 8 Denormalization Notes

Certain fields are intentionally denormalized to improve query
performance and preserve historical accuracy.

### 8.1 Add customer_name into `orders` table

The `orders` table includes a denormalized `customer_name` column.  
This stores the customer's full name at the time of order creation.

```sql
ALTER TABLE order
ADD COLUMN customer_name VARCHAR(200);
```

Benefits:

- Faster reporting queries that list orders with customer names
- No need to join with `customers`
- Historical accuracy preserved even if the customer later updates their name

---
