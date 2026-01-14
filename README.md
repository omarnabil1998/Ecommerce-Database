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

### 2- SQL Query to Find the top customers by total spending.
```sql
SELECT customer.customer_id, first_name, last_name , total 
FROM customer 
JOIN (
    SELECT customer_id, sum(total_amount) as total 
    FROM "order" 
    GROUP BY customer_id 
    ORDER BY total DESC 
    LIMIT 10
) as aggregated
ON aggregated.customer_id = customer.customer_id;
```

#### Before Optimization
```
Nested Loop  (cost=1969671.40..1969755.60 rows=10 width=61) (actual time=13624.342..13739.540 rows=10 loops=1)
  ->  Limit  (cost=1969670.97..1969671.00 rows=10 width=36) (actual time=13615.155..13615.169 rows=10 loops=1)
        ->  Sort  (cost=1969670.97..1981818.51 rows=4859015 width=36) (actual time=13504.899..13504.908 rows=10 loops=1)
              Sort Key: (sum("order".total_amount)) DESC
              Sort Method: top-N heapsort  Memory: 25kB
              ->  HashAggregate  (cost=1608621.52..1864669.40 rows=4859015 width=36) (actual time=7083.139..12859.119 rows=5000000 loops=1)
                    Group Key: "order".customer_id
                    Planned Partitions: 256  Batches: 257  Memory Usage: 8209kB  Disk Usage: 769216kB
                    ->  Seq Scan on "order"  (cost=0.00..327386.64 rows=19999764 width=9) (actual time=0.015..1343.014 rows=20000000 loops=1)
  ->  Index Scan using customer_pkey on customer  (cost=0.43..8.45 rows=1 width=29) (actual time=12.427..12.427 rows=1 loops=10)
        Index Cond: (customer_id = "order".customer_id)
Planning Time: 0.172 ms
JIT:
  Functions: 17
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 1.079 ms, Inlining 37.413 ms, Optimization 76.239 ms, Emission 51.165 ms, Total 165.896 ms
Execution Time: 13855.064 ms
```

#### After Optimization

```sql
CREATE INDEX idx_order_customer_id ON "order"(customer_id,total_amount);
ANALYZE "order";
```

```
Nested Loop  (cost=878653.26..878737.45 rows=10 width=61) (actual time=8751.149..8751.248 rows=10 loops=1)
  ->  Limit  (cost=878652.83..878652.85 rows=10 width=36) (actual time=8751.101..8751.104 rows=10 loops=1)
        ->  Sort  (cost=878652.83..889505.79 rows=4341187 width=36) (actual time=8685.737..8685.738 rows=10 loops=1)
              Sort Key: (sum("order".total_amount)) DESC
              Sort Method: top-N heapsort  Memory: 25kB
              ->  GroupAggregate  (cost=0.56..784841.34 rows=4341187 width=36) (actual time=0.402..7985.130 rows=5000000 loops=1)
                    Group Key: "order".customer_id
                    ->  Index Only Scan using idx_order_customer_id on "order"  (cost=0.56..630576.14 rows=20000072 width=9) (actual time=0.376..4587.526 rows=20000000 loops=1)
                          Heap Fetches: 900008
  ->  Index Scan using customer_pkey on customer  (cost=0.43..8.45 rows=1 width=29) (actual time=0.011..0.011 rows=1 loops=10)
        Index Cond: (customer_id = "order".customer_id)
Planning Time: 0.271 ms
JIT:
  Functions: 10
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 1.531 ms, Inlining 10.095 ms, Optimization 31.340 ms, Emission 23.953 ms, Total 66.919 ms
Execution Time: 8752.859 ms
```

| Execution Time Before Optimization | Optimization Technique                                      | Execution Time After Optimization |
| ---------------------------------- | ----------------------------------------------------------- | --------------------------------- |
| 13855.064 ms                       | Added Composite Index on on order(customer_id,total_amount) | 8752.859 ms                       |

- Now After optimzation we changed the sequential scan on order to be a covering index so we don't need to access the table 

### 3- SQL Query to Retrieve the most recent orders with customer information with 1000 orders.
```sql
SELECT customer.customer_id, first_name, last_name, order_date 
FROM customer 
JOIN (
    SELECT customer_id, order_date
    FROM "order" 
    ORDER BY order_date DESC 
    LIMIT 1000
) as aggregated
ON aggregated.customer_id = customer.customer_id;
```

