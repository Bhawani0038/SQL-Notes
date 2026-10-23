# Topic 6: Aggregation (COUNT, SUM, AVG, GROUP BY, HAVING)

So far, each query has worked row by row. **Aggregation** combines many rows into one value: "number of orders," "total revenue," or "average price."

## Aggregate functions

| Function | What it does |
|---|---|
| `COUNT(*)` | Counts rows (including those with NULL values) |
| `COUNT(col)` | Counts the column's **non-NULL** values |
| `COUNT(DISTINCT col)` | Counts unique non-NULL values |
| `SUM(col)` | Sum |
| `AVG(col)` | Average |
| `MIN(col)`, `MAX(col)` | Minimum / maximum |

```sql
SELECT COUNT(*) AS total_customers,
       COUNT(phone) AS with_phone,
       COUNT(DISTINCT state) AS states
FROM customers;
```

**Output:**

| total_customers | with_phone | states |
|---|---|---|
| 1445 | 178 | 3 |

Remember the difference between `COUNT(*)` and `COUNT(phone)`: NULL values in `phone` are not counted by `COUNT(phone)`. This is rule 3 for NULL (file 02): aggregate functions ignore NULL. The same applies to `AVG`, `SUM`, `MIN`, and `MAX`.

```sql
SELECT MIN(list_price) AS cheapest,
       MAX(list_price) AS costliest,
       ROUND(AVG(list_price), 2) AS avg_price
FROM products;
```

**Output:**

| cheapest | costliest | avg_price |
|---|---|---|
| 89.99 | 11999.99 | 1520.59 |

### Net revenue

In file 01, we calculated `line_total`. Putting it inside `SUM` gives the total revenue. In this tutorial, **net revenue** means this:

```sql
SELECT SUM(quantity) AS units_sold,
       ROUND(SUM(quantity * list_price * (1 - discount)), 2) AS net_revenue
FROM order_items;
```

**Output:**

| units_sold | net_revenue |
|---|---|
| 7078 | 7689116.56 |

This includes all orders, including Rejected and Pending ones. To filter by status, first join with `orders` (file 04).

---

## GROUP BY: answers for each group

An overall total gives one number, but often we want "orders for each store" or "orders for each year." `GROUP BY` divides rows into groups and aggregates each group.

Orders for each store:

```sql
SELECT store_id, COUNT(*) AS total_orders
FROM orders
GROUP BY store_id;
```

**Output:**

| store_id | total_orders |
|---|---|
| 1 | 348 |
| 2 | 1093 |
| 3 | 174 |

Orders for each year. Expressions (functions) can also be used in `GROUP BY`:

```sql
SELECT YEAR(order_date) AS yr, COUNT(*) AS total_orders
FROM orders
GROUP BY YEAR(order_date)
ORDER BY yr;
```

**Output:**

| yr | total_orders |
|---|---|
| 2016 | 635 |
| 2017 | 688 |
| 2018 | 292 |

Customer email domains (completing the example from file 02):

```sql
SELECT SUBSTRING_INDEX(email, '@', -1) AS email_domain,
       COUNT(*) AS customers
FROM customers
GROUP BY email_domain
ORDER BY customers DESC;
```

**Output:**

| email_domain | customers |
|---|---|
| yahoo.com | 305 |
| gmail.com | 305 |
| msn.com | 291 |
| aol.com | 275 |
| hotmail.com | 269 |

MySQL allows aliases in `GROUP BY` (other databases may not; there you must write the full expression).

Product summary for each category:

```sql
SELECT category_id,
       COUNT(*) AS products,
       ROUND(AVG(list_price), 2) AS avg_price,
       MAX(list_price) AS max_price
FROM products
GROUP BY category_id
ORDER BY avg_price DESC;
```

**Output:**

| category_id | products | avg_price | max_price |
|---|---|---|---|
| 5 | 24 | 3281.66 | 4999.99 |
| 7 | 60 | 3175.36 | 11999.99 |
| 4 | 10 | 2542.79 | 3999.99 |
| 6 | 60 | 1649.76 | 5299.99 |
| 3 | 78 | 730.41 | 2999.99 |
| 2 | 30 | 682.12 | 2599.99 |
| 1 | 59 | 287.79 | 489.99 |

### Grouping by multiple columns

Groups are formed from the **unique combinations** of those columns:

```sql
SELECT YEAR(order_date) AS yr, order_status, COUNT(*) AS total_orders
FROM orders
GROUP BY YEAR(order_date), order_status
ORDER BY yr, order_status;
```

**Output:**

| yr | order_status | total_orders |
|---|---|---|
| 2016 | 3 | 15 |
| 2016 | 4 | 620 |
| 2017 | 3 | 17 |
| 2017 | 4 | 671 |
| 2018 | 1 | 62 |
| 2018 | 2 | 63 |
| 2018 | 3 | 13 |
| 2018 | 4 | 154 |

Notice: 2016 and 2017 have only Rejected (3) and Completed (4) orders. Pending (1) and Processing (2) appear only in 2018 because those orders are still in progress. `GROUP BY` makes this pattern visible.

---

## WHERE vs HAVING

Both filter data, but at **different stages**:

- `WHERE` filters rows **before** groups are created. You cannot use aggregate functions in it.
- `HAVING` filters groups **after** they are created. You can use aggregate functions in it.

