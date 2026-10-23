# Topic 7: Joins

Our data is stored in separate tables. `orders` contains only `customer_id` (a number), not the customer's name. The name is in `customers`. A **join** connects rows from two or more tables through a shared column to create a larger result. This is the same primary key / foreign key relationship introduced in file 00.

## Join types at a glance

| Join | What it returns |
|---|---|
| `INNER JOIN` | Only rows with a match in both tables |
| `LEFT JOIN` | **All** rows from the left table. NULL when there is no match on the right |
| `RIGHT JOIN` | **All** rows from the right table. NULL when there is no match on the left |
| `CROSS JOIN` | Every combination of rows from both tables |
| Self join | A table joined to itself (using different aliases) |

MySQL does not support `FULL OUTER JOIN`. File 05 shows how to do this with `UNION`.

---

## INNER JOIN

Get the customer name for each order:

```sql
SELECT o.order_id, o.order_date, c.first_name, c.last_name
FROM orders AS o
INNER JOIN customers AS c
        ON o.customer_id = c.customer_id
LIMIT 5;
```

**Output:**

| order_id | order_date | first_name | last_name |
|---|---|---|---|
| 1 | 2016-01-01 | Johnathan | Velazquez |
| 2 | 2016-01-01 | Jaqueline | Cummings |
| 3 | 2016-01-02 | Joshua | Robertson |
| 4 | 2016-01-03 | Nova | Hess |
| 5 | 2016-01-03 | Arla | Ellis |

Explanation:
- `orders AS o`, `customers AS c`: **table aliases**. Short names make queries easier to write.
- `ON o.customer_id = c.customer_id`: the join condition. It means "the order's customer_id equals the customer's customer_id."
- You do not have to write `INNER`; `JOIN` by itself is an inner join.

When both tables have a column with the same name (such as `customer_id`), MySQL cannot tell which one you mean if you omit the table name:

```sql
SELECT customer_id
FROM orders AS o
JOIN customers AS c ON o.customer_id = c.customer_id;
```

**Error:**

```text
ERROR 1052 (23000): Column 'customer_id' in field list is ambiguous
```

Fix: write `o.customer_id` or `c.customer_id`. Get into the habit of writing `alias.column` in joins.

If the column has exactly the same name in both tables, you can use `USING` instead of `ON` (the column then appears only once in the output):

```sql
SELECT order_id, customer_id, first_name
FROM orders
JOIN customers USING (customer_id)
LIMIT 3;
```

**Output:**

| order_id | customer_id | first_name |
|---|---|---|
| 599 | 1 | Debra |
| 1555 | 1 | Debra |
| 1613 | 1 | Debra |

### Joining three tables

Get each product with its brand and category:

```sql
SELECT p.product_name, b.brand_name, c.category_name, p.list_price
FROM products AS p
JOIN brands AS b ON p.brand_id = b.brand_id
JOIN categories AS c ON p.category_id = c.category_id
ORDER BY p.list_price DESC
LIMIT 5;
```

**Output:**

| product_name | brand_name | category_name | list_price |
|---|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | Trek | Road Bikes | 11999.99 |
| Trek Domane SLR 8 Disc - 2018 | Trek | Road Bikes | 7499.99 |
| Trek Silque SLR 8 Women's - 2017 | Trek | Road Bikes | 6499.99 |
| Trek Domane SL Frameset - 2018 | Trek | Road Bikes | 6499.99 |
| Trek Domane SL Frameset Women's - 2018 | Trek | Road Bikes | 6499.99 |

Each new `JOIN ... ON ...` adds another table to the previous result.

---

## LEFT JOIN

With `INNER JOIN`, a row disappears if there is no match. `LEFT JOIN` keeps all rows from the left table; columns from the right table are NULL when there is no match.

Count the order lines for each product (0 for products that have never sold):

```sql
SELECT p.product_id, p.product_name, COUNT(oi.order_id) AS order_lines
FROM products AS p
LEFT JOIN order_items AS oi ON p.product_id = oi.product_id
GROUP BY p.product_id, p.product_name
ORDER BY order_lines ASC, p.product_id
LIMIT 5;
```

**Output:**

| product_id | product_name | order_lines |
|---|---|---|
| 1 | Trek 820 - 2016 | 0 |
| 121 | Surly Krampus Frameset - 2018 | 0 |
| 125 | Trek Kids' Dual Sport - 2018 | 0 |
| 154 | Trek Domane SLR 6 Disc Women's - 2018 | 0 |
| 195 | Electra Townie Go! 8i Ladies' - 2018 | 0 |

Here we use `COUNT(oi.order_id)`, not `COUNT(*)`. `COUNT(*)` also counts the row containing NULL as 1, while `COUNT(oi.order_id)` ignores NULL and returns 0 (file 03).

### Anti-join: "rows with no match"

After a LEFT JOIN, add `WHERE right_column IS NULL` to keep only rows without a match. This is a useful pattern for finding "products that have never sold":

