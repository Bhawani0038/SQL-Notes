# Topic 12: INSERT, UPDATE, DELETE, and Transactions

So far, we have only **read** data (`SELECT`). In this file, we will learn to **change** data. These commands (`INSERT`, `UPDATE`, `DELETE`) modify data for real, so first we will set up a safety measure.

## Setup: practice copies

We will not modify the original tables in `bike_store`. We will make **copies** and run all experiments on those copies. This keeps the original data safe and preserves the results of queries in the other files.

```sql
DROP TABLE IF EXISTS customers_practice, products_practice, stocks_practice;

CREATE TABLE customers_practice LIKE customers;
INSERT INTO customers_practice SELECT * FROM customers;

CREATE TABLE products_practice LIKE products;
INSERT INTO products_practice SELECT * FROM products;

CREATE TABLE stocks_practice LIKE stocks;
INSERT INTO stocks_practice SELECT * FROM stocks;
```

- `CREATE TABLE x LIKE y`: creates an **empty** table like `y` (columns, primary key, and indexes are copied; foreign keys are not).
- `INSERT INTO x SELECT * FROM y`: copies all rows from `y` into `x`.
- `DROP TABLE IF EXISTS`: removes the table if it exists; otherwise it does nothing. Rerunning the script does not cause an error. File 08 covers `CREATE`/`DROP` in detail.

```sql
SELECT (SELECT COUNT(*) FROM customers_practice) AS customers_practice,
       (SELECT COUNT(*) FROM products_practice)  AS products_practice,
       (SELECT COUNT(*) FROM stocks_practice)    AS stocks_practice;
```

**Output:**

| customers_practice | products_practice | stocks_practice |
|---|---|---|
| 1445 | 321 | 939 |

---

## INSERT: add a new row

```sql
INSERT INTO customers_practice
    (customer_id, first_name, last_name, phone, email, street, city, state, zip_code)
VALUES
    (1446, 'Aarav', 'Sharma', NULL, 'aarav.sharma@example.com', '12 Park Street', 'Brooklyn', 'NY', 11201);
```

Always specify a column list (`INSERT INTO table (col1, col2) VALUES (...)`). Without it, values must follow the table's column order, and the query can break if the table changes.

Verify:

```sql
SELECT customer_id, first_name, last_name, phone, city
FROM customers_practice
WHERE customer_id = 1446;
```

**Output:**

| customer_id | first_name | last_name | phone | city |
|---|---|---|---|---|
| 1446 | Aarav | Sharma | NULL | Brooklyn |

### Multiple rows at once

```sql
INSERT INTO customers_practice
    (customer_id, first_name, last_name, email, city, state)
VALUES
    (1447, 'Diya', 'Verma', 'diya.verma@example.com', 'Austin', 'TX'),
    (1448, 'Kabir', 'Singh', 'kabir.singh@example.com', 'San Jose', 'CA');
```

Note: `phone`, `street`, and `zip_code` were omitted here. Columns not in the list get NULL (or the default specified for the table).

```sql
SELECT customer_id, first_name, phone, street, city
FROM customers_practice
WHERE customer_id > 1445;
```

**Output:**

| customer_id | first_name | phone | street | city |
|---|---|---|---|---|
| 1446 | Aarav | NULL | 12 Park Street | Brooklyn |
| 1447 | Diya | NULL | NULL | Austin |
| 1448 | Kabir | NULL | NULL | San Jose |

### A primary key cannot be duplicated

```sql
INSERT INTO customers_practice (customer_id, first_name, last_name, email)
VALUES (1446, 'Test', 'Duplicate', 'test@example.com');
```

**Error:**

```text
ERROR 1062 (23000): Duplicate entry '1446' for key 'customers_practice.PRIMARY'
```

### INSERT ... SELECT: insert a query result into a table

Create a new table containing only products priced above 5000:

```sql
DROP TABLE IF EXISTS expensive_products_practice;
CREATE TABLE expensive_products_practice LIKE products;

INSERT INTO expensive_products_practice
SELECT * FROM products WHERE list_price > 5000;

SELECT COUNT(*) AS rows_copied FROM expensive_products_practice;
```

**Output:**

| rows_copied |
|---|
| 13 |

### Upsert: "update if it exists, otherwise insert"

The primary key of `stocks_practice` is the pair (`store_id`, `product_id`). If a row with that pair already exists, a new `INSERT` returns an error. `ON DUPLICATE KEY UPDATE` updates it instead:

```sql
SELECT store_id, product_id, quantity FROM stocks_practice WHERE store_id = 1 AND product_id = 1;

INSERT INTO stocks_practice (store_id, product_id, quantity)
VALUES (1, 1, 10) AS new_row
ON DUPLICATE KEY UPDATE quantity = stocks_practice.quantity + new_row.quantity;

SELECT store_id, product_id, quantity FROM stocks_practice WHERE store_id = 1 AND product_id = 1;
```

