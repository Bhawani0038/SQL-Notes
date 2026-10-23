# Topics 8-9: Set Operations (UNION) and Subqueries

Joins combine tables **side by side** (adding columns). This file introduces two other techniques: **UNION** combines rows **vertically**, and a **subquery** places one query inside another.

---

## 8. UNION, UNION ALL

`UNION` stacks the results of two or more `SELECT` statements into one result.

**Rules:**
1. Each `SELECT` must return the **same number** of columns.
2. The columns must have **compatible data types** (number under number, text under text).
3. Output column names come from the **first SELECT**.

Our system has two kinds of people: customers and staff. Combine them into one contact list:

```sql
SELECT first_name, last_name, email, 'Customer' AS person_type
FROM customers
UNION ALL
SELECT first_name, last_name, email, 'Staff' AS person_type
FROM staffs
ORDER BY last_name, first_name
LIMIT 5;
```

**Output:**

| first_name | last_name | email | person_type |
|---|---|---|---|
| Ester | Acevedo | ester.acevedo@gmail.com | Customer |
| Jamika | Acevedo | jamika.acevedo@yahoo.com | Customer |
| Penny | Acevedo | penny.acevedo@yahoo.com | Customer |
| Bettyann | Acosta | bettyann.acosta@gmail.com | Customer |
| Shery | Acosta | shery.acosta@yahoo.com | Customer |

Write `ORDER BY` and `LIMIT` at the very end; they apply to the entire combined result. `'Customer'` and `'Staff'` above are fixed text values that identify where each row came from.

### UNION vs UNION ALL

| | Duplicates | Speed |
|---|---|---|
| `UNION` | Removes duplicates | Slower (extra work to find duplicates) |
| `UNION ALL` | Keeps duplicates | Faster |

Rule of thumb: if you know there are no duplicates, or you want to keep them, use `UNION ALL`. Compare these two ways to list states from customers and stores:

```sql
SELECT state FROM customers
UNION
SELECT state FROM stores;
```

**Output:**

| state |
|---|
| NY |
| CA |
| TX |

```sql
SELECT state FROM customers
UNION ALL
SELECT state FROM stores;
```

**Output:**

| state |
|---|
| NY |
| CA |
| CA |

_(The first 3 of 1448 total rows are shown.)_

`UNION` removed duplicates and returned only 3 unique states. `UNION ALL` returned the full stack of 1445 + 3 rows (the total row count appears below the output).

### INTERSECT and EXCEPT

Starting with MySQL 8.0.31, `INTERSECT` (rows found in both sets) and `EXCEPT` (rows found in the first set but not the second) are also available. They work in MySQL 8.0.46. Older MySQL versions do not support them.

Customers who placed an order in **both** 2016 and 2017:

```sql
SELECT customer_id FROM orders WHERE YEAR(order_date) = 2016
INTERSECT
SELECT customer_id FROM orders WHERE YEAR(order_date) = 2017;
```

**Output:**

| customer_id |
|---|
| 12 |
| 46 |
| 43 |
| 24 |
| 13 |

_(The first 5 of 13 total rows are shown.)_

Customers who ordered in 2016 but **not** in 2017:

```sql
SELECT customer_id FROM orders WHERE YEAR(order_date) = 2016
EXCEPT
SELECT customer_id FROM orders WHERE YEAR(order_date) = 2017;
```

**Output:**

| customer_id |
|---|
| 259 |
| 1212 |
| 523 |
| 175 |
| 1324 |

_(The first 5 of 612 total rows are shown.)_

### How to do a FULL OUTER JOIN

MySQL does not have `FULL OUTER JOIN`. You can create one by combining a `LEFT JOIN` and a `RIGHT JOIN` with `UNION`:

```sql
SELECT a.id, b.id
FROM table_a AS a LEFT JOIN table_b AS b ON a.id = b.id
UNION
SELECT a.id, b.id
FROM table_a AS a RIGHT JOIN table_b AS b ON a.id = b.id;
```

(`UNION` keeps rows that appear in both joins only once.) This is a syntax template. This dataset does not contain tables named `table_a` or `table_b`.

---

## 9. Subqueries

A **subquery** is a `SELECT` statement written inside parentheses within another query. The inner query runs first, and the outer query uses its result.

A subquery can appear in three places: `WHERE`, the `SELECT` list, and `FROM`. We will look at five types below.

### 1. Scalar subquery (returns a single value)

Show products priced **above the overall average**:

```sql
SELECT product_name, list_price
FROM products
WHERE list_price > (SELECT AVG(list_price) FROM products)
ORDER BY list_price
LIMIT 5;
```

**Output:**

| product_name | list_price |
|---|---|
| Surly Straggler - 2018 | 1549.00 |
| Surly Straggler - 2018 | 1549.00 |
| Surly Straggler 650b - 2018 | 1549.00 |
| Surly Straggler - 2016 | 1549.00 |
| Surly Straggler 650b - 2018 | 1549.00 |

The inner query returns a single value (the average), so the outer query can compare against it with `>`. If the inner query returns **more than one row** and you use an operator such as `=` or `>`, you get an error:

```sql
SELECT product_name FROM products
WHERE list_price = (SELECT list_price FROM products);
```

**Error:**

```text
ERROR 1242 (21000): Subquery returns more than 1 row
```

A scalar subquery also works in the `SELECT` list:

```sql
SELECT product_name, list_price,
       ROUND(list_price - (SELECT AVG(list_price) FROM products), 2) AS diff_from_avg
FROM products
LIMIT 3;
```

**Output:**

| product_name | list_price | diff_from_avg |
|---|---|---|
| Trek 820 - 2016 | 379.99 | -1140.60 |
| Ritchey Timberwolf Frameset - 2016 | 749.99 | -770.60 |
| Surly Wednesday Frameset - 2016 | 999.99 | -520.60 |

### 2. IN subquery (returns a list of values)

Get products in the "Road Bikes" category without using a join:

```sql
SELECT product_name, list_price
FROM products
WHERE category_id IN (SELECT category_id
                      FROM categories
                      WHERE category_name = 'Road Bikes')
LIMIT 5;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Emonda S 4 - 2017 | 1499.99 |
| Trek Domane SL 6 - 2017 | 3499.99 |
| Trek Silque SLR 7 Women's - 2017 | 5999.99 |
| Trek Silque SLR 8 Women's - 2017 | 6499.99 |
| Surly Steamroller - 2017 | 875.99 |

Count customers who placed any order in 2018:

```sql
SELECT COUNT(*) AS customers_ordered_2018
FROM customers
WHERE customer_id IN (SELECT customer_id
                      FROM orders
                      WHERE YEAR(order_date) = 2018);
```

**Output:**

| customers_ordered_2018 |
|---|
| 270 |

**The NULL trap with `NOT IN`.** If the inner list contains **even one NULL**, `NOT IN` evaluates to "unknown" for every row, and no rows are returned:

```sql
SELECT (1 NOT IN (2, 3))    AS without_null,
       (1 NOT IN (2, NULL)) AS with_null;
```

**Output:**

| without_null | with_null |
|---|---|
| 1 | NULL |

`with_null` returned NULL (meaning it is not "true"). So when a column might contain NULL, use `NOT EXISTS` instead of `NOT IN`.

### 3. EXISTS / NOT EXISTS

`EXISTS` only checks whether the inner query returns **any rows** (true/false). It does not matter what the query returns, so we write `SELECT 1`.

Find products that have never sold (we found these with a LEFT JOIN in file 04):

```sql
SELECT p.product_id, p.product_name
FROM products AS p
WHERE NOT EXISTS (SELECT 1
                  FROM order_items AS oi
                  WHERE oi.product_id = p.product_id);
```

**Output:**

| product_id | product_name |
|---|---|
| 1 | Trek 820 - 2016 |
| 121 | Surly Krampus Frameset - 2018 |
| 125 | Trek Kids' Dual Sport - 2018 |
| 154 | Trek Domane SLR 6 Disc Women's - 2018 |
| 195 | Electra Townie Go! 8i Ladies' - 2018 |

_(The first 5 of 14 total rows are shown.)_

Here, `p.product_id` in the inner query is a column from the outer query. This kind of subquery is called **correlated** (see below).

### 4. Correlated subquery

A **correlated subquery** runs again for each row of the outer query because it depends on an outer column. Show products priced above **their category's average**:

```sql
SELECT p.product_name, p.category_id, p.list_price
FROM products AS p
WHERE p.list_price > (SELECT AVG(p2.list_price)
                      FROM products AS p2
                      WHERE p2.category_id = p.category_id)