```sql
SELECT p.product_id, p.product_name
FROM products AS p
LEFT JOIN order_items AS oi ON p.product_id = oi.product_id
WHERE oi.order_id IS NULL;
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

### Common mistake: filtering a LEFT JOIN with WHERE

Putting a condition on the right table in `WHERE` effectively turns a LEFT JOIN into an INNER JOIN because rows with NULL are removed by `WHERE`. Compare the two approaches:

```sql
-- Condition in ON: all 321 products remain
SELECT COUNT(DISTINCT p.product_id) AS products_in_result
FROM products AS p
LEFT JOIN order_items AS oi ON p.product_id = oi.product_id AND oi.quantity > 1;
```

**Output:**

| products_in_result |
|---|
| 321 |

```sql
-- Condition in WHERE: only products with a line whose quantity is > 1
SELECT COUNT(DISTINCT p.product_id) AS products_in_result
FROM products AS p
LEFT JOIN order_items AS oi ON p.product_id = oi.product_id
WHERE oi.quantity > 1;
```

**Output:**

| products_in_result |
|---|
| 257 |

Rule: put a right-table condition that is part of the "match" in `ON`. Put a condition that filters the entire result in `WHERE`.

---

## RIGHT JOIN

`RIGHT JOIN` is the mirror image of `LEFT JOIN`: reversing the table order gives the same result. In practice, people often use only `LEFT JOIN` and rearrange the tables.

```sql
SELECT p.product_id, p.product_name
FROM order_items AS oi
RIGHT JOIN products AS p ON oi.product_id = p.product_id
WHERE oi.order_id IS NULL;
```

**Output:**

| product_id | product_name |
|---|---|
| 1 | Trek 820 - 2016 |
| 121 | Surly Krampus Frameset - 2018 |
| 125 | Trek Kids' Dual Sport - 2018 |

_(The first 3 of 14 total rows are shown.)_

This is the same result as the anti-join above, with the table order reversed.

---

## CROSS JOIN

There is no condition. Every row on the left is paired with every row on the right. Rows = left rows x right rows. In this dataset: stores (3) x categories (7):

```sql
SELECT s.store_name, c.category_name
FROM stores AS s
CROSS JOIN categories AS c
ORDER BY s.store_id, c.category_id
LIMIT 5;
```

**Output:**

| store_name | category_name |
|---|---|
| Santa Cruz Bikes | Children Bicycles |
| Santa Cruz Bikes | Comfort Bicycles |
| Santa Cruz Bikes | Cruisers Bicycles |
| Santa Cruz Bikes | Cyclocross Bicycles |
| Santa Cruz Bikes | Electric Bikes |

This is useful when you need every combination (for example, the structure for a "every store x every category" report). It can also happen by mistake: if you omit `ON` from a `JOIN`, MySQL performs a cross join.

```sql
SELECT COUNT(*) AS rows_created
FROM orders, customers;
```

**Output:**

| rows_created |
|---|
| 2333675 |

1615 orders x 1445 customers = more than 2.3 million rows. On real tables, this mistake can make a query very slow. The same happens with the old comma style `FROM a, b` if you omit the condition from `WHERE`.

---

## Joining many tables

Get the full story for order number 1 in one query: customer, store, staff, product, and amount.

```sql
SELECT o.order_id,
       CONCAT(c.first_name, ' ', c.last_name) AS customer,
       s.store_name,
       CONCAT(st.first_name, ' ', st.last_name) AS handled_by,
       p.product_name,
       oi.quantity,
       ROUND(oi.quantity * oi.list_price * (1 - oi.discount), 2) AS line_total
FROM orders AS o
JOIN customers   AS c  ON o.customer_id = c.customer_id
JOIN stores      AS s  ON o.store_id    = s.store_id
JOIN staffs      AS st ON o.staff_id    = st.staff_id
JOIN order_items AS oi ON o.order_id    = oi.order_id
JOIN products    AS p  ON oi.product_id = p.product_id
WHERE o.order_id = 1;
```

**Output:**

| order_id | customer | store_name | handled_by | product_name | quantity | line_total |
|---|---|---|---|---|---|---|
| 1 | Johnathan Velazquez | Santa Cruz Bikes | Mireya Copeland | Electra Townie Original 7D EQ - Women's - 2016 | 1 | 479.99 |
| 1 | Johnathan Velazquez | Santa Cruz Bikes | Mireya Copeland | Trek Remedy 29 Carbon Frameset - 2016 | 2 | 3347.98 |
| 1 | Johnathan Velazquez | Santa Cruz Bikes | Mireya Copeland | Surly Straggler - 2016 | 2 | 2943.10 |
| 1 | Johnathan Velazquez | Santa Cruz Bikes | Mireya Copeland | Electra Townie Original 7D EQ - 2016 | 2 | 1139.98 |
| 1 | Johnathan Velazquez | Santa Cruz Bikes | Mireya Copeland | Trek Fuel EX 8 29 - 2016 | 1 | 2319.99 |

Tip: do not write a large join all at once. Start by joining 2 tables and running the query; if it looks right, add the next `JOIN`.

---

## Self join

Join a table to itself using **two different aliases**. In `staffs`, `manager_id` points to a `staff_id` in the same table, so a self join is needed to show each employee and their manager.

```sql
SELECT CONCAT(e.first_name, ' ', e.last_name) AS employee,
       CONCAT(m.first_name, ' ', m.last_name) AS manager
