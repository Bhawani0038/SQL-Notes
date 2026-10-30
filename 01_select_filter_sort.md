# Topics 2-4: SELECT, Filtering, and Sorting

In this file, we will learn to **retrieve** data (`SELECT`), **filter** it (`WHERE`), and **sort** it (`ORDER BY`, `CASE`). All queries use `bike_store`, so first run `USE bike_store;`.

---

## 2. SELECT basics

`SELECT` means "show these columns from a table."

```sql
SELECT * FROM brands;
```

**Output:**

| brand_id | brand_name |
|---|---|
| 1 | Electra |
| 2 | Haro |
| 3 | Heller |
| 4 | Pure Cycles |
| 5 | Ritchey |
| 6 | Strider |
| 7 | Sun Bicycles |
| 8 | Surly |
| 9 | Trek |

`*` means "all columns." It is fine for learning, but in real work request only the columns you need. This makes the query faster and the output clearer.

```sql
SELECT product_name, list_price FROM products;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek 820 - 2016 | 379.99 |
| Ritchey Timberwolf Frameset - 2016 | 749.99 |
| Surly Wednesday Frameset - 2016 | 999.99 |
| Trek Fuel EX 8 29 - 2016 | 2899.99 |
| Heller Shagamaw Frame - 2016 | 1320.99 |

_(The first 5 of 321 total rows are shown.)_

### Alias (`AS`): give a column a new name

An alias changes the name only in the output; it does not change the table. Use backticks for a name containing spaces.

```sql
SELECT product_name AS bike, list_price AS `price in USD`
FROM products;
```

**Output:**

| bike | price in USD |
|---|---|
| Trek 820 - 2016 | 379.99 |
| Ritchey Timberwolf Frameset - 2016 | 749.99 |
| Surly Wednesday Frameset - 2016 | 999.99 |

_(The first 3 of 321 total rows are shown.)_

### DISTINCT: remove duplicates

`customers` has 1445 rows, but how many different states are there?

```sql
SELECT DISTINCT state FROM customers;
```

**Output:**

| state |
|---|
| NY |
| CA |
| TX |

With multiple columns, `DISTINCT` returns unique **combinations**:

```sql
SELECT DISTINCT state, city FROM customers;
```

**Output:**

| state | city |
|---|---|
| NY | Orchard Park |
| CA | Campbell |
| CA | Redondo Beach |
| NY | Uniondale |
| CA | Sacramento |

_(The first 5 of 195 total rows are shown.)_

### LIMIT and OFFSET: show only some rows

```sql
SELECT product_name, list_price FROM products LIMIT 3;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek 820 - 2016 | 379.99 |
| Ritchey Timberwolf Frameset - 2016 | 749.99 |
| Surly Wednesday Frameset - 2016 | 999.99 |

`OFFSET` skips the earlier rows. It is useful for pagination (page 2 = skip the first 3 rows and get the next 3):

```sql
SELECT product_name, list_price FROM products LIMIT 3 OFFSET 3;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Fuel EX 8 29 - 2016 | 2899.99 |
| Heller Shagamaw Frame - 2016 | 1320.99 |
| Surly Ice Cream Truck Frameset - 2016 | 469.99 |

> Without `ORDER BY`, "the first 3 rows" has no definite meaning when using `LIMIT`. The order is not guaranteed, so `LIMIT` is often used with `ORDER BY` (below).

---

## 3. Filtering: WHERE

`WHERE` specifies which rows you want. A row is returned only when the condition is **true**.

```sql
SELECT product_name, list_price
FROM products
WHERE list_price > 5000;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Fuel EX 9.8 27.5 Plus - 2017 | 5299.99 |
| Trek Remedy 9.8 - 2017 | 5299.99 |
| Trek Silque SLR 7 Women's - 2017 | 5999.99 |
| Trek Silque SLR 8 Women's - 2017 | 6499.99 |
| Trek Domane SLR 6 Disc - 2017 | 5499.99 |

_(The first 5 of 13 total rows are shown.)_

Comparison operators: `=`, `<>` (or `!=`), `<`, `>`, `<=`, `>=`. SQL uses `=` for equality, not `==`.

Write text and dates in quotes:

```sql
SELECT first_name, last_name, city
FROM customers
WHERE city = 'Brooklyn';
```

**Output:**