#### Before Optimization
```
Nested Loop  (cost=668632.40..670893.44 rows=1000 width=33) (actual time=1805.358..1817.677 rows=1000 loops=1)
  ->  Limit  (cost=668631.96..668748.64 rows=1000 width=8) (actual time=1805.308..1815.876 rows=1000 loops=1)
        ->  Gather Merge  (cost=668631.96..2613219.09 rows=16666726 width=8) (actual time=1747.580..1758.093 rows=1000 loops=1)
              Workers Planned: 2
              Workers Launched: 2
              ->  Sort  (cost=667631.94..688465.35 rows=8333363 width=8) (actual time=1725.293..1725.336 rows=1000 loops=3)
                    Sort Key: "order".order_date DESC
                    Sort Method: top-N heapsort  Memory: 88kB
                    Worker 0:  Sort Method: top-N heapsort  Memory: 88kB
                    Worker 1:  Sort Method: top-N heapsort  Memory: 88kB
                    ->  Parallel Seq Scan on "order"  (cost=0.00..210722.63 rows=8333363 width=8) (actual time=80.974..798.673 rows=6666667 loops=3)
  ->  Memoize  (cost=0.44..8.29 rows=1 width=29) (actual time=0.002..0.002 rows=1 loops=1000)
        Cache Key: "order".customer_id
        Cache Mode: logical
        Hits: 0  Misses: 1000  Evictions: 0  Overflows: 0  Memory Usage: 128kB
        ->  Index Scan using customer_pkey on customer  (cost=0.43..8.28 rows=1 width=29) (actual time=0.001..0.001 rows=1 loops=1000)
              Index Cond: (customer_id = "order".customer_id)
Planning Time: 0.206 ms
JIT:
  Functions: 17
  Options: Inlining true, Optimization true, Expressions true, Deforming true
  Timing: Generation 2.697 ms, Inlining 192.824 ms, Optimization 36.410 ms, Emission 71.308 ms, Total 303.238 ms
Execution Time: 1818.241 ms
```

#### After Optimization

```sql
CREATE INDEX idx_order_order_date ON "order"(order_date);
ANALYZE "order";
```

```
Nested Loop  (cost=0.88..2012.79 rows=1000 width=33) (actual time=0.038..2.241 rows=1000 loops=1)
  ->  Limit  (cost=0.44..25.19 rows=1000 width=8) (actual time=0.017..0.260 rows=1000 loops=1)
        ->  Index Scan Backward using idx_order_order_date on "order"  (cost=0.44..495073.52 rows=20000072 width=8) (actual time=0.016..0.200 rows=1000 loops=1)
  ->  Memoize  (cost=0.44..8.29 rows=1 width=29) (actual time=0.002..0.002 rows=1 loops=1000)
        Cache Key: "order".customer_id
        Cache Mode: logical
        Hits: 0  Misses: 1000  Evictions: 0  Overflows: 0  Memory Usage: 128kB
        ->  Index Scan using customer_pkey on customer  (cost=0.43..8.28 rows=1 width=29) (actual time=0.001..0.001 rows=1 loops=1000)
              Index Cond: (customer_id = "order".customer_id)
Planning Time: 0.247 ms
Execution Time: 2.291 ms
```

| Execution Time Before Optimization | Optimization Technique           | Execution Time After Optimization |
| ---------------------------------- | -------------------------------- | --------------------------------- |
| 1818.241 ms                        | Added Index on order(order_date) | 2.291 ms                          |

- Before optimzation PostgreSQL had to Sequentially scan the entire order table (≈ 20M rows) Sort by order_date DESC, Keep only the top 1000 rows
- But Now it can do Backward index scan on idx_order_order_date, Reads rows already ordered and Stops immediately after 1000 rows, an index on order.customer_id is NOT needed because PostgreSQL never searches the order table by customer_id, The order rows are already chosen by order_date
- So instead of sequential scan and then sorting we added an index on order_date so that sorting isnot needed and No full table scan is needed

### 4- SQL Query to List products that have low stock quantities of less than 10 quantities.
```sql
SELECT name, stock_quantity 
FROM product 
WHERE stock_quantity < 10;
```

#### Before Optimization
```
Gather  (cost=1000.00..121935.00 rows=86942 width=19) (actual time=10.136..251.838 rows=95000 loops=1)
   Workers Planned: 2
   Workers Launched: 2
   ->  Parallel Seq Scan on product  (cost=0.00..112240.80 rows=36226 width=19) (actual time=11.617..213.543 rows=31667 loops=3)
         Filter: (stock_quantity < 10)
         Rows Removed by Filter: 1635000
 Planning Time: 0.052 ms
 JIT:
   Functions: 12
   Options: Inlining false, Optimization false, Expressions true, Deforming true
   Timing: Generation 0.800 ms, Inlining 0.000 ms, Optimization 4.421 ms, Emission 30.203 ms, Total 35.424 ms
 Execution Time: 255.235 ms
(12 rows)
```

#### After Optimization

```sql
CREATE INDEX idx_product_stock_quantity ON product(stock_quantity) INCLUDE (name) WHERE stock_quantity < 10;
ANALYZE product;
```

```
Index Only Scan using idx_product_stock_quantity on product  (cost=0.42..3119.27 rows=88190 width=19) (actual time=0.046..14.129 rows=95000 loops=1)
  Heap Fetches: 0
Planning Time: 0.200 ms
Execution Time: 17.014 ms
(4 rows)
```