FROM staffs AS e
LEFT JOIN staffs AS m ON e.manager_id = m.staff_id
ORDER BY e.staff_id;
```

**Output:**

| employee | manager |
|---|---|
| Fabiola Jackson | NULL |
| Mireya Copeland | Fabiola Jackson |
| Genna Serrano | Mireya Copeland |
| Virgie Wiggins | Mireya Copeland |
| Jannette David | Fabiola Jackson |
| Marcelene Boyer | Jannette David |
| Venita Daniel | Jannette David |
| Kali Vargas | Fabiola Jackson |
| Layla Terrell | Venita Daniel |
| Bernardine Houston | Venita Daniel |

Here, `LEFT JOIN` is necessary. With `INNER JOIN`, Fabiola (whose `manager_id` is NULL, meaning she is the top manager) would disappear from the result.

---

## Join + aggregate: practical analysis

Joins become especially powerful with `GROUP BY`. We can now calculate net revenue by category for Completed orders (status 4), which was not possible in file 03:

```sql
SELECT c.category_name,
       SUM(oi.quantity) AS units_sold,
       ROUND(SUM(oi.quantity * oi.list_price * (1 - oi.discount)), 2) AS net_revenue
FROM order_items AS oi
JOIN orders     AS o ON oi.order_id   = o.order_id
JOIN products   AS p ON oi.product_id = p.product_id
JOIN categories AS c ON p.category_id = c.category_id
WHERE o.order_status = 4
GROUP BY c.category_name
ORDER BY net_revenue DESC;
```

**Output:**

| category_name | units_sold | net_revenue |
|---|---|---|
| Mountain Bikes | 1607 | 2486420.26 |
| Road Bikes | 449 | 1327155.64 |
| Cruisers Bicycles | 1865 | 866524.40 |
| Electric Bikes | 252 | 733493.75 |
| Cyclocross Bicycles | 365 | 642584.83 |
| Comfort Bicycles | 733 | 346450.00 |
| Children Bicycles | 1047 | 259986.37 |

We will calculate revenue by store and average order value later in file 06 and the mini project.

## Common mistakes

- **Forgetting `ON`** (which creates a cross join) or joining on the wrong columns (such as `o.order_id = c.customer_id`). The result may contain many rows but be incorrect.
- **Counting rows with `COUNT(*)` after a join** when a row can have multiple matches. An order has multiple items, so joining `orders` to `order_items` and using `COUNT(*)` counts **items**, not orders. Use `COUNT(DISTINCT o.order_id)` to count orders.
- **Putting a right-table filter in `WHERE` in a LEFT JOIN.** See above.

## Practice

1. Show the top 5 customers (with names) who have the most orders.
2. Show total units sold by brand (with the brand name), highest first.
3. Show each store name and how many staff members work there.

<details>
<summary>Answers</summary>

```sql
SELECT CONCAT(c.first_name, ' ', c.last_name) AS customer,
       COUNT(o.order_id) AS orders_count
FROM customers AS c
JOIN orders AS o ON c.customer_id = o.customer_id
GROUP BY c.customer_id, c.first_name, c.last_name
ORDER BY orders_count DESC, customer
LIMIT 5;
```

**Output:**

| customer | orders_count |
|---|---|
| Aleta Shepard | 3 |
| Araceli Golden | 3 |
| Bobbie Foster | 3 |
| Bridgette Guerra | 3 |
| Caren Stephens | 3 |

```sql
SELECT b.brand_name, SUM(oi.quantity) AS units_sold
FROM order_items AS oi
JOIN products AS p ON oi.product_id = p.product_id
JOIN brands   AS b ON p.brand_id    = b.brand_id
GROUP BY b.brand_name
ORDER BY units_sold DESC;
```

**Output:**

| brand_name | units_sold |
|---|---|
| Electra | 2612 |
| Trek | 1839 |
| Surly | 908 |
| Sun Bicycles | 731 |
| Pure Cycles | 376 |
| Haro | 331 |
| Heller | 138 |
| Ritchey | 118 |
| Strider | 25 |

```sql
SELECT s.store_name, COUNT(st.staff_id) AS staff_count
FROM stores AS s
LEFT JOIN staffs AS st ON s.store_id = st.store_id
GROUP BY s.store_id, s.store_name;
```

**Output:**

| store_name | staff_count |
|---|---|
| Santa Cruz Bikes | 4 |
| Baldwin Bikes | 3 |
| Rowlett Bikes | 3 |

</details>