ORDER BY p.list_price DESC
LIMIT 5;
```

**Output:**

| product_name | category_id | list_price |
|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | 7 | 11999.99 |
| Trek Domane SLR 8 Disc - 2018 | 7 | 7499.99 |
| Trek Domane SL Frameset Women's - 2018 | 7 | 6499.99 |
| Trek Silque SLR 8 Women's - 2017 | 7 | 6499.99 |
| Trek Domane SL Frameset - 2018 | 7 | 6499.99 |

For each row, the subquery calculates the average for its category. This can be slow on large tables, so it is often better to write this using a join or window function (file 06).

### 5. Derived table (subquery in FROM)

A subquery in `FROM` works like a **temporary table** for the outer query. Important: a derived table **must have an alias**.

First calculate each order's total, then find the average of those totals:

```sql
SELECT ROUND(AVG(order_total), 2) AS avg_order_value
FROM (SELECT order_id,
             SUM(quantity * list_price * (1 - discount)) AS order_total
      FROM order_items
      GROUP BY order_id) AS order_totals;
```

**Output:**

| avg_order_value |
|---|
| 4761.06 |

If you forget the alias:

```sql
SELECT AVG(order_total)
FROM (SELECT order_id, SUM(quantity * list_price) AS order_total
      FROM order_items GROUP BY order_id);
```

**Error:**

```text
ERROR 1248 (42000): Every derived table must have its own alias
```

---

## Subquery, JOIN, EXISTS: when to use each

| Zaroorat | Behtar tareeka |
|---|---|
| Need columns from another table in the result | `JOIN` |
| Only need to check whether a match exists | `EXISTS` / `NOT EXISTS` |
| Compare against one calculated value (average, max) | Scalar subquery |
| Aggregate first, then calculate further | Derived table (or CTE, file 06) |
| "No match found" | `NOT EXISTS` or `LEFT JOIN ... IS NULL` |

`IN (subquery)` and `JOIN` often return the same result. The difference is that a `JOIN` can **duplicate** rows when there are multiple matches, while `IN` does not. Modern MySQL optimizes both well, so choose whichever is easier to read.

## Common mistakes

- **Using `=` with a subquery that returns multiple rows** (see the error above). Use `IN`.
- **Using `NOT IN` when NULL is possible** (see above).
- **Forgetting the derived table alias.**
- **Not using aliases in a correlated subquery**, which can make it unclear which query a column belongs to. The example uses separate aliases `p` and `p2`.

## Practice

1. Count customers who placed **no orders** in 2018 (using `NOT EXISTS`).
2. Show the names and prices of products priced above **their brand's average** (top 5 by price).
3. Show 5 orders whose total is **above the average order value**.

<details>
<summary>Answers</summary>

```sql
SELECT COUNT(*) AS customers_no_order_2018
FROM customers AS c
WHERE NOT EXISTS (SELECT 1
                  FROM orders AS o
                  WHERE o.customer_id = c.customer_id
                    AND YEAR(o.order_date) = 2018);
```

**Output:**

| customers_no_order_2018 |
|---|
| 1175 |

```sql
SELECT p.product_name, p.brand_id, p.list_price
FROM products AS p
WHERE p.list_price > (SELECT AVG(p2.list_price)
                      FROM products AS p2
                      WHERE p2.brand_id = p.brand_id)
ORDER BY p.list_price DESC
LIMIT 5;
```

**Output:**

| product_name | brand_id | list_price |
|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | 9 | 11999.99 |
| Trek Domane SLR 8 Disc - 2018 | 9 | 7499.99 |
| Trek Silque SLR 8 Women's - 2017 | 9 | 6499.99 |
| Trek Domane SL Frameset Women's - 2018 | 9 | 6499.99 |
| Trek Domane SL Frameset - 2018 | 9 | 6499.99 |

```sql
SELECT order_id, ROUND(SUM(quantity * list_price * (1 - discount)), 2) AS order_total
FROM order_items
GROUP BY order_id
HAVING order_total > (SELECT AVG(t.order_total)
                      FROM (SELECT SUM(quantity * list_price * (1 - discount)) AS order_total
                            FROM order_items
                            GROUP BY order_id) AS t)
ORDER BY order_total DESC
LIMIT 5;
```

**Output:**

| order_id | order_total |
|---|---|
| 1541 | 29147.03 |
| 937 | 27050.72 |
| 1506 | 25574.96 |
| 1482 | 25365.43 |
| 1364 | 24890.62 |

</details>