```sql
SELECT brand_id, COUNT(*) AS products
FROM products
GROUP BY brand_id
HAVING COUNT(*) > 30
ORDER BY products DESC;
```

**Output:**

| brand_id | products |
|---|---|
| 9 | 135 |
| 1 | 118 |

Using both: first filter for products from 2018 with `WHERE`, then keep groups whose average price is above 1000:

```sql
SELECT category_id, ROUND(AVG(list_price), 2) AS avg_price
FROM products
WHERE model_year = 2018
GROUP BY category_id
HAVING AVG(list_price) > 1000
ORDER BY avg_price DESC;
```

**Output:**

| category_id | avg_price |
|---|---|
| 5 | 3295.23 |
| 7 | 3211.80 |
| 4 | 2532.99 |
| 6 | 1662.44 |

Using an aggregate in `WHERE` causes an error:

```sql
SELECT brand_id, COUNT(*) FROM products WHERE COUNT(*) > 30 GROUP BY brand_id;
```

**Error:**

```text
ERROR 1111 (HY000): Invalid use of group function
```

### The order in which a query runs

We write clauses in one order, but MySQL processes them in this order. This explains why aliases do not work in `WHERE` (file 01) and why `HAVING` can use aggregates:

| Step | Clause | What it does |
|---|---|---|
| 1 | `FROM` | Choose the table |
| 2 | `WHERE` | Filter rows |
| 3 | `GROUP BY` | Create groups |
| 4 | `HAVING` | Filter groups |
| 5 | `SELECT` | Create columns/aliases |
| 6 | `ORDER BY` | Sort |
| 7 | `LIMIT` | Limit rows |

In this order, the `SELECT` alias is created at step 5, so step 2 (`WHERE`) cannot see it. Step 6 (`ORDER BY`) can.

---

## Common mistakes

**Selecting a non-aggregate column that is not in `GROUP BY`.** MySQL's default mode (`ONLY_FULL_GROUP_BY`) rejects this:

```sql
SELECT store_id, order_id, COUNT(*) FROM orders GROUP BY store_id;
```

**Error:**

```text
ERROR 1055 (42000): Expression #2 of SELECT list is not in GROUP BY clause and contains nonaggregated column 'bike_store.orders.order_id' which is not functionally dependent on columns in GROUP BY clause; this is incompatible with sql_mode=only_full_group_by
```

Why: each store's group contains many `order_id` values. Which one should appear in a single row? Rule: `SELECT` must contain either columns from `GROUP BY` or columns inside aggregate functions.

**Treating `COUNT(col)` and `COUNT(*)` as the same.** They give different results when the column contains NULL values (see the customer example above).

**Not checking the row count alongside an average.** An average over 2 rows is not as reliable as one over 2000 rows. Check `COUNT(*)` alongside `AVG`.

## Bonus: WITH ROLLUP and GROUP_CONCAT

`WITH ROLLUP` adds a **grand total** row after the groups (the grouping column is NULL in the total row):

```sql
SELECT store_id, COUNT(*) AS total_orders
FROM orders
GROUP BY store_id WITH ROLLUP;
```

**Output:**

| store_id | total_orders |
|---|---|
| 1 | 348 |
| 2 | 1093 |
| 3 | 174 |
| NULL | 1615 |

`GROUP_CONCAT` combines a group's values into one text string:

```sql
SELECT category_id, GROUP_CONCAT(DISTINCT brand_id ORDER BY brand_id) AS brand_ids
FROM products
GROUP BY category_id;
```

**Output:**

| category_id | brand_ids |
|---|---|
| 1 | 1,2,6,7,9 |
| 2 | 1,7 |
| 3 | 1,4,7 |
| 4 | 8,9 |
| 5 | 1,7,9 |
| 6 | 2,3,5,7,8,9 |
| 7 | 8,9 |

## Practice

1. Find the total `quantity` for each `product_id` in `order_items`; show the top 5.
2. How many staff members are at each store?
3. Show the months (year + month) with more than 60 orders, highest count first.

<details>
<summary>Answers</summary>

```sql
SELECT product_id, SUM(quantity) AS units_sold
FROM order_items
GROUP BY product_id
ORDER BY units_sold DESC
LIMIT 5;
```

**Output:**

| product_id | units_sold |
|---|---|
| 6 | 167 |
| 13 | 157 |
| 16 | 156 |
| 23 | 154 |
| 7 | 154 |

```sql
SELECT store_id, COUNT(*) AS staff_count
FROM staffs
GROUP BY store_id;
```

**Output:**

| store_id | staff_count |
|---|---|
| 1 | 4 |
| 2 | 3 |
| 3 | 3 |

```sql
SELECT YEAR(order_date) AS yr, MONTH(order_date) AS mth, COUNT(*) AS total_orders
FROM orders
GROUP BY YEAR(order_date), MONTH(order_date)
HAVING COUNT(*) > 60
ORDER BY total_orders DESC;
```

**Output:**

| yr | mth | total_orders |
|---|---|---|
| 2018 | 4 | 125 |
| 2018 | 3 | 68 |
| 2016 | 9 | 67 |
| 2017 | 3 | 67 |
| 2017 | 8 | 65 |
| 2017 | 10 | 65 |
| 2016 | 10 | 64 |
| 2016 | 8 | 63 |
| 2017 | 6 | 63 |

</details>