**Output 1:**

| store_id | product_id | quantity |
|---|---|---|
| 1 | 1 | 27 |

**Output 2:**

| store_id | product_id | quantity |
|---|---|---|
| 1 | 1 | 37 |

The quantity was 27 before and 37 after the upsert. `VALUES(quantity)` used to be the standard syntax, but it has been deprecated since MySQL 8.0.20, so this uses the `AS new_row` syntax.

---

## UPDATE: change existing rows

`UPDATE table SET column = value WHERE condition`.

**Always include `WHERE`.** Without `WHERE`, **all rows** are changed.

```sql
UPDATE customers_practice
SET phone = '(718) 555-0101'
WHERE customer_id = 1446;
```

Update multiple columns at once:

```sql
UPDATE customers_practice
SET city = 'Queens', zip_code = 11375
WHERE customer_id = 1446;

SELECT customer_id, phone, city, zip_code
FROM customers_practice
WHERE customer_id = 1446;
```

**Output:**

| customer_id | phone | city | zip_code |
|---|---|---|---|
| 1446 | (718) 555-0101 | Queens | 11375 |

### Update using an expression

Increase the price of all products from brand 9 (Trek) by 10%. Show the prices of 3 products before and after:

```sql
SELECT product_id, list_price FROM products_practice WHERE product_id IN (1, 4, 7);

UPDATE products_practice
SET list_price = ROUND(list_price * 1.10, 2)
WHERE brand_id = 9;

SELECT product_id, list_price FROM products_practice WHERE product_id IN (1, 4, 7);
```

**Output 1:**

| product_id | list_price |
|---|---|
| 1 | 379.99 |
| 4 | 2899.99 |
| 7 | 3999.99 |

**Output 2:**

| product_id | list_price |
|---|---|
| 1 | 417.99 |
| 4 | 3189.99 |
| 7 | 4399.99 |

### UPDATE with a JOIN

The brand name is in `brands`, not in `products_practice`. A join is needed to apply a 5% discount to the Surly brand:

```sql
UPDATE products_practice AS p
JOIN brands AS b ON p.brand_id = b.brand_id
SET p.list_price = ROUND(p.list_price * 0.95, 2)
WHERE b.brand_name = 'Surly';

SELECT p.product_name, p.list_price AS new_price, orig.list_price AS old_price
FROM products_practice AS p
JOIN products AS orig ON p.product_id = orig.product_id
JOIN brands AS b ON p.brand_id = b.brand_id
WHERE b.brand_name = 'Surly'
LIMIT 3;
```

**Output:**

| product_name | new_price | old_price |
|---|---|---|
| Surly Wednesday Frameset - 2016 | 949.99 | 999.99 |
| Surly Ice Cream Truck Frameset - 2016 | 446.49 | 469.99 |
| Surly Straggler - 2016 | 1471.55 | 1549.00 |

(`old_price` is shown by comparing with the original `products` table, which was not changed during practice.)

### Safe Update Mode (Workbench error 1175)

In MySQL Workbench, **Safe Updates** is ON by default. In this mode, Workbench blocks an `UPDATE`/`DELETE` with no `WHERE`, or one whose `WHERE` does not use a key column (such as a primary key):

`Error Code: 1175. You are using safe update mode and you tried to update a table without a WHERE that uses a KEY column.`

There are two options: use a primary key (or indexed column) in `WHERE`, or turn off the mode for the session (with care):

```sql
SET SQL_SAFE_UPDATES = 0;
```

---

## DELETE: remove rows

`DELETE FROM table WHERE condition`. Again, **forgetting `WHERE` is the biggest mistake**: it empties the entire table.

Delete the 3 test customers we added. Count the rows before and after:

```sql
SELECT COUNT(*) AS customers_before FROM customers_practice;

DELETE FROM customers_practice WHERE customer_id > 1445;

SELECT COUNT(*) AS customers_after FROM customers_practice;
```

**Output 1:**

| customers_before |
|---|
| 1448 |

**Output 2:**

| customers_after |
|---|
| 1445 |