| first_name | last_name | city |
|---|---|---|
| Stephaine | Riddle | Brooklyn |
| Chere | Hardin | Brooklyn |
| Rosa | Kinney | Brooklyn |
| Saran | Moses | Brooklyn |
| Vernetta | Banks | Brooklyn |

_(The first 5 of 9 total rows are shown.)_

### AND, OR, NOT

- `AND`: both conditions are true.
- `OR`: either condition is true.
- `NOT`: reverses a condition.

Road Bikes (category 7) made by Trek (brand 9):

```sql
SELECT product_name, list_price
FROM products
WHERE category_id = 7 AND brand_id = 9;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Emonda S 4 - 2017 | 1499.99 |
| Trek Domane SL 6 - 2017 | 3499.99 |
| Trek Silque SLR 7 Women's - 2017 | 5999.99 |

_(The first 3 of 53 total rows are shown.)_

**Common mistake: using AND and OR together without parentheses.** `AND` has higher precedence than `OR`. This query can be easy to misread:

```sql
SELECT COUNT(*) AS rows_found
FROM products
WHERE brand_id = 9 OR brand_id = 1 AND list_price > 5000;
```

**Output:**

| rows_found |
|---|
| 135 |

This is read as: `brand_id = 9` **or** (`brand_id = 1` and price > 5000). That means all Trek products are included. If you meant "brand 9 or 1, and price > 5000," add parentheses:

```sql
SELECT COUNT(*) AS rows_found
FROM products
WHERE (brand_id = 9 OR brand_id = 1) AND list_price > 5000;
```

**Output:**

| rows_found |
|---|
| 13 |

### IN: match any value in a list

A shorter alternative to a long chain of `OR` conditions:

```sql
SELECT product_name, brand_id
FROM products
WHERE brand_id IN (1, 2, 9)
LIMIT 5;
```

**Output:**

| product_name | brand_id |
|---|---|
| Electra Townie Original 21D - 2016 | 1 |
| Electra Cruiser 1 (24-Inch) - 2016 | 1 |
| Electra Girl's Hawaii 1 (16-inch) - 2015/2016 | 1 |
| Electra Moto 1 - 2016 | 1 |
| Electra Townie Original 7D EQ - 2016 | 1 |

### BETWEEN: a range (including both ends)

```sql
SELECT product_name, list_price
FROM products
WHERE list_price BETWEEN 500 AND 600;
```

**Output:**

| product_name | list_price |
|---|---|
| Electra Townie Original 21D - 2016 | 549.99 |
| Electra Moto 1 - 2016 | 529.99 |
| Electra Townie Original 7D EQ - 2016 | 599.99 |
| Electra Townie Original 7D EQ - Women's - 2016 | 599.99 |
| Electra Townie Original 21D - 2016 | 549.99 |

_(The first 5 of 20 total rows are shown.)_

`BETWEEN 500 AND 600` means `>= 500 AND <= 600`. Both 500 and 600 are included. It also works with dates:

```sql
SELECT order_id, order_date
FROM orders
WHERE order_date BETWEEN '2018-01-01' AND '2018-01-31';
```

**Output:**

| order_id | order_date |
|---|---|
| 1324 | 2018-01-01 |
| 1325 | 2018-01-01 |
| 1326 | 2018-01-01 |
| 1327 | 2018-01-02 |
| 1328 | 2018-01-02 |

_(The first 5 of 52 total rows are shown.)_

### LIKE: text pattern

- `%` = any number of characters (zero or more).
- `_` = exactly one character.

```sql
SELECT product_name FROM products WHERE product_name LIKE 'Trek 8%';
```

**Output:**

| product_name |
|---|
| Trek 820 - 2016 |
| Trek 820 - 2018 |

```sql
SELECT product_name FROM products WHERE product_name LIKE '%Women%';
```

**Output:**

| product_name |
|---|
| Pure Cycles Western 3-Speed - Women's - 2015/2016 |
| Electra Townie Original 7D EQ - Women's - 2016 |
| Trek Silque SLR 7 Women's - 2017 |
| Trek Silque SLR 8 Women's - 2017 |
| Sun Bicycles Cruz 3 - Women's - 2017 |

_(The first 5 of 19 total rows are shown.)_

### IS NULL: missing value

Most values in `customers.phone` are NULL. NULL means "unknown/missing," so `= NULL` does not work.

```sql
SELECT COUNT(*) AS rows_found FROM customers WHERE phone = NULL;
```

**Output:**

| rows_found |
|---|
| 0 |

The result was 0, even though many rows have a NULL phone. Any comparison with NULL evaluates to "unknown," not "true." The correct syntax is `IS NULL`:

```sql
SELECT COUNT(*) AS without_phone FROM customers WHERE phone IS NULL;
```

**Output:**

| without_phone |
|---|
| 1267 |

The opposite is `IS NOT NULL`.

```sql
SELECT first_name, last_name, phone
FROM customers
WHERE phone IS NOT NULL
LIMIT 3;
```

**Output:**

| first_name | last_name | phone |
|---|---|---|
| Charolette | Rice | (916) 381-6003 |
| Latasha | Hays | (716) 986-3359 |
| Robby | Sykes | (516) 583-7761 |

---

## 4. Sorting and calculated columns

### ORDER BY

The default order is ascending (`ASC`). Use `DESC` for descending order.

The 5 most expensive products:

```sql
SELECT product_name, list_price
FROM products
ORDER BY list_price DESC
LIMIT 5;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek Domane SLR 9 Disc - 2018 | 11999.99 |
| Trek Domane SLR 8 Disc - 2018 | 7499.99 |
| Trek Domane SL Frameset - 2018 | 6499.99 |
| Trek Silque SLR 8 Women's - 2017 | 6499.99 |
| Trek Domane SL Frameset Women's - 2018 | 6499.99 |

