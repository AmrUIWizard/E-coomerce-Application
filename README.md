This is planning phase for e-commerce

## Database Schema Script

```sql
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(100) NOT NULL UNIQUE,
    password TEXT NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL
);

CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    category_name VARCHAR(100) NOT NULL UNIQUE
);

CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    category_id INTEGER NOT NULL REFERENCES categories(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    description TEXT,
    price NUMERIC(10, 2) NOT NULL,
    stock_quantity INTEGER NOT NULL DEFAULT 0 CHECK (stock_quantity >= 0)
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id) ON DELETE CASCADE,
    order_date TIMESTAMP NOT NULL DEFAULT NOW(),
    total_amount NUMERIC(10, 2) NOT NULL
);

CREATE TABLE order_details (
    id SERIAL PRIMARY KEY,
    order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
    product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE CASCADE,
    unit_price NUMERIC(10, 2) NOT NULL,
    quantity INTEGER NOT NULL DEFAULT 1 CHECK (quantity >= 1)
);
```

## The Relationships Between Entities
| Parent     | Child         | Relationship | Foreign Key              |
| ---------- | ------------- | ------------ | ------------------------ |
| customers  | orders        | 1 : M        | orders.customer_id       |
| categories | products      | 1 : M        | products.category_id     |
| orders     | order_details | 1 : M        | order_details.order_id   |
| products   | order_details | 1 : M        | order_details.product_id |

## The Erd Diagram
![ERD Diagram](erd_diagram.png)

## SQL query that generate a daily report of the total revenue for a specific date

```sql
SELECT 
    DATE(order_date) AS report_date SUM(total_amount)
FROM orders 
WHERE DATE(order_date) = 'YYYY/MM/DD' #Example: '2025-01-18'
GROUP BY DATE(order_date);
```

## SQL query that generate a monthly report of the top-selling products in a given month.

```sql
SELECT
    p.id AS product_id,
    p.name AS product_nam,
    SUM(od.quantity) AS total_sold
FROM order_details od
JOIN orders o ON od.order_id
JOIN products p ON od.product_id 
WHERE DATE_TRUNC('month', o.order_date) = DATE_TRUNC('month', DATE 'YYYY/MM/DD') #Example: '2025-01-01'
GROUP BY p.id, p.name
ORDER BY total_sold DESC
LIMIT 3;
```

## SQL query that retrieve a list of customers who have placed orders totaling more than $500 in the past month. 

```sql
SELECT
    c.id AS customer_id
    c.first_name || ' ' || c.last_name AS customer_name
    SUM(o.total_amount) AS total_spent
FROM customers c
JOIN orders o ON o.id = c.customer_id
WHERE DATE_TRUNC('month', o.order_date) = DATE 'YYYY/MM/DD' #Example: '2025-01-01'
GROUP BY c.id, customer_name
HAVING SUM(o.total_amount) > 500
ORDER BY total_spent DESC
```

## ### Denormalization of Orders Table

To reduce query time and avoid joins with the customers table, a new column customer_name is added to the orders table. This stores the customer’s full name as a snapshot at the time of order creation.

```sql
ALTER TABLE orders
ADD COLUMN customer_name VARCHAR(200);
```

This improves performance for reports and order listings while preserving historical customer names.