# E-Commerce Database Documentation

This document describes the database schema, entity relationships, and example analytical SQL queries used in the e-commerce system. The goal is to provide a clear, professional overview for development and showcase purposes.

---

## 1. Database Schema
This section defines the core database structure used to store and manage e-commerce data efficiently.

```sql
CREATE DATABASE ecommerce

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

## 2. Entity Relationships
This section explains how the main tables are connected and how data flows between tables

| Parent   | Child         | Relationship | Foreign Key              |
| -------- | ------------- | ------------ | ------------------------ |
| customer | order         | 1 : M        | order.customer_id        |
| category | product       | 1 : M        | product.category_id      |
| order    | order_details | 1 : M        | order_details.order_id   |
| product  | order_details | 1 : M        | order_details.product_id |


---

## 3. ERD Diagram
This section provides a visual representation of the database schema and the relationships between its entities.
![ERD Diagram](erd_diagram.png)

---
## 4. Analytical SQL Queries
This section includes useful SQL queries for getting insights from the e-commerce data. These examples show how to check daily revenue, find top-selling products, and identify high-value customers.

### 4.1 Daily Revenue Report

```sql
SELECT
    DATE(order_date) AS report_date,
    SUM(total_amount) AS total_revenue
FROM orders
WHERE DATE(order_date) = DATE '2025-01-18'
GROUP BY DATE(order_date);
```

### 4.2 Top-Selling Products in a Month

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

### 4.3 Customers Who Spent More Than 500 in a Month

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

### 4.4 Search for Products Containing the Word `camera`

```sql
SELECT *
FROM products
WHERE name ILIKE '%camera%'
   OR description ILIKE '%camera%';
```

### 4.5 Suggest Popular Products From the Same Category, Excluding the Selected Product

```sql
SELECT *
FROM products
WHERE category_id = <category_id>
  AND product_id != <product_id>
ORDER BY stock_quantity DESC
LIMIT 5;
```

---

## 5. Denormalization Notes
This section highlights fields that store redundant or snapshot data to improve query performance and simplify reporting,

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
