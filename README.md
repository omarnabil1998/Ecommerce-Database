# 📦 E-Commerce Database SQL

This project demonstrates a sample relational database schema with multiple entities and relationships.  
It also includes SQL queries for reporting and analytics.

---

## 1️⃣ Database Schema

Below is the full DB schema containing the following entities:

- `category`
- `product`
- `customer`
- `order`
- `order_details`

```sql
-- CATEGORY
CREATE TABLE category
  (
     category_id   SERIAL PRIMARY KEY,
     category_name VARCHAR(50) NOT NULL
  );

-- PRODUCT
CREATE TABLE product
  (
     product_id     SERIAL PRIMARY KEY,
     category_id    INT NOT NULL,
     name           VARCHAR(50) NOT NULL,
     description    VARCHAR(80),
     price          DECIMAL(10, 2) NOT NULL CHECK (price > 0),
     stock_quantity INT NOT NULL,
     FOREIGN KEY (category_id) REFERENCES category
  ); 

-- CUSTOMER
CREATE TABLE customer
  (
     customer_id SERIAL PRIMARY KEY,
     first_name  VARCHAR(50) NOT NULL,
     last_name   VARCHAR(50) NOT NULL,
     email       VARCHAR(50) UNIQUE,
     password    VARCHAR(80) NOT NULL
  ); 

-- ORDER
CREATE TABLE "order" (
    order_id        SERIAL PRIMARY KEY,
    customer_id     INT NOT NULL,
    order_date      DATE NOT NULL,
    total_amount    DECIMAL(10,2) NOT NULL CHECK (total_amount > 0),
    FOREIGN KEY (customer_id) REFERENCES customer
);

-- ORDER_DETAILS
CREATE TABLE order_details (
    order_detail_id SERIAL PRIMARY KEY,
    order_id        INT NOT NULL,
    product_id      INT NOT NULL,
    quantity        INT NOT NULL CHECK (quantity > 0),
    unit_price      DECIMAL(10,2) NOT NULL CHECK (unit_price > 0),
    FOREIGN KEY (product_id) REFERENCES product,
    FOREIGN KEY (order_id) REFERENCES "order"
);
```

## 2️⃣ SQL Queries

### 📅 Daily Revenue
```sql
SELECT order_date,
       SUM("order".total_amount) AS revenue
FROM   "order"
WHERE  order_date = '2025-01-12'
GROUP  BY "order".order_date; 
```

---

### 📈 Monthly Top-Selling Products
```sql
SELECT product.name,
       SUM(order_details.quantity) AS quantity
FROM   (order_details
        join "order"
          ON order_details.order_id = "order".order_id)
       join product
         ON order_details.product_id = product.product_id
WHERE  Extract(month FROM order_date) = 1
GROUP  BY product.name
ORDER  BY quantity DESC; 
```

---

### 💲 Customers Who Spent Over $500
```sql
WITH customer_totals AS
(
         SELECT   customer.first_name,
                  customer.last_name,
                  SUM("order".total_amount) AS total
         FROM     "order"
         join     customer
         ON       "order".customer_id = customer.customer_id
         WHERE    "order".order_date >= Date_trunc('month', current_date) - interval '1 month'
         and      "order".order_date < date_trunc('month', current_date)
         GROUP BY customer.first_name,
                  customer.last_name)
SELECT *
FROM   customer_totals
WHERE  total > 500;
```