| Execution Time Before Optimization | Optimization Technique                                      | Execution Time After Optimization |
| ---------------------------------- | ----------------------------------------------------------- | --------------------------------- |
| 255.235 ms                         | Added conditional covering Index on product(stock_quantity) | 17.014 ms                         |

- Added a conditional index so that it stores entries only for rows that match
- Note Index key columns are Used for WHERE conditions, ORDER BY, Index scans and Stored in sorted order while INCLUDE columns Stored only in the leaf pages Not part of sorting and Not used for filtering 
- So if we added INDEX ON product(stock_quantity, name), name is now part of the sorted key, Sorting + comparisons are more expensive, Larger index and No benefit unless we ORDER BY name
- We Put columns in the index key only if they are used to find or order rows and we put columns in INCLUDE only if they are used to return data.

### 5- SQL Query to Calculate the revenue generated from each product category.
```sql
SELECT category.category_id, category_name, SUM(quantity * unit_price) AS revenue
FROM order_details
JOIN product ON order_details.product_id = product.product_id
JOIN category ON product.category_id = category.category_id
GROUP BY category.category_id, category_name;
```

#### Before Optimization
```
Finalize GroupAggregate  (cost=3055002.66..3081087.62 rows=100000 width=50) (actual time=139708.484..140057.307 rows=100000 loops=1)
   Group Key: category.category_id
   ->  Gather Merge  (cost=3055002.66..3078337.62 rows=200000 width=50) (actual time=139708.439..139942.660 rows=300000 loops=1)
         Workers Planned: 2
         Workers Launched: 2
         ->  Sort  (cost=3054002.63..3054252.63 rows=100000 width=50) (actual time=139201.411..139216.344 rows=100000 loops=3)
               Sort Key: category.category_id
               Sort Method: external merge  Disk: 9192kB
               Worker 0:  Sort Method: external merge  Disk: 9192kB
               Worker 1:  Sort Method: external merge  Disk: 9192kB
               ->  Partial HashAggregate  (cost=2756197.59..3042278.31 rows=100000 width=50) (actual time=128886.383..139152.229 rows=100000 loops=3)
                     Group Key: category.category_id
                     Planned Partitions: 8  Batches: 9  Memory Usage: 8273kB  Disk Usage: 633352kB
                     Worker 0:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 653880kB
                     Worker 1:  Batches: 9  Memory Usage: 8273kB  Disk Usage: 655912kB
                     ->  Hash Join  (cost=144101.01..991874.70 rows=20833333 width=27) (actual time=69201.454..118379.719 rows=16666667 loops=3)
                           Hash Cond: (product.category_id = category.category_id)
                           ->  Parallel Hash Join  (cost=141214.01..934297.89 rows=20833333 width=13) (actual time=68942.745..110499.399 rows=16666667 loops=3)
                                 Hash Cond: (order_details.product_id = product.product_id)
                                 ->  Parallel Seq Scan on order_details  (cost=0.00..526805.33 rows=20833333 width=13) (actual time=9.837..64907.646 rows=16666667 loops=3)
                                 ->  Parallel Hash  (cost=107032.78..107032.78 rows=2083378 width=8) (actual time=704.843..704.844 rows=1666667 loops=3)
                                       Buckets: 262144  Batches: 64  Memory Usage: 5184kB
                                       ->  Parallel Seq Scan on product  (cost=0.00..107032.78 rows=2083378 width=8) (actual time=0.036..314.497 rows=1666667 loops=3)
                           ->  Hash  (cost=1637.00..1637.00 rows=100000 width=18) (actual time=258.001..258.001 rows=100000 loops=3)
                                 Buckets: 131072  Batches: 1  Memory Usage: 5994kB
                                 ->  Seq Scan on category  (cost=0.00..1637.00 rows=100000 width=18) (actual time=0.020..8.829 rows=100000 loops=3)
 Planning Time: 0.206 ms
 JIT:
   Functions: 75
   Options: Inlining true, Optimization true, Expressions true, Deforming true
   Timing: Generation 21.188 ms, Inlining 375.354 ms, Optimization 291.964 ms, Emission 275.882 ms, Total 964.387 ms
 Execution Time: 140128.619 ms
(32 rows)
```

#### After Optimization

```sql
CREATE MATERIALIZED VIEW category_revenue_mv AS
SELECT category.category_id, category_name, SUM(quantity * unit_price) AS revenue
FROM order_details
JOIN product ON order_details.product_id = product.product_id
JOIN category ON product.category_id = category.category_id
GROUP BY category.category_id, category_name;

explain analyze SELECT * FROM category_revenue_mv;
```

```
Seq Scan on category_revenue_mv  (cost=0.00..1768.00 rows=100000 width=24) (actual time=0.007..6.018 rows=100000 loops=1)
Planning Time: 0.031 ms
Execution Time: 8.873 ms
```

| Execution Time Before Optimization | Optimization Technique                  | Execution Time After Optimization |
| ---------------------------------- | --------------------------------------- | --------------------------------- |
| 140128.619 ms                      | Created MATERIALIZED view for the query | 8.873 ms                          |
