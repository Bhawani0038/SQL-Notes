# Topic 14: Views and Indexes

This file covers two ideas. A **view** gives a long query a name, so your code stays short and reusable. An **index** makes queries faster by helping MySQL find data quickly.

---

## Views

A **view** is a saved `SELECT` query that behaves like a table. It does **not store data**. Every time you `SELECT` from a view, the saved query runs again and returns fresh data.

Why use views:
- **Name a long join.** You do not have to rewrite the 6-table query from file 04 every time.
- **Hide sensitive columns.** Expose only the columns people should see.
- **Keep logic in one place.** If the revenue formula changes, you change the view once.

### An order details view

Let us put the join from file 04 into a view. It also adds a status label and the net amount:

```sql
CREATE OR REPLACE VIEW v_order_details AS
SELECT o.order_id,
       o.order_date,
       o.order_status,
       CASE o.order_status
            WHEN 1 THEN 'Pending'
            WHEN 2 THEN 'Processing'
            WHEN 3 THEN 'Rejected'
            WHEN 4 THEN 'Completed'
       END AS status_name,
       o.customer_id,
       CONCAT(c.first_name, ' ', c.last_name) AS customer_name,
       s.store_name,
       CONCAT(st.first_name, ' ', st.last_name) AS staff_name,
       p.product_id,
       p.product_name,
       b.brand_name,
       cat.category_name,
       oi.quantity,
       oi.list_price,
       oi.discount,
       ROUND(oi.quantity * oi.list_price * (1 - oi.discount), 2) AS net_amount
FROM orders AS o
JOIN customers   AS c   ON o.customer_id = c.customer_id
JOIN stores      AS s   ON o.store_id    = s.store_id
JOIN staffs      AS st  ON o.staff_id    = st.staff_id
JOIN order_items AS oi  ON o.order_id    = oi.order_id
JOIN products    AS p   ON oi.product_id = p.product_id
JOIN brands      AS b   ON p.brand_id    = b.brand_id
JOIN categories  AS cat ON p.category_id = cat.category_id;
```

`CREATE OR REPLACE VIEW` means: replace the view if it exists, create it otherwise. Now an 8-table query becomes a short one:

```sql
SELECT order_id, customer_name, product_name, net_amount
FROM v_order_details
WHERE order_id = 1;
```

**Output:**

| order_id | customer_name | product_name | net_amount |
|---|---|---|---|
| 1 | Johnathan Velazquez | Electra Townie Original 7D EQ - Women's - 2016 | 479.99 |
| 1 | Johnathan Velazquez | Trek Remedy 29 Carbon Frameset - 2016 | 3347.98 |
| 1 | Johnathan Velazquez | Surly Straggler - 2016 | 2943.10 |
| 1 | Johnathan Velazquez | Electra Townie Original 7D EQ - 2016 | 1139.98 |
| 1 | Johnathan Velazquez | Trek Fuel EX 8 29 - 2016 | 2319.99 |

Analysis gets easier too. Completed revenue per store:

```sql
SELECT store_name,
       COUNT(DISTINCT order_id) AS orders,
       ROUND(SUM(net_amount), 2) AS net_revenue
FROM v_order_details
WHERE status_name = 'Completed'
GROUP BY store_name
ORDER BY net_revenue DESC;
```

**Output:**

| store_name | orders | net_revenue |
|---|---|---|
| Baldwin Bikes | 1019 | 4701205.27 |
| Santa Cruz Bikes | 284 | 1255490.45 |
| Rowlett Bikes | 142 | 705913.42 |

> **This view is also used in file 11 (the mini project).** Keep it created.

### Hiding sensitive columns

The marketing team needs each customer's city and state, but not the phone number or email:

```sql
CREATE OR REPLACE VIEW v_customer_directory AS
SELECT customer_id, first_name, last_name, city, state
FROM customers;

SELECT * FROM v_customer_directory LIMIT 3;
```

**Output:**

| customer_id | first_name | last_name | city | state |
|---|---|---|---|---|
| 1 | Debra | Burks | Orchard Park | NY |
| 2 | Kasha | Todd | Campbell | CA |
| 3 | Tameka | Fisher | Redondo Beach | CA |

### Managing views