With multiple columns, sorting uses the first column, then the next one to break ties:

```sql
SELECT brand_id, product_name, list_price
FROM products
ORDER BY brand_id ASC, list_price DESC
LIMIT 5;
```

**Output:**

| brand_id | product_name | list_price |
|---|---|---|
| 1 | Electra Townie Commute Go! - 2018 | 2999.99 |
| 1 | Electra Townie Commute Go! - 2018 | 2999.99 |
| 1 | Electra Townie Commute Go! Ladies' - 2018 | 2999.99 |
| 1 | Electra Townie Commute Go! Ladies' - 2018 | 2999.99 |
| 1 | Electra Loft Go! 8i - 2018 | 2799.99 |

You can also sort by an alias (shown in the calculated column below).

### Calculated columns

SQL can perform arithmetic (`+ - * /`) on columns. The final amount for each order line is `quantity x price`, minus the discount.

```sql
SELECT order_id, item_id, quantity, list_price, discount,
       quantity * list_price * (1 - discount) AS line_total
FROM order_items
ORDER BY line_total DESC
LIMIT 5;
```

**Output:**

| order_id | item_id | quantity | list_price | discount | line_total |
|---|---|---|---|---|---|
| 1364 | 2 | 2 | 11999.99 | 0.10 | 21599.9820 |
| 1541 | 1 | 2 | 11999.99 | 0.10 | 21599.9820 |
| 1506 | 3 | 2 | 7499.99 | 0.05 | 14249.9810 |
| 1463 | 1 | 2 | 6499.99 | 0.05 | 12349.9810 |
| 1297 | 3 | 2 | 6499.99 | 0.05 | 12349.9810 |

Here, `discount` of 0.20 means 20%, so `(1 - discount)` gives the portion remaining after the discount. We will call this `line_total` formula **net revenue** from here on.

### CASE WHEN: if-else logic

`CASE` checks conditions for each row and returns a label. Divide products into price bands:

```sql
SELECT product_name, list_price,
       CASE
           WHEN list_price < 500 THEN 'Budget'
           WHEN list_price < 2000 THEN 'Mid-range'
           ELSE 'Premium'
       END AS price_band
FROM products
ORDER BY list_price DESC
LIMIT 5;
```

**Output:**

| product_name | list_price | price_band |
|---|---|---|
| Trek Domane SLR 9 Disc - 2018 | 11999.99 | Premium |
| Trek Domane SLR 8 Disc - 2018 | 7499.99 | Premium |
| Trek Domane SL Frameset - 2018 | 6499.99 | Premium |
| Trek Silque SLR 8 Women's - 2017 | 6499.99 | Premium |
| Trek Domane SL Frameset Women's - 2018 | 6499.99 | Premium |

`CASE` checks from top to bottom and stops at the **first true condition**. So values below 500 have already been labeled `Budget` before reaching the `< 2000` condition. If no condition matches and there is no `ELSE`, the result is NULL.

Convert the `order_status` numbers to labels:

```sql
SELECT order_id, order_status,
       CASE order_status
           WHEN 1 THEN 'Pending'
           WHEN 2 THEN 'Processing'
           WHEN 3 THEN 'Rejected'
           WHEN 4 THEN 'Completed'
       END AS status_name
FROM orders
LIMIT 5;
```

**Output:**

| order_id | order_status | status_name |
|---|---|---|
| 1 | 4 | Completed |
| 2 | 4 | Completed |
| 3 | 4 | Completed |
| 4 | 4 | Completed |
| 5 | 4 | Completed |

---

## Query writing order

The order in which we write a query is:

```sql
SELECT ...
FROM ...
WHERE ...
ORDER BY ...
LIMIT ...;
```

(`GROUP BY` and `HAVING` are covered in file 03; they come after `WHERE` and before `ORDER BY`.)

## Common mistakes

- **Forgetting quotes around text** (`WHERE city = Brooklyn` is wrong) or using backticks instead of quotes.
- **Writing `= NULL`** (use `IS NULL`).
- **Using an alias in `WHERE`.** Aliases work in `ORDER BY`, but not in `WHERE`:

```sql
SELECT list_price * 2 AS double_price FROM products WHERE double_price > 1000;
```

**Error:**

```text
ERROR 1054 (42S22): Unknown column 'double_price' in 'where clause'
```

Reason: `WHERE` runs first, and the alias is created later by `SELECT`. Fix: write `WHERE list_price * 2 > 1000`.

## Practice

1. Show the names and prices of products from model year 2018 that cost less than 1000.
2. Show customers whose email ends with `@gmail.com` (only the first 5).
3. Show `order_id`, `order_date`, and `shipped_date` for orders where `shipped_date` is NULL, newest `order_date` first; show only 5.

## SQL Practice Questions Part-2

Q1. Write an SQL query to retrieve all records from the categories table.

Q2. Write an SQL query to retrieve all records from the products table where brand_id is equal to 4.

Q3. Write an SQL query to display the product names and their list prices for all products manufactured in the year 2017.

Q4. Write an SQL query to retrieve the 5 most expensive products based on their list prices.

Q5. Write an SQL query to retrieve all products whose list prices are between 3000 and 6000 (inclusive).

Q6. Write an SQL query to find the second least expensive product based on its list price.

Q7. Write an SQL query to retrieve all customers whose email addresses belong to the yahoo.com domain.

Q8. Write an SQL query to retrieve all customers from the state of California whose phone numbers are missing (NULL).

Q9. Write an SQL query to retrieve all customers from New York whose email addresses belong to the hotmail.com domain.

<details>
<summary>Answers</summary>

```sql
SELECT product_name, list_price
FROM products
WHERE model_year = 2018 AND list_price < 1000
LIMIT 5;
```

**Output:**

| product_name | list_price |
|---|---|
| Trek 820 - 2018 | 379.99 |
| Trek Marlin 5 - 2018 | 489.99 |
| Trek Marlin 6 - 2018 | 579.99 |
| Trek Marlin 7 - 2017/2018 | 749.99 |
| Trek X-Caliber 8 - 2018 | 999.99 |

```sql
SELECT first_name, last_name, email
FROM customers
WHERE email LIKE '%@gmail.com'
LIMIT 5;
```

**Output:**

| first_name | last_name | email |
|---|---|---|
| Pamelia | Newman | pamelia.newman@gmail.com |
| Linnie | Branch | linnie.branch@gmail.com |
| Tobie | Little | tobie.little@gmail.com |
| Adelle | Larsen | adelle.larsen@gmail.com |
| Regenia | Vaughan | regenia.vaughan@gmail.com |

```sql
SELECT order_id, order_date, shipped_date
FROM orders
WHERE shipped_date IS NULL
ORDER BY order_date DESC
LIMIT 5;
```

**Output:**

| order_id | order_date | shipped_date |
|---|---|---|
| 1615 | 2018-12-28 | NULL |
| 1614 | 2018-11-28 | NULL |
| 1613 | 2018-11-18 | NULL |
| 1612 | 2018-10-21 | NULL |
| 1611 | 2018-09-06 | NULL |

</details>
