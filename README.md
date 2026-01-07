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

### Suggest related products in the same category, excluding the Purchased products by the current customer from the recommendations
```sql
SELECT name, category_id 
FROM product
WHERE category_id IN
    (SELECT DISTINCT category_id
     FROM order_details
     JOIN "order" ON order_details.order_id = "order".order_id
     JOIN product ON order_details.product_id = product.product_id
     WHERE customer_id=2)
AND product_id NOT IN 
    (SELECT DISTINCT product_id
     FROM order_details
     JOIN "order" ON order_details.order_id = "order".order_id
     WHERE customer_id=2);
```

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

## Optimize Queries

### 1- SQL Query to Retrieve the total number of products in each category.
```sql
SELECT category_name, total_products
FROM category  
JOIN (
    SELECT category_id, count(*) as total_products 
    FROM product 
    GROUP BY category_id
) as aggregated 
ON aggregated.category_id = category.category_id;
```

#### Before Optimization
```
Merge Join  (cost=250891.37..282219.15 rows=100902 width=22) (actual time=893.117..1029.586 rows=100000 loops=1)
  Merge Cond: (category.category_id = product.category_id)
  ->  Index Scan using category_pkey on category  (cost=0.29..3244.29 rows=100000 width=18) (actual time=0.018..16.455 rows=100000 loops=1)
  ->  Finalize GroupAggregate  (cost=250891.08..276454.57 rows=100902 width=12) (actual time=886.833..988.114 rows=100000 loops=1)
        Group Key: product.category_id
        ->  Gather Merge  (cost=250891.08..274436.53 rows=201804 width=12) (actual time=886.798..959.981 rows=300000 loops=1)
              Workers Planned: 2
              Workers Launched: 2
              ->  Sort  (cost=249891.06..250143.31 rows=100902 width=12) (actual time=842.082..852.148 rows=100000 loops=3)
                    Sort Key: product.category_id
                    Sort Method: external merge  Disk: 2552kB
                    Worker 0:  Sort Method: external merge  Disk: 2552kB
                    Worker 1:  Sort Method: external merge  Disk: 2552kB
                    ->  Partial HashAggregate  (cost=224219.74..241504.79 rows=100902 width=12) (actual time=721.360..799.639 rows=100000 loops=3)
                          Group Key: product.category_id
                          Batches: 5  Memory Usage: 8241kB  Disk Usage: 11160kB
                          Worker 0:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 7776kB
                          Worker 1:  Batches: 5  Memory Usage: 8241kB  Disk Usage: 11064kB
                          ->  Parallel Seq Scan on product  (cost=0.00..107032.32 rows=2083332 width=4) (actual time=0.027..219.962 rows=1666667 loops=3)
Planning Time: 0.113 ms
JIT:
  Functions: 29
  Options: Inlining false, Optimization false, Expressions true, Deforming true
  Timing: Generation 2.776 ms, Inlining 0.000 ms, Optimization 3.261 ms, Emission 28.593 ms, Total 34.630 ms
Execution Time: 1038.367 ms
(25 rows)
```

#### After Optimization

```sql
CREATE INDEX idx_product_category_id ON product(category_id);
ANALYZE product;
```

```
Hash Join  (cost=3887.46..104863.24 rows=99686 width=22) (actual time=61.433..375.721 rows=100000 loops=1)
   Hash Cond: (product.category_id = category.category_id)
   ->  Finalize GroupAggregate  (cost=1000.46..100717.69 rows=99686 width=12) (actual time=39.911..328.565 rows=100000 loops=1)
         Group Key: product.category_id
         ->  Gather Merge  (cost=1000.46..98723.97 rows=199372 width=12) (actual time=39.882..312.140 rows=100000 loops=1)
               Workers Planned: 2
               Workers Launched: 2
               ->  Partial GroupAggregate  (cost=0.43..74711.47 rows=99686 width=12) (actual time=6.115..209.494 rows=33333 loops=3)
                     Group Key: product.category_id
                     ->  Parallel Index Only Scan using idx_product_category_id on product  (cost=0.43..63297.91 rows=2083340 width=4) (actual time=0.155..126.217 rows=1666667 loops=3)
                           Heap Fetches: 0
   ->  Hash  (cost=1637.00..1637.00 rows=100000 width=18) (actual time=21.340..21.341 rows=100000 loops=1)
         Buckets: 131072  Batches: 1  Memory Usage: 6102kB
         ->  Seq Scan on category  (cost=0.00..1637.00 rows=100000 width=18) (actual time=0.014..8.644 rows=100000 loops=1)
 Planning Time: 0.349 ms
 JIT:
   Functions: 20
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 1.664 ms, Inlining 0.000 ms, Optimization 0.881 ms, Emission 16.885 ms, Total 19.430 ms
 Execution Time: 379.727 ms
(20 rows)
```

| Execution Time Before Optimization | Optimization Technique                             | Execution Time After Optimization |
| ---------------------------------- | -------------------------------------------------- | --------------------------------- |
| 1038.367 ms                        | Added Index on Foreign key on Product(category_id) | 379.727 ms                        |

- PostgreSQL have 2 main ways to Group rows that have the same category_id: HashAggregate & GroupAggregate
- In HashAggregate Think of a hash table, PostgreSQL will Read a row, Compute a hash of category_id and Put/update it in a hash table in memory
  - It Works even if data is random order but it Needs memory, If hash table is large → spills to disk → slow and Extra CPU cost to hash every row
- In GroupAggregate it works only if the input rows are already sorted by the GROUP BY column, No hash table. Just a counter, Very low memory usage
- original query used HashAggregate so Before the index existed, PostgreSQL read the table with Parallel Seq Scan on product so Rows are read in physical order ≠ ordered by category_id
- In a B-Tree index All values in are stored in sorted order so When PostgreSQL scans this index It reads entries in order
- PostgreSQL switched to GroupAggregate after the index so Now PostgreSQL sees Parallel Index Only Scan using idx_product_category_id Meaning it Reads rows from index and Rows are already ordered by category_id so PostgreSQL does not need to read the table at all.
- Using the subquery helped because it reduced the amount of data to be joined, turned millions of product rows into ~100k aggregated rows