| Task | Command |
|---|---|
| List all views | `SHOW FULL TABLES WHERE Table_type = 'VIEW';` |
| See a view's definition | `SHOW CREATE VIEW v_order_details;` |
| Remove a view | `DROP VIEW v_customer_directory;` |

```sql
SHOW FULL TABLES WHERE Table_type = 'VIEW';
```

**Output:**

| Tables_in_bike_store | Table_type |
|---|---|
| v_customer_directory | VIEW |
| v_order_details | VIEW |

**Keep in mind:**
- Views built on joins or aggregates are effectively **read-only** (no `INSERT`/`UPDATE` through them). A simple view on a single table can sometimes be updated.
- A view does not make anything faster by itself. The query inside runs at its normal speed. For speed you need indexes (below).

---

## Indexes

Think of the index at the back of a book. To find a word, you do not read every page. You look it up in the index and jump to the page. A database index works the same way: a separate sorted structure (a B-tree) that lets MySQL jump straight to the rows with a given value.

- **Without an index:** MySQL checks every row of the table. This is called a **full table scan**.
- **With an index:** MySQL looks the value up in the index and goes directly to the matching rows.

Some indexes already exist. MySQL creates one automatically for a **primary key**, a **unique key**, and a **foreign key** (this is why `product_id` and `customer_id` showed `MUL` in file 08).

```sql
SHOW INDEX FROM orders;
```

**Output:**

| Key_name | Column_name | Non_unique |
|---|---|---|
| PRIMARY | order_id | 0 |
| customer_id | customer_id | 1 |
| store_id | store_id | 1 |
| staff_id | staff_id | 1 |

### EXPLAIN: how will this query run?

`EXPLAIN` does not run the query. It shows how MySQL plans to run it, which makes it the main tool for understanding indexes. The columns to focus on:

| Column | Meaning |
|---|---|
| `type` | How rows are accessed. `ALL` = full scan (slow). `ref`, `range`, `const` = index used (fast) |
| `key` | Which index was used (`NULL` = none) |
| `rows` | MySQL's estimate of how many rows it must read |

**One step first.** Right after you load a lot of data, MySQL's table statistics can be out of date, and `EXPLAIN` may then show odd row estimates or ignore an index. Refresh the statistics once:

```sql
ANALYZE TABLE customers, orders, products;
```

There is no index on `last_name` in `customers`:

```sql
EXPLAIN SELECT * FROM customers WHERE last_name = 'Larson';
```

**Output:**

| table | type | key | rows |
|---|---|---|---|
| customers | ALL | NULL | 1445 |

`type = ALL` and `rows = 1445`: the whole table was read to find a handful of rows. Let us create an index:

```sql
CREATE INDEX idx_customers_last_name ON customers (last_name);
```

```sql
EXPLAIN SELECT * FROM customers WHERE last_name = 'Larson';
```

**Output:**

| table | type | key | rows |
|---|---|---|---|
| customers | ref | idx_customers_last_name | 7 |

Now `type = ref`, the new index is used, and only 7 rows are read instead of 1445. The bigger the table, the bigger this gap gets (seconds versus milliseconds on millions of rows).

### Composite index and the leftmost prefix rule

An index can cover several columns. **Column order matters a lot.** An index on `(state, city)` is sorted by `state` first, then by `city` within each state, just like a phone book sorted by surname and then first name.

```sql
CREATE INDEX idx_customers_state_city ON customers (state, city);
```

The index helps only when the query starts from the **leftmost** column:

```sql
EXPLAIN SELECT * FROM customers WHERE state = 'NY' AND city = 'Brooklyn';
```

**Output:**

| type | key | rows |
|---|---|---|
| ref | idx_customers_state_city | 9 |

```sql
EXPLAIN SELECT * FROM customers WHERE city = 'Brooklyn';
```

**Output:**

| type | key | rows |
|---|---|---|
| ALL | NULL | 1445 |

The first query uses the index (both `state` and `city`). The second filters only on `city`, which is like searching a phone book by first name without a surname, so the index is not used (`ALL`).

### Habits that defeat an index

**1. Wrapping the column in a function.** In file 02 we used `YEAR(order_date)`. But a function on a column in `WHERE` stops MySQL from using the index:

```sql
CREATE INDEX idx_orders_order_date ON orders (order_date);
```

```sql
EXPLAIN SELECT * FROM orders WHERE YEAR(order_date) = 2017 AND MONTH(order_date) = 3;
```

**Output:**

| type | key | rows |
|---|---|---|
| ALL | NULL | 1615 |

```sql
EXPLAIN SELECT * FROM orders
WHERE order_date >= '2017-03-01' AND order_date < '2017-04-01';
```

**Output:**

| type | key | rows |
|---|---|---|
| range | idx_orders_order_date | 67 |

Both queries mean the same thing (March 2017). The first scanned the whole table (1615 rows). The second used the index and read only 67 rows. **Fix:** do not apply the function to the column. Write the condition as a date range instead.

**2. A leading `%` in `LIKE`.** An index stores text sorted from the beginning, so `'Trek%'` can use it, but `'%Women%'` cannot:

```sql
CREATE INDEX idx_products_name ON products (product_name);
```

```sql
EXPLAIN SELECT * FROM products WHERE product_name LIKE 'Trek 8%';
```

**Output:**

| type | key | rows |
|---|---|---|
| range | idx_products_name | 2 |

```sql
EXPLAIN SELECT * FROM products WHERE product_name LIKE '%Women%';
```

**Output:**

| type | key | rows |
|---|---|---|
| ALL | NULL | 321 |

### When to create an index, and when not to

| Create one | Do not create one |
|---|---|
| Columns often used in `WHERE`, `JOIN`, `ORDER BY` | Tiny tables (like `brands` with 9 rows) |
| Columns with many different values (`email`, `last_name`) | Columns with few different values (`order_status` has only 4, or 0/1 flags) |
| Tables that are read a lot and changed rarely | Tables with heavy `INSERT`/`UPDATE` traffic |

Every index has a cost: it uses **disk space** and makes every `INSERT`/`UPDATE`/`DELETE` **slower** (the index must be updated too). So do not index every column. When a query is slow, look at `EXPLAIN` first, then add an index.

### Dropping an index

```sql
DROP INDEX idx_customers_last_name  ON customers;
DROP INDEX idx_customers_state_city ON customers;
DROP INDEX idx_orders_order_date    ON orders;
DROP INDEX idx_products_name        ON products;
```

This removes the four demo indexes so the original tables are back to how they were. Leave the views (`v_order_details`, `v_customer_directory`) in place.

## Common mistakes

- **Indexing every column.** Writes become slow and space is wasted.
- **Creating an index and never checking `EXPLAIN`.** Confirm it is actually used (look at the `key` column).
- **Using a function or calculation on an indexed column** (`WHERE price * 2 > 100`, `WHERE YEAR(d) = 2017`).
- **Wrong column order in a composite index.** Put the column that appears most often with `=` on the left.
- **Trying to insert data through a view** that is built on a join.

## Practice

1. Create a view `v_product_catalog` with `product_name`, `brand_name`, `category_name`, `list_price` (from products + brands + categories). Then use it to show the 3 most expensive Trek products.
2. `orders` already has an index on `customer_id` (created because of the foreign key). Use `EXPLAIN` to verify that `WHERE customer_id = 259` uses it.

<details>
<summary>Answers</summary>

```sql
CREATE OR REPLACE VIEW v_product_catalog AS
SELECT p.product_name, b.brand_name, cat.category_name, p.list_price
FROM products AS p
JOIN brands     AS b   ON p.brand_id    = b.brand_id
JOIN categories AS cat ON p.category_id = cat.category_id;

SELECT product_name, category_name, list_price
FROM v_product_catalog
WHERE brand_name = 'Trek'
ORDER BY list_price DESC
LIMIT 3;
```

**Output:**

| product_name | category_name | list_price |
|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | Road Bikes | 11999.99 |
| Trek Domane SLR 8 Disc - 2018 | Road Bikes | 7499.99 |
| Trek Silque SLR 8 Women's - 2017 | Road Bikes | 6499.99 |

```sql
EXPLAIN SELECT * FROM orders WHERE customer_id = 259;
```

**Output:**

| type | key | rows |
|---|---|---|
| ref | customer_id | 1 |

</details>