**Effect of foreign keys:** if rows in another table point to a row (such as a customer's orders), you cannot delete that customer from the original `bike_store`. File 08 explains this. The practice copies do not have foreign keys, so this restriction does not apply here.

### DELETE vs TRUNCATE vs DROP

| Command | What it does | `WHERE` | Can it be rolled back? |
|---|---|---|---|
| `DELETE FROM t WHERE ...` | Removes selected rows | Yes | Yes, inside a transaction |
| `TRUNCATE TABLE t` | Removes **all rows** at once but keeps the table | No | No (immediately permanent) |
| `DROP TABLE t` | Removes the table itself (structure + data) | No | No |

```sql
TRUNCATE TABLE expensive_products_practice;
SELECT COUNT(*) AS rows_left FROM expensive_products_practice;
```

**Output:**

| rows_left |
|---|
| 0 |

```sql
DROP TABLE expensive_products_practice;
```

---

## Transactions: all or nothing

A **transaction** is a group of statements that either all complete **together** or none do. Suppose you need to move a product's stock from Store 1 to Store 2. Two updates are required (subtract from one and add to the other). If something fails in between, the data will be incomplete. A transaction protects against this.

Basic commands:

| Command | Meaning |
|---|---|
| `START TRANSACTION` | Start a transaction |
| `COMMIT` | Save the changes (they become visible to everyone) |
| `ROLLBACK` | Undo all changes made since it started |

By default, MySQL has **autocommit ON**: each standalone statement is immediately committed. `START TRANSACTION` suspends autocommit until `COMMIT` or `ROLLBACK`.

### ROLLBACK: recover from a mistake

Suppose you accidentally set all prices to zero by forgetting `WHERE`:

```sql
START TRANSACTION;

UPDATE products_practice SET list_price = 0;   -- galti: WHERE bhool gaye

SELECT MIN(list_price) AS min_price, MAX(list_price) AS max_price FROM products_practice;

ROLLBACK;

SELECT MIN(list_price) AS min_price, MAX(list_price) AS max_price FROM products_practice;
```

**Output 1:**

| min_price | max_price |
|---|---|
| 0.00 | 0.00 |

**Output 2:**

| min_price | max_price |
|---|---|
| 89.99 | 13199.99 |

The first result shows all zeros (inside the transaction); after `ROLLBACK`, prices return to normal.

### COMMIT: stock transfer

Move 10 units of product 1 from Store 1 to Store 2. Put both updates in one transaction:

```sql
START TRANSACTION;

UPDATE stocks_practice SET quantity = quantity - 10 WHERE store_id = 1 AND product_id = 1;
UPDATE stocks_practice SET quantity = quantity + 10 WHERE store_id = 2 AND product_id = 1;

COMMIT;

SELECT store_id, product_id, quantity FROM stocks_practice WHERE product_id = 1 ORDER BY store_id;
```

**Output:**

| store_id | product_id | quantity |
|---|---|---|
| 1 | 1 | 27 |
| 2 | 1 | 24 |
| 3 | 1 | 14 |

(Store 1's quantity had increased to 37 in the earlier upsert, so it is now 27. Store 2's quantity increased from 14 to 24.)

One more thing: data definition (DDL) commands such as `ALTER`, `CREATE`, `DROP`, and `TRUNCATE` **commit the transaction automatically** and cannot be rolled back.

## Common mistakes

- **`UPDATE`/`DELETE` without `WHERE`.** First run a `SELECT` with the same `WHERE` to see which rows match, then change the `SELECT` to `UPDATE`/`DELETE`.
- **Experimenting directly on the original table.** Use a practice copy or a transaction (`START TRANSACTION` ... `ROLLBACK`).
- **Forgetting `COMMIT`.** If you close the connection with a transaction open, its changes are discarded.
- **Treating `TRUNCATE` like `DELETE`.** It does not accept `WHERE` and cannot be rolled back.

## Practice

(Use the practice copies only.)

1. Add a new customer (`customer_id` 1450) to `customers_practice` and change their city to 'Dallas'. Verify it.
2. Delete rows in `stocks_practice` where quantity is 0. How many were there first?
3. Increase the prices of products in the Road Bikes category in `products_practice` by 5% (using a join), then show the new `MIN` and `MAX` prices.

<details>
<summary>Answers</summary>

```sql
INSERT INTO customers_practice (customer_id, first_name, last_name, email, city, state)
VALUES (1450, 'Neha', 'Gupta', 'neha.gupta@example.com', 'Houston', 'TX');

UPDATE customers_practice SET city = 'Dallas' WHERE customer_id = 1450;

SELECT customer_id, first_name, city FROM customers_practice WHERE customer_id = 1450;
```

**Output:**

| customer_id | first_name | city |
|---|---|---|
| 1450 | Neha | Dallas |

```sql
SELECT COUNT(*) AS zero_stock_rows FROM stocks_practice WHERE quantity = 0;

DELETE FROM stocks_practice WHERE quantity = 0;

SELECT COUNT(*) AS zero_stock_rows_after FROM stocks_practice WHERE quantity = 0;
```

**Output 1:**

| zero_stock_rows |
|---|
| 25 |

**Output 2:**

| zero_stock_rows_after |
|---|
| 0 |

```sql
UPDATE products_practice AS p
JOIN categories AS c ON p.category_id = c.category_id
SET p.list_price = ROUND(p.list_price * 1.05, 2)
WHERE c.category_name = 'Road Bikes';

SELECT MIN(p.list_price) AS min_price, MAX(p.list_price) AS max_price
FROM products_practice AS p
JOIN categories AS c ON p.category_id = c.category_id
WHERE c.category_name = 'Road Bikes';
```

**Output:**

| min_price | max_price |
|---|---|
| 748.11 | 13859.99 |

</details>
