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
     search_vector  tsvector GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED,
     FOREIGN KEY (category_id) REFERENCES category
  ); 

CREATE INDEX product_search_index on product using GIN(search_vector);

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

-- SALE_HISTROY
CREATE TABLE sale_history (
    sale_id         SERIAL PRIMARY KEY,
    order_date      DATE NOT NULL,
    customer_id     INT NOT NULL,
    product_id      INT NOT NULL,
    total_amount    DECIMAL(10,2) NOT NULL CHECK (total_amount > 0),
    quantity        INT NOT NULL CHECK (quantity > 0),
    FOREIGN KEY (product_id) REFERENCES product,
    FOREIGN KEY (customer_id) REFERENCES customer
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

---

### SQL query to search for all products with the word "camera" in either the product name or description
```sql
select * from product WHERE name LIKE '%camera%' or description LIKE '%camera%';
```
> Note that this query performs a sequential scan sequential scan of the table which might be slow.

---

### Alternative solution by using full Text Search
```sql
CREATE INDEX product_search_index on product using GIN(search_vector);

select * from product WHERE search_vector @@ to_tsquery('camera');
```
- Created a new column of type **tsvector** in the product table that combine the name and description columns

---

### Write a trigger to Create a sale history [Above customer , product], when a new order is made in the "Orders" table, automatically generates a sale history record for that order,capturing details such as the order date, customer, product, total amount, and quantity. The trigger should be triggered on Order insertion
```sql
CREATE OR REPLACE FUNCTION set_sale_history()
    RETURNS TRIGGER
    LANGUAGE PLPGSQL
AS
$$
BEGIN
  INSERT INTO sale_history (order_date, customer_id, product_id, total_amount,quantity)
  select order_date, customer_id, product_id, total_amount, quantity from order_details join "order" on order_details.order_id = "order".order_id where order_details.order_id = New.order_id;
  RETURN NULL;
END;
$$;

CREATE TRIGGER set_sale_history_trigger
  AFTER INSERT
  ON "order_details"
  FOR EACH ROW
EXECUTE PROCEDURE set_sale_history();
```
- Created a new Table sale_history

---

### A transaction query to lock the field quantity with product id = 211 from being updated
```sql
BEGIN;
SELECT product_id FROM product WHERE product_id=211 FOR UPDATE;
COMMIT;
```

---

### Write a transaction query to lock row with product id = 211 from being updated
```sql
BEGIN;
SELECT * FROM product WHERE product_id=211 FOR UPDATE;
COMMIT;
```