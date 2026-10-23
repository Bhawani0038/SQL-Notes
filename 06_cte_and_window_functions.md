# Topics 10-11: CTEs and Window Functions

In earlier complex queries, subqueries became nested inside each other. A **CTE** gives them names and makes them easier to read. **Window functions** solve another problem: getting a group-level value without collapsing the rows.

---

## 10. CTE (Common Table Expression)

A **CTE** is a named temporary result that exists only for the duration of that statement. Syntax:

```sql
WITH cte_name AS (
    SELECT ...
)
SELECT ... FROM cte_name;
```

Rewrite the "average order value" derived table from file 05 using a CTE. The result is the same, but easier to read: first a named step, then work on it.

```sql
WITH order_totals AS (
    SELECT order_id,
           SUM(quantity * list_price * (1 - discount)) AS order_total
    FROM order_items
    GROUP BY order_id
)
SELECT ROUND(AVG(order_total), 2) AS avg_order_value,
       ROUND(MAX(order_total), 2) AS biggest_order
FROM order_totals;
```

**Output:**

| avg_order_value | biggest_order |
|---|---|
| 4761.06 | 29147.03 |

### Multiple CTEs

You can write multiple CTEs separated by commas, and a later CTE can use an earlier one. Top 5 customers based only on Completed orders:

```sql
WITH order_totals AS (
    SELECT order_id,
           SUM(quantity * list_price * (1 - discount)) AS order_total
    FROM order_items
    GROUP BY order_id
),
customer_spend AS (
    SELECT o.customer_id,
           COUNT(*) AS orders_count,
           SUM(t.order_total) AS total_spent
    FROM orders AS o
    JOIN order_totals AS t ON o.order_id = t.order_id
    WHERE o.order_status = 4
    GROUP BY o.customer_id
)
SELECT CONCAT(c.first_name, ' ', c.last_name) AS customer,
       cs.orders_count,
       ROUND(cs.total_spent, 2) AS total_spent
FROM customer_spend AS cs
JOIN customers AS c ON cs.customer_id = c.customer_id
ORDER BY cs.total_spent DESC
LIMIT 5;
```

**Output:**

| customer | orders_count | total_spent |
|---|---|---|
| Melanie Hayes | 1 | 27050.72 |
| Shena Carter | 1 | 24890.62 |
| Abram Copeland | 1 | 24607.03 |
| Brigid Sharp | 1 | 20648.95 |
| Augustina Joyner | 1 | 20509.43 |

Written with subqueries, this query would be nested three levels deep. Here, each step has its own name, and reading from top to bottom makes the logic clear.

### CTE vs subquery vs view

| | Kab use karein |
|---|---|
| Subquery | Small, one-time task |
| CTE | Multi-step logic or reusing one result multiple times in a query. Exists for one statement only |
| View (file 09) | Reuse the same query across statements. Saved in the database |

### Recursive CTE

**A recursive CTE** calls itself until no new data is found. It is useful for hierarchies (manager > employee). First, a small example that counts from 1 to 5:

```sql
WITH RECURSIVE numbers AS (
    SELECT 1 AS n                 -- anchor: starting point
    UNION ALL
    SELECT n + 1 FROM numbers     -- recursive step: next value from previous row
    WHERE n < 5                   -- stopping condition
)
SELECT n FROM numbers;
```

**Output:**

| n |
|---|
| 1 |
| 2 |
| 3 |
| 4 |
| 5 |

There are three parts: **anchor** (the first row), **recursive step** (the next rows based on the previous result), and **stopping condition** (`WHERE`). If you forget the stopping condition, the query keeps running (MySQL eventually stops it at a limit).

Build the full organization chart for `staffs`. Anchor: the row whose `manager_id` is NULL (the top manager). Recursive step: staff whose manager is in the previous row.

```sql
WITH RECURSIVE org AS (
    SELECT staff_id,
           CAST(CONCAT(first_name, ' ', last_name) AS CHAR(300)) AS path,
           1 AS level
    FROM staffs
    WHERE manager_id IS NULL
    UNION ALL
    SELECT s.staff_id,
           CONCAT(org.path, ' > ', s.first_name, ' ', s.last_name),
           org.level + 1
    FROM staffs AS s
    JOIN org ON s.manager_id = org.staff_id
)
SELECT level, path
FROM org
ORDER BY path;
```

**Output:**

| level | path |
|---|---|
| 1 | Fabiola Jackson |
| 2 | Fabiola Jackson > Jannette David |
| 3 | Fabiola Jackson > Jannette David > Marcelene Boyer |
| 3 | Fabiola Jackson > Jannette David > Venita Daniel |
| 4 | Fabiola Jackson > Jannette David > Venita Daniel > Bernardine Houston |
| 4 | Fabiola Jackson > Jannette David > Venita Daniel > Layla Terrell |
| 2 | Fabiola Jackson > Kali Vargas |
| 2 | Fabiola Jackson > Mireya Copeland |
| 3 | Fabiola Jackson > Mireya Copeland > Genna Serrano |
| 3 | Fabiola Jackson > Mireya Copeland > Virgie Wiggins |

`path` shows the full reporting chain from top to bottom. `level` tells how many steps away someone is from the top manager (1 = top manager).

---

## 11. Window Functions

`GROUP BY` collapses rows: grouping 1615 orders by store produces 3 store rows. Sometimes you want **every row to remain** while also showing a group value beside it (for example, "what share of the order is this line?"). A **window function** does that.

Syntax:

```sql
function_name(...) OVER (
    PARTITION BY column     -- groups (optional)
    ORDER BY column         -- order within a group (optional)
)
```

- `OVER (...)` indicates that this is a window function.
- `PARTITION BY` divides rows into groups (windows) without removing rows.
- `ORDER BY` sets the order within the window.

### Aggregate ko window banana

Show every line in order 1, its order total, and its share of the order:

```sql
SELECT order_id, item_id,
       ROUND(quantity * list_price * (1 - discount), 2) AS line_total,
       ROUND(SUM(quantity * list_price * (1 - discount)) OVER (PARTITION BY order_id), 2) AS order_total,
       ROUND(100 * quantity * list_price * (1 - discount)
             / SUM(quantity * list_price * (1 - discount)) OVER (PARTITION BY order_id), 1) AS pct_of_order
FROM order_items
WHERE order_id IN (1, 2)
ORDER BY order_id, item_id;
```

**Output:**

| order_id | item_id | line_total | order_total | pct_of_order |
|---|---|---|---|---|
| 1 | 1 | 479.99 | 10231.05 | 4.7 |
| 1 | 2 | 3347.98 | 10231.05 | 32.7 |
| 1 | 3 | 2943.10 | 10231.05 | 28.8 |
| 1 | 4 | 1139.98 | 10231.05 | 11.1 |
| 1 | 5 | 2319.99 | 10231.05 | 22.7 |
| 2 | 1 | 557.99 | 1697.97 | 32.9 |
| 2 | 2 | 1139.98 | 1697.97 | 67.1 |

Every line remains, and `order_total` appears alongside each line. This is not possible with `GROUP BY`.

### Ranking: ROW_NUMBER, RANK, DENSE_RANK

All three assign a number to each row, but behave differently when there are **ties (equal values)**:

| Function | Behavior for ties |
|---|---|
| `ROW_NUMBER()` | Alag-alag number (1, 2, 3, 4), tie ignore |
| `RANK()` | Ties get the same rank, then a **gap** (1, 2, 2, 4) |
| `DENSE_RANK()` | Tie ko same rank, **bina gap** (1, 2, 2, 3) |

Several products in category 1 (Children Bicycles) have the same price, so ties are visible here:

```sql
SELECT product_name, list_price,
       ROW_NUMBER() OVER (ORDER BY list_price DESC, product_id) AS row_num,
       RANK()       OVER (ORDER BY list_price DESC) AS rnk,
       DENSE_RANK() OVER (ORDER BY list_price DESC) AS dense_rnk
FROM products
WHERE category_id = 1
ORDER BY list_price DESC, product_id
LIMIT 8;
```

**Output:**

| product_name | list_price | row_num | rnk | dense_rnk |
|---|---|---|---|---|
| Electra Straight 8 3i (20-inch) - Boy's - 2017 | 489.99 | 1 | 1 | 1 |
| Electra Townie 3i EQ (20-inch) - Boys' - 2017 | 489.99 | 2 | 1 | 1 |
| Trek Superfly 24 - 2017/2018 | 489.99 | 3 | 1 | 1 |
| Trek Superfly 20 - 2018 | 399.99 | 4 | 4 | 2 |
| Electra Straight 8 1 (20-inch) - Boy's - 2018 | 389.99 | 5 | 5 | 3 |
| Trek Precaliber 24 21-speed Boy's - 2018 | 369.99 | 6 | 6 | 4 |
| Trek Precaliber 24 21-speed Girl's - 2018 | 369.99 | 7 | 6 | 4 |
| Electra Superbolt 3i 20" - 2018 | 369.99 | 8 | 6 | 4 |

With different prices, notice that `rnk` has gaps while `dense_rnk` does not.

### "Top N in each group"

You cannot put a window function directly in `WHERE` because `WHERE` runs before window functions (see the execution order in file 03):

```sql
SELECT product_name,
       ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY list_price DESC) AS rn
FROM products
WHERE ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY list_price DESC) <= 2;
```

**Error:**

```text
ERROR 3593 (HY000): You cannot use the window function 'row_number' in this context.'
```

Fix: calculate the rank in a CTE (or derived table) first, then apply `WHERE` outside it. Get the 2 most expensive products in each category:

```sql
WITH ranked AS (
    SELECT category_id, product_name, list_price,
           ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY list_price DESC, product_id) AS rn
    FROM products
)
SELECT category_id, product_name, list_price
FROM ranked
WHERE rn <= 2
ORDER BY category_id, rn;
```

**Output:**

| category_id | product_name | list_price |
|---|---|---|
| 1 | Electra Straight 8 3i (20-inch) - Boy's - 2017 | 489.99 |
| 1 | Electra Townie 3i EQ (20-inch) - Boys' - 2017 | 489.99 |
| 2 | Electra Townie Go! 8i - 2017/2018 | 2599.99 |
| 2 | Electra Townie Balloon 7i EQ - 2018 | 899.99 |
| 3 | Electra Townie Commute Go! - 2018 | 2999.99 |
| 3 | Electra Townie Commute Go! Ladies' - 2018 | 2999.99 |
| 4 | Trek Boone 7 Disc - 2018 | 3999.99 |
| 4 | Trek Boone 7 - 2017 | 3499.99 |
| 5 | Trek Powerfly 8 FS Plus - 2017 | 4999.99 |
| 5 | Trek Powerfly 7 FS - 2018 | 4999.99 |
| 6 | Trek Fuel EX 9.8 27.5 Plus - 2017 | 5299.99 |
| 6 | Trek Remedy 9.8 - 2017 | 5299.99 |
| 7 | Trek Domane SLR 9 Disc - 2018 | 11999.99 |
| 7 | Trek Domane SLR 8 Disc - 2018 | 7499.99 |

### Running total and month-over-month

First calculate monthly revenue for Completed orders in a CTE, then apply window functions to it.

**Running total** (`SUM() OVER (ORDER BY ...)`) returns the cumulative sum up to each row:

```sql
WITH monthly AS (
    SELECT DATE_FORMAT(o.order_date, '%Y-%m') AS ym,
           SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.order_status = 4
    GROUP BY ym
)
SELECT ym,
       ROUND(revenue, 2) AS revenue,
       ROUND(SUM(revenue) OVER (ORDER BY ym), 2) AS running_total
FROM monthly
ORDER BY ym
LIMIT 6;
```

**Output:**

| ym | revenue | running_total |
|---|---|---|
| 2016-01 | 215146.42 | 215146.42 |
| 2016-02 | 148506.36 | 363652.78 |
| 2016-03 | 180600.33 | 544253.11 |
| 2016-04 | 164399.08 | 708652.19 |
| 2016-05 | 197179.09 | 905831.28 |
| 2016-06 | 205463.98 | 1111295.26 |

**LAG / LEAD**: look at the value in the previous / next row. For month-over-month growth, use `LAG(revenue)`:

```sql
WITH monthly AS (
    SELECT DATE_FORMAT(o.order_date, '%Y-%m') AS ym,
           SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.order_status = 4
    GROUP BY ym
)
SELECT ym,
       ROUND(revenue, 2) AS revenue,
       ROUND(LAG(revenue) OVER (ORDER BY ym), 2) AS prev_month,
       ROUND(100 * (revenue - LAG(revenue) OVER (ORDER BY ym))
             / NULLIF(LAG(revenue) OVER (ORDER BY ym), 0), 1) AS growth_pct
FROM monthly
ORDER BY ym
LIMIT 6;
```

**Output:**

| ym | revenue | prev_month | growth_pct |
|---|---|---|---|
| 2016-01 | 215146.42 | NULL | NULL |
| 2016-02 | 148506.36 | 215146.42 | -31.0 |
| 2016-03 | 180600.33 | 148506.36 | 21.6 |
| 2016-04 | 164399.08 | 180600.33 | -9.0 |
| 2016-05 | 197179.09 | 164399.08 | 19.9 |
| 2016-06 | 205463.98 | 197179.09 | 4.2 |

`prev_month` is NULL for the first month because there is no earlier row. `NULLIF(..., 0)` prevents division by zero (file 02). `LEAD()` does the opposite and looks at the next row.

**Moving average**: the average over the last 3 months, used to smooth a trend. Here you must specify a **frame**, which tells how many rows to include in the window:

```sql
WITH monthly AS (
    SELECT DATE_FORMAT(o.order_date, '%Y-%m') AS ym,
           SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.order_status = 4
    GROUP BY ym
)
SELECT ym,
       ROUND(revenue, 2) AS revenue,
       ROUND(AVG(revenue) OVER (ORDER BY ym ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3m
FROM monthly
ORDER BY ym
LIMIT 6;
```

**Output:**

| ym | revenue | moving_avg_3m |
|---|---|---|
| 2016-01 | 215146.42 | 215146.42 |
| 2016-02 | 148506.36 | 181826.39 |
| 2016-03 | 180600.33 | 181417.70 |
| 2016-04 | 164399.08 | 164501.92 |
| 2016-05 | 197179.09 | 180726.17 |
| 2016-06 | 205463.98 | 189014.05 |

`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` means the current row and the 2 preceding rows. For the first 2 months, fewer rows are available, so the average uses a smaller window.

### NTILE: divide into equal groups

`NTILE(4)` divides rows into 4 equal groups (quartiles). Divide products into 4 price groups (1 = most expensive):

```sql
SELECT product_name, list_price,
       NTILE(4) OVER (ORDER BY list_price DESC) AS price_quartile
FROM products
ORDER BY list_price DESC
LIMIT 3;
```

**Output:**

| product_name | list_price | price_quartile |
|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | 11999.99 | 1 |
| Trek Domane SLR 8 Disc - 2018 | 7499.99 | 1 |
| Trek Silque SLR 8 Women's - 2017 | 6499.99 | 1 |

---

## Window function vs GROUP BY

| | `GROUP BY` | Window function |
|---|---|---|
| Rows | **Fewer** rows, one per group | **All** rows remain |
| When to use | "Total for each store" | "Store total beside each order," ranking, running total |
| Filtering | `HAVING` | Calculate in a CTE/subquery, then filter outside with `WHERE` |

## Common mistakes

- **Putting a window function in `WHERE` or `GROUP BY`** (see the error above). Calculate it in a CTE.
- **Using `LAG`/running total without `ORDER BY`.** The order is undefined, so the result is unreliable. Always write `OVER (ORDER BY ...)`.
- **Ignoring ties.** `ROW_NUMBER` assigns different numbers even for ties, so some tied products may be excluded from the "top 3." To account for ties, use `RANK` or `DENSE_RANK`, and add a tie-breaker column (such as `product_id`) to `ROW_NUMBER`.

## Practice

1. Within each store, rank staff members by their number of Completed orders.
2. Show the most expensive product for each brand (`ROW_NUMBER` + CTE).
3. For each year, calculate total Completed revenue and growth percentage compared with the previous year (`LAG`).

<details>
<summary>Answers</summary>

```sql
SELECT st.store_id,
       CONCAT(st.first_name, ' ', st.last_name) AS staff,
       COUNT(o.order_id) AS completed_orders,
       RANK() OVER (PARTITION BY st.store_id ORDER BY COUNT(o.order_id) DESC) AS rank_in_store
FROM staffs AS st
LEFT JOIN orders AS o ON st.staff_id = o.staff_id AND o.order_status = 4
GROUP BY st.store_id, st.staff_id, st.first_name, st.last_name
ORDER BY st.store_id, rank_in_store;
```

**Output:**

| store_id | staff | completed_orders | rank_in_store |
|---|---|---|---|
| 1 | Genna Serrano | 147 | 1 |
| 1 | Mireya Copeland | 137 | 2 |
| 1 | Fabiola Jackson | 0 | 3 |
| 1 | Virgie Wiggins | 0 | 3 |
| 2 | Marcelene Boyer | 521 | 1 |
| 2 | Venita Daniel | 498 | 2 |
| 2 | Jannette David | 0 | 3 |
| 3 | Kali Vargas | 71 | 1 |
| 3 | Layla Terrell | 71 | 1 |
| 3 | Bernardine Houston | 0 | 3 |

```sql
WITH ranked AS (
    SELECT brand_id, product_name, list_price,
           ROW_NUMBER() OVER (PARTITION BY brand_id ORDER BY list_price DESC, product_id) AS rn
    FROM products
)
SELECT b.brand_name, r.product_name, r.list_price
FROM ranked AS r
JOIN brands AS b ON r.brand_id = b.brand_id
WHERE r.rn = 1
ORDER BY r.list_price DESC;
```

**Output:**

| brand_name | product_name | list_price |
|---|---|---|
| Trek | Trek Domane SLR 9 Disc - 2018 | 11999.99 |
| Electra | Electra Townie Commute Go! - 2018 | 2999.99 |
| Heller | Heller Bloodhound Trail - 2018 | 2599.00 |
| Surly | Surly Karate Monkey 27.5+ Frameset - 2017 | 2499.99 |
| Sun Bicycles | Sun Bicycles ElectroLite - 2017 | 1559.99 |
| Haro | Haro Shift R3 - 2017 | 1469.99 |
| Ritchey | Ritchey Timberwolf Frameset - 2016 | 749.99 |
| Pure Cycles | Pure Cycles Western 3-Speed - Women's - 2015/2016 | 449.00 |
| Strider | Strider Strider 20 Sport - 2018 | 289.99 |

```sql
WITH yearly AS (
    SELECT YEAR(o.order_date) AS yr,
           SUM(oi.quantity * oi.list_price * (1 - oi.discount)) AS revenue
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.order_status = 4
    GROUP BY YEAR(o.order_date)
)
SELECT yr,
       ROUND(revenue, 2) AS revenue,
       ROUND(100 * (revenue - LAG(revenue) OVER (ORDER BY yr))
             / LAG(revenue) OVER (ORDER BY yr), 1) AS growth_pct
FROM yearly
ORDER BY yr;
```

**Output:**

| yr | revenue | growth_pct |
|---|---|---|
| 2016 | 2372849.20 | NULL |
| 2017 | 3346972.65 | 41.1 |
| 2018 | 942793.40 | -71.8 |

</details>
