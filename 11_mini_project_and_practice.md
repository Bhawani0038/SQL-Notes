# Topic 16: Mini Project and Practice

Now we use everything together (joins, aggregation, CTEs, window functions, views). We will build a **sales report** for the bike store **owner**: where the money comes from, which customers, products and stores are doing well, and where the problems are.

> **Do this first:** the `v_order_details` view from file 09 must exist. Most queries in this file run on it. If you do not have it, run its `CREATE VIEW` query from file 09.

## Report definitions

So that the numbers are not up for debate, we fix the definitions first:

- **Revenue** = the `net_amount` of Completed orders (`status_name = 'Completed'`), that is, quantity x price after discount.
- Rejected, Pending and Processing orders do **not** count as revenue.
- **Order value** = the net amount of all items in one order.

**An important limit in the data:** Completed orders in 2018 only run from **January to March** (the April 2018 orders are still Pending or Processing, as we saw in file 03). So do **not** compare 2018 with 2016 or 2017 as if it were a full year.

---

## Part 1: Where does the money come from?

### 1. Revenue by year and store (pivot)

Putting a `CASE` inside a `SUM` turns rows into columns. This is called **conditional aggregation** (a pivot):

```sql
SELECT YEAR(order_date) AS yr,
       ROUND(SUM(CASE WHEN store_name = 'Santa Cruz Bikes' THEN net_amount END), 2) AS santa_cruz,
       ROUND(SUM(CASE WHEN store_name = 'Baldwin Bikes'    THEN net_amount END), 2) AS baldwin,
       ROUND(SUM(CASE WHEN store_name = 'Rowlett Bikes'    THEN net_amount END), 2) AS rowlett,
       ROUND(SUM(net_amount), 2) AS total
FROM v_order_details
WHERE status_name = 'Completed'
GROUP BY YEAR(order_date)
ORDER BY yr;
```

**Output:**

| yr | santa_cruz | baldwin | rowlett | total |
|---|---|---|---|---|
| 2016 | 544658.67 | 1585134.05 | 243054.16 | 2372846.88 |
| 2017 | 544196.40 | 2433947.10 | 368826.03 | 3346969.53 |
| 2018 | 166635.38 | 682124.12 | 94033.23 | 942792.73 |

Baldwin Bikes (NY) is clearly the biggest store. The 2018 figure covers only 3 months.

### 2. Seasonality: which months are strong?

We use only the full years (2016 and 2017) so the comparison is fair:

```sql
SELECT MONTH(order_date) AS month_no,
       MONTHNAME(order_date) AS month_name,
       ROUND(SUM(net_amount), 2) AS revenue
FROM v_order_details
WHERE status_name = 'Completed'
  AND YEAR(order_date) IN (2016, 2017)
GROUP BY MONTH(order_date), MONTHNAME(order_date)
ORDER BY revenue DESC
LIMIT 5;
```

**Output:**

| month_no | month_name | revenue |
|---|---|---|
| 6 | June | 581041.04 |
| 9 | September | 564761.47 |
| 10 | October | 515499.91 |
| 1 | January | 500762.49 |
| 3 | March | 488757.25 |

### 3. Store scorecard

Orders, revenue, average order value and revenue share (using a window function) for each store:

```sql
WITH store_stats AS (
    SELECT store_name,
           COUNT(DISTINCT order_id) AS orders,
           SUM(net_amount) AS revenue
    FROM v_order_details
    WHERE status_name = 'Completed'
    GROUP BY store_name
)
SELECT store_name,
       orders,
       ROUND(revenue, 2) AS revenue,
       ROUND(revenue / orders, 2) AS avg_order_value,
       ROUND(100 * revenue / SUM(revenue) OVER (), 1) AS revenue_share_pct
FROM store_stats
ORDER BY revenue DESC;
```

**Output:**

| store_name | orders | revenue | avg_order_value | revenue_share_pct |
|---|---|---|---|---|
| Baldwin Bikes | 1019 | 4701205.27 | 4613.55 | 70.6 |
| Santa Cruz Bikes | 284 | 1255490.45 | 4420.74 | 18.8 |
| Rowlett Bikes | 142 | 705913.42 | 4971.22 | 10.6 |

`SUM(revenue) OVER ()` is the total of the whole result (an empty `OVER ()` means one single window).

---

## Part 2: What is selling?

### 4. Revenue share by category

```sql
SELECT category_name,
       SUM(quantity) AS units_sold,
       ROUND(SUM(net_amount), 2) AS revenue,
       ROUND(100 * SUM(net_amount) / SUM(SUM(net_amount)) OVER (), 1) AS share_pct
FROM v_order_details
WHERE status_name = 'Completed'
GROUP BY category_name
ORDER BY revenue DESC;
```

**Output:**

| category_name | units_sold | revenue | share_pct |
|---|---|---|---|
| Mountain Bikes | 1607 | 2486418.58 | 37.3 |
| Road Bikes | 449 | 1327155.20 | 19.9 |
| Cruisers Bicycles | 1865 | 866522.82 | 13.0 |
| Electric Bikes | 252 | 733493.48 | 11.0 |
| Cyclocross Bicycles | 365 | 642584.58 | 9.6 |
| Comfort Bicycles | 733 | 346449.25 | 5.2 |
| Children Bicycles | 1047 | 259985.23 | 3.9 |

In `SUM(SUM(net_amount)) OVER ()`, the inner `SUM` gives each category's total and the outer `SUM ... OVER ()` adds those totals up. Mountain Bikes and Road Bikes together bring in most of the money. Cruisers sell the most units, but their revenue is far below Mountain and Road. So counting units alone gives an incomplete story.

### 5. Top product in each category

```sql
WITH product_revenue AS (
    SELECT category_name, product_name,
           SUM(net_amount) AS revenue
    FROM v_order_details
    WHERE status_name = 'Completed'
    GROUP BY category_name, product_name
),
ranked AS (
    SELECT category_name, product_name, revenue,
           ROW_NUMBER() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS rn
    FROM product_revenue
)
SELECT category_name, product_name, ROUND(revenue, 2) AS revenue
FROM ranked
WHERE rn = 1
ORDER BY revenue DESC;
```

**Output:**

| category_name | product_name | revenue |
|---|---|---|
| Mountain Bikes | Trek Slash 8 27.5 - 2016 | 544318.49 |
| Electric Bikes | Trek Conduit+ - 2016 | 381148.58 |
| Cyclocross Bicycles | Surly Straggler 650b - 2016 | 220713.83 |
| Road Bikes | Trek Domane SLR 6 Disc - 2017 | 181279.63 |
| Cruisers Bicycles | Electra Townie Original 7D EQ - 2016 | 79978.49 |
| Comfort Bicycles | Electra Townie Original 7D EQ - 2016 | 71308.68 |
| Children Bicycles | Electra Girl's Hawaii 1 (20-inch) - 2015/2016 | 40447.48 |

---

## Part 3: People (customers and staff)

### 6. Top 10 customers

```sql
SELECT customer_name,
       SUM(quantity) AS units_bought,
       ROUND(SUM(net_amount), 2) AS total_spent
FROM v_order_details
WHERE status_name = 'Completed'
GROUP BY customer_id, customer_name
ORDER BY total_spent DESC
LIMIT 10;
```

**Output:**

| customer_name | units_bought | total_spent |
|---|---|---|
| Melanie Hayes | 9 | 27050.71 |
| Shena Carter | 5 | 24890.62 |
| Abram Copeland | 8 | 24607.02 |
| Brigid Sharp | 5 | 20648.95 |
| Augustina Joyner | 8 | 20509.42 |
| Cindi Larson | 7 | 20177.74 |
| Adena Blake | 6 | 19329.94 |
| Bess Mcbride | 7 | 18853.35 |
| Penny Acevedo | 8 | 18670.92 |
| Danielle Bond | 8 | 18553.72 |

### 7. The order pattern of customers

How many orders did each customer place, and how many of those were Completed? This is a first attempt at understanding retention:

```sql
WITH customer_orders AS (
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(order_status = 4) AS completed_orders
    FROM orders
    GROUP BY customer_id
)
SELECT total_orders,
       COUNT(*) AS customers,
       SUM(completed_orders) AS completed_orders_total,
       ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct
FROM customer_orders
GROUP BY total_orders
ORDER BY total_orders;
```

**Output:**

| total_orders | customers | completed_orders_total | pct |
|---|---|---|---|
| 1 | 1314 | 1314 | 90.9 |
| 2 | 92 | 92 | 6.4 |
| 3 | 39 | 39 | 2.7 |

Look at the pattern: `customers` and `completed_orders_total` are **equal** in every row. That means every customer has **exactly one** Completed order. The customers with 2 or 3 orders have extra orders that are Pending, Processing or Rejected. This dataset has no "repeat buyer" behaviour (it is practice data), so the question "who are the loyal customers?" cannot be answered from it. In real analysis you check the data like this first, and only then decide which questions to ask.

---

### 8. Staff leaderboard (within each store)

```sql
SELECT store_name,
       staff_name,
       COUNT(DISTINCT order_id) AS orders_handled,
       ROUND(SUM(net_amount), 2) AS revenue,
       RANK() OVER (PARTITION BY store_name ORDER BY SUM(net_amount) DESC) AS rank_in_store
FROM v_order_details
WHERE status_name = 'Completed'
GROUP BY store_name, staff_name
ORDER BY store_name, rank_in_store;
```

**Output:**

| store_name | staff_name | orders_handled | revenue | rank_in_store |
|---|---|---|---|---|
| Baldwin Bikes | Marcelene Boyer | 521 | 2405215.61 | 1 |
| Baldwin Bikes | Venita Daniel | 498 | 2295989.66 | 2 |
| Rowlett Bikes | Kali Vargas | 71 | 375464.48 | 1 |
| Rowlett Bikes | Layla Terrell | 71 | 330448.94 | 2 |
| Santa Cruz Bikes | Genna Serrano | 147 | 643627.35 | 1 |
| Santa Cruz Bikes | Mireya Copeland | 137 | 611863.10 | 2 |

Note: some staff (managers) are missing here because they handled no orders. To include them you would need a `LEFT JOIN` from `staffs` (like the practice in file 06).

---

## Part 4: Where are the problems? (operations)

### 9. Late shipments

What percentage of Completed orders shipped after the `required_date`:

```sql
SELECT s.store_name,
       COUNT(*) AS completed_orders,
       SUM(o.shipped_date > o.required_date) AS late_orders,
       ROUND(100 * SUM(o.shipped_date > o.required_date) / COUNT(*), 1) AS late_pct
FROM orders AS o
JOIN stores AS s ON o.store_id = s.store_id
WHERE o.order_status = 4
GROUP BY s.store_id, s.store_name
ORDER BY late_pct DESC;
```

**Output:**

| store_name | completed_orders | late_orders | late_pct |
|---|---|---|---|
| Santa Cruz Bikes | 284 | 104 | 36.6 |
| Baldwin Bikes | 1019 | 317 | 31.1 |
| Rowlett Bikes | 142 | 37 | 26.1 |

`shipped_date > required_date` gives 1 when true and 0 when false, so `SUM` directly counts the late orders. This is a MySQL feature.

### 10. Low stock alert

How many products have a stock of 2 or less, per store:

```sql
SELECT s.store_name, COUNT(*) AS low_stock_products
FROM stocks AS st
JOIN stores AS s ON st.store_id = s.store_id
WHERE st.quantity <= 2
GROUP BY s.store_id, s.store_name
ORDER BY low_stock_products DESC;
```

**Output:**

| store_name | low_stock_products |
|---|---|
| Santa Cruz Bikes | 34 |
| Baldwin Bikes | 33 |
| Rowlett Bikes | 30 |

And the list of products with **zero** stock (worth reordering):

```sql
SELECT s.store_name, p.product_name, st.quantity
FROM stocks AS st
JOIN stores   AS s ON st.store_id   = s.store_id
JOIN products AS p ON st.product_id = p.product_id
WHERE st.quantity = 0
ORDER BY s.store_name, p.product_name
LIMIT 5;
```

**Output:**

| store_name | product_name | quantity |
|---|---|---|
| Baldwin Bikes | Electra Girl's Hawaii 1 (16-inch) - 2015/2016 | 0 |
| Baldwin Bikes | Electra Townie Commute Go! - 2018 | 0 |
| Baldwin Bikes | Electra Townie Commute Go! - 2018 | 0 |
| Baldwin Bikes | Electra Townie Go! 8i - 2017/2018 | 0 |
| Baldwin Bikes | Electra Townie Original 21D - 2018 | 0 |

### 11. Dead stock: never sold, yet sitting in stores

```sql
SELECT p.product_name,
       SUM(st.quantity) AS units_in_stock
FROM products AS p
JOIN stocks AS st ON st.product_id = p.product_id
LEFT JOIN order_items AS oi ON oi.product_id = p.product_id
WHERE oi.order_id IS NULL
GROUP BY p.product_id, p.product_name
HAVING SUM(st.quantity) > 0
ORDER BY units_in_stock DESC
LIMIT 5;
```

**Output:**

| product_name | units_in_stock |
|---|---|
| Trek Precaliber 12 Girl's - 2018 | 60 |
| Electra Savannah 1 (20-inch) - Girl's - 2018 | 59 |
| Trek 820 - 2016 | 55 |
| Trek Domane SLR 6 Disc Women's - 2018 | 44 |
| Electra Sweet Ride 1 (20-inch) - Girl's - 2018 | 41 |

These products never appeared in any order. Money is tied up in them, so they are candidates for a discount or a clearance sale.

---

## Practice questions (15)

Try each one yourself first, then check the answer. Levels: 1-4 easy, 5-9 medium, 10-15 hard.

1. Show Electra products priced above 1000, most expensive first.
2. How many customers are in each state?
3. How many direct reports does each manager have (with the manager's name)?
4. Orders from 2017 whose total is above 5000 (top 5).
5. The most recent order date for each store.
6. How many customers bought from **more than one store**?
7. On average, how many days did Completed orders take to ship, per store?
8. How many products cost more than the **average price of Mountain Bikes**?
9. The share (in %) of each `discount` value among order lines.
10. The top 3 customers in each state (by total spend on Completed orders).
11. The month-by-month **cumulative** order count for each store.
12. The highest-revenue month in each year.
13. Products priced above their brand's average that were **never sold**.
14. The brand with the highest revenue in each category.
15. Customers whose **most recent order is not Completed**, counted by status.

<details>
<summary>Answers</summary>

**1.**
```sql
SELECT p.product_name, p.list_price
FROM products AS p
JOIN brands AS b ON p.brand_id = b.brand_id
WHERE b.brand_name = 'Electra' AND p.list_price > 1000
ORDER BY p.list_price DESC;
```

**Output:**

| product_name | list_price |
|---|---|
| Electra Townie Commute Go! - 2018 | 2999.99 |
| Electra Townie Commute Go! Ladies' - 2018 | 2999.99 |
| Electra Townie Commute Go! - 2018 | 2999.99 |
| Electra Townie Commute Go! Ladies' - 2018 | 2999.99 |
| Electra Loft Go! 8i - 2018 | 2799.99 |

_(Showing the first 5 of 13 rows.)_

**2.**
```sql
SELECT state, COUNT(*) AS customers
FROM customers
GROUP BY state
ORDER BY customers DESC;
```

**Output:**

| state | customers |
|---|---|
| NY | 1019 |
| CA | 284 |
| TX | 142 |

**3.**
```sql
SELECT CONCAT(m.first_name, ' ', m.last_name) AS manager,
       COUNT(*) AS direct_reports
FROM staffs AS e
JOIN staffs AS m ON e.manager_id = m.staff_id
GROUP BY m.staff_id, m.first_name, m.last_name
ORDER BY direct_reports DESC;
```

**Output:**

| manager | direct_reports |
|---|---|
| Fabiola Jackson | 3 |
| Mireya Copeland | 2 |
| Jannette David | 2 |
| Venita Daniel | 2 |

**4.**
```sql
SELECT o.order_id,
       ROUND(SUM(oi.quantity * oi.list_price * (1 - oi.discount)), 2) AS total
FROM orders AS o
JOIN order_items AS oi ON o.order_id = oi.order_id
WHERE YEAR(o.order_date) = 2017
GROUP BY o.order_id
HAVING total > 5000
ORDER BY total DESC
LIMIT 5;
```

**Output:**

| order_id | total |
|---|---|
| 937 | 27050.72 |
| 930 | 24607.03 |
| 973 | 20177.75 |
| 1118 | 19329.95 |
| 1115 | 18670.93 |

**5.**
```sql
SELECT s.store_name, MAX(o.order_date) AS latest_order
FROM stores AS s
JOIN orders AS o ON s.store_id = o.store_id
GROUP BY s.store_id, s.store_name;
```

**Output:**

| store_name | latest_order |
|---|---|
| Santa Cruz Bikes | 2018-10-21 |
| Baldwin Bikes | 2018-11-18 |
| Rowlett Bikes | 2018-12-28 |

**6.**
```sql
SELECT COUNT(*) AS customers_in_multiple_stores
FROM (SELECT customer_id
      FROM orders
      GROUP BY customer_id
      HAVING COUNT(DISTINCT store_id) > 1) AS t;
```

**Output:**

| customers_in_multiple_stores |
|---|
| 0 |

The result is 0: every customer buys from only one store. This is a property of the data (each customer has a "home store").

**7.**
```sql
SELECT s.store_name,
       ROUND(AVG(DATEDIFF(o.shipped_date, o.order_date)), 1) AS avg_days_to_ship
FROM orders AS o
JOIN stores AS s ON o.store_id = s.store_id
WHERE o.order_status = 4
GROUP BY s.store_id, s.store_name;
```

**Output:**

| store_name | avg_days_to_ship |
|---|---|
| Santa Cruz Bikes | 2.0 |
| Baldwin Bikes | 2.0 |
| Rowlett Bikes | 1.9 |

**8.**
```sql
SELECT COUNT(*) AS products_above_mountain_avg
FROM products
WHERE list_price > (SELECT AVG(p.list_price)
                    FROM products AS p
                    JOIN categories AS c ON p.category_id = c.category_id
                    WHERE c.category_name = 'Mountain Bikes');
```

**Output:**

| products_above_mountain_avg |
|---|
| 99 |

**9.**
```sql
SELECT discount,
       COUNT(*) AS order_lines,
       ROUND(100 * COUNT(*) / SUM(COUNT(*)) OVER (), 1) AS pct
FROM order_items
GROUP BY discount
ORDER BY discount;
```

**Output:**

| discount | order_lines | pct |
|---|---|---|
| 0.05 | 1205 | 25.5 |
| 0.07 | 1156 | 24.5 |
| 0.10 | 1158 | 24.5 |
| 0.20 | 1203 | 25.5 |

**10.**
```sql
WITH customer_spend AS (
    SELECT c.customer_id, c.state,
           SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS spent
    FROM customers AS c
    JOIN orders      AS o  ON c.customer_id = o.customer_id
    JOIN order_items AS oi ON o.order_id    = oi.order_id
    WHERE o.order_status = 4
    GROUP BY c.customer_id, c.state
),
ranked AS (
    SELECT customer_id, state, spent,
           ROW_NUMBER() OVER (PARTITION BY state ORDER BY spent DESC) AS rn
    FROM customer_spend
)
SELECT state, customer_id, ROUND(spent, 2) AS spent
FROM ranked
WHERE rn <= 3
ORDER BY state, rn;
```

**Output:**

| state | customer_id | spent |
|---|---|---|
| CA | 1214 | 20648.95 |
| CA | 607 | 17580.72 |
| CA | 3 | 15999.96 |
| NY | 73 | 27050.72 |
| NY | 122 | 24890.62 |
| NY | 425 | 20509.43 |
| TX | 1224 | 24607.03 |
| TX | 289 | 17834.96 |
| TX | 1382 | 16491.52 |

**11.**
```sql
WITH monthly AS (
    SELECT store_id, DATE_FORMAT(order_date, '%Y-%m') AS ym, COUNT(*) AS orders_in_month
    FROM orders
    GROUP BY store_id, ym
)
SELECT store_id, ym, orders_in_month,
       SUM(orders_in_month) OVER (PARTITION BY store_id ORDER BY ym) AS running_orders
FROM monthly
ORDER BY store_id, ym
LIMIT 5;
```

**Output:**

| store_id | ym | orders_in_month | running_orders |
|---|---|---|---|
| 1 | 2016-01 | 14 | 14 |
| 1 | 2016-02 | 10 | 24 |
| 1 | 2016-03 | 10 | 34 |
| 1 | 2016-04 | 12 | 46 |
| 1 | 2016-05 | 8 | 54 |

**12.**
```sql
WITH monthly AS (
    SELECT YEAR(order_date) AS yr, MONTH(order_date) AS mth,
           SUM(net_amount) AS revenue
    FROM v_order_details
    WHERE status_name = 'Completed'
    GROUP BY YEAR(order_date), MONTH(order_date)
),
ranked AS (
    SELECT yr, mth, revenue,
           RANK() OVER (PARTITION BY yr ORDER BY revenue DESC) AS rk
    FROM monthly
)
SELECT yr, mth, ROUND(revenue, 2) AS revenue
FROM ranked
WHERE rk = 1
ORDER BY yr;
```

**Output:**

| yr | mth | revenue |
|---|---|---|
| 2016 | 9 | 271356.44 |
| 2017 | 6 | 375577.23 |
| 2018 | 1 | 381429.86 |

**13.**
```sql
SELECT p.product_name, p.list_price
FROM products AS p
WHERE p.list_price > (SELECT AVG(p2.list_price)
                      FROM products AS p2
                      WHERE p2.brand_id = p.brand_id)
  AND NOT EXISTS (SELECT 1
                  FROM order_items AS oi
                  WHERE oi.product_id = p.product_id)
ORDER BY p.list_price DESC;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Domane SLR 6 Disc Women's - 2018 | 5499.99 |
| Trek Checkpoint SL 6 - 2019 | 3799.99 |
| Trek Checkpoint ALR Frameset - 2019 | 3199.99 |
| Trek Checkpoint SL 5 Women's - 2019 | 2799.99 |
| Electra Townie Go! 8i Ladies' - 2018 | 2599.99 |
| Surly Krampus Frameset - 2018 | 2499.99 |

**14.**
```sql
WITH brand_revenue AS (
    SELECT category_name, brand_name, SUM(net_amount) AS revenue
    FROM v_order_details
    WHERE status_name = 'Completed'
    GROUP BY category_name, brand_name
),
ranked AS (
    SELECT category_name, brand_name, revenue,
           ROW_NUMBER() OVER (PARTITION BY category_name ORDER BY revenue DESC) AS rn
    FROM brand_revenue
)
SELECT category_name, brand_name, ROUND(revenue, 2) AS revenue
FROM ranked
WHERE rn = 1
ORDER BY revenue DESC;
```

**Output:**

| category_name | brand_name | revenue |
|---|---|---|
| Mountain Bikes | Trek | 1706671.76 |
| Road Bikes | Trek | 1268937.08 |
| Electric Bikes | Trek | 682027.81 |
| Cruisers Bicycles | Electra | 583682.47 |
| Cyclocross Bicycles | Surly | 421340.31 |
| Comfort Bicycles | Electra | 234396.91 |
| Children Bicycles | Electra | 188438.80 |

**15.**
```sql
WITH latest AS (
    SELECT customer_id, order_status,
           ROW_NUMBER() OVER (PARTITION BY customer_id
                              ORDER BY order_date DESC, order_id DESC) AS rn
    FROM orders
)
SELECT order_status, COUNT(*) AS customers
FROM latest
WHERE rn = 1 AND order_status <> 4
GROUP BY order_status
ORDER BY order_status;
```

**Output:**

| order_status | customers |
|---|---|
| 1 | 57 |
| 2 | 60 |
| 3 | 13 |

</details>

---

## Cleanup: removing the practice objects

During the tutorial we created some **practice tables, views, functions, procedures and triggers**. Run this to remove them all at once. It does not touch the 9 original tables (`brands` ... `stores`).

```sql
DROP TRIGGER   IF EXISTS trg_products_price_log;
DROP TRIGGER   IF EXISTS trg_sales_reduce_stock;
DROP TRIGGER   IF EXISTS trg_sales_check_qty;

DROP PROCEDURE IF EXISTS sp_customer_orders;
DROP PROCEDURE IF EXISTS sp_store_revenue;
DROP PROCEDURE IF EXISTS sp_restock;
DROP PROCEDURE IF EXISTS sp_top_products;

DROP FUNCTION  IF EXISTS fn_net_amount;
DROP FUNCTION  IF EXISTS fn_price_band;
DROP FUNCTION  IF EXISTS fn_full_name;

DROP VIEW IF EXISTS v_customer_directory, v_product_catalog;

DROP TABLE IF EXISTS review_likes, product_reviews, wishlist,
                     customers_practice, products_practice, stocks_practice,
                     price_log, sales_demo, stocks_demo, products_demo;
```

Final check: the database should now hold the original 9 tables, plus the `v_order_details` view (keep it if you like):

```sql
SHOW FULL TABLES;
```

**Output:**

| Tables_in_bike_store | Table_type |
|---|---|
| brands | BASE TABLE |
| categories | BASE TABLE |
| customers | BASE TABLE |
| order_items | BASE TABLE |
| orders | BASE TABLE |
| products | BASE TABLE |
| staffs | BASE TABLE |
| stocks | BASE TABLE |
| stores | BASE TABLE |
| v_order_details | VIEW |

If you want a complete reset, run `bike_store_full.sql` again. It rebuilds the database from scratch.

---

## What to learn next

After this tutorial, these are natural next steps:

1. **Normalization:** the rules of good table design (1NF, 2NF, 3NF).
2. **Query optimization:** go deeper into `EXPLAIN`, the slow query log, covering indexes.
3. **Users and permissions:** `CREATE USER`, `GRANT`, role-based access.
4. **Backup and restore:** `mysqldump`, `mysql < file.sql`.
5. **Connecting MySQL to code:** Python (`mysql-connector`, SQLAlchemy, `pandas.read_sql`) or Node.js.
6. **JSON columns, full-text search, partitioning** (advanced).
