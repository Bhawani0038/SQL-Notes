# Topic 15: Stored Functions, Procedures and Triggers

Until now you wrote and ran every query yourself. In this file we build **saved programs inside the database** that you can call again and again.

| Item | What it is | How it runs |
|---|---|---|
| **Function** | Returns a **single value** | Used inside a `SELECT` |
| **Stored procedure** | A saved program with parameters, can run many statements | `CALL procedure_name(...)` |
| **Trigger** | Runs **automatically** when a table gets an `INSERT`/`UPDATE`/`DELETE` | You never call it, it fires by itself |

This file stays at a basic level: understand the idea and build small, useful examples.

---

## DELIMITER: understand this first

Normally `;` ends a statement. But a function or procedure contains several `;` **inside** its body, and MySQL would stop at the first one and get confused. So we temporarily change the end marker:

```sql
DELIMITER $$
-- ... the procedure/function goes here, with its inner ; ...
END$$
DELIMITER ;
```

It reads as: "from now on `$$` ends a statement" → create the program → "go back to `;`". MySQL Workbench runs this directly.

---

## Functions

A function takes input and returns **one value**. So far we have repeated the net amount formula in query after query. Let us put it in a function:

```sql
DROP FUNCTION IF EXISTS fn_net_amount;

DELIMITER $$

CREATE FUNCTION fn_net_amount(p_qty INT, p_price DECIMAL(10,2), p_discount DECIMAL(4,2))
RETURNS DECIMAL(12,2)
DETERMINISTIC
BEGIN
    RETURN ROUND(p_qty * p_price * (1 - p_discount), 2);
END$$

DELIMITER ;
```

Notes:
- `p_qty`, `p_price`: parameters. The `p_` prefix is a habit that keeps them from clashing with column names.
- `RETURNS DECIMAL(12,2)`: the type of the returned value.
- `DETERMINISTIC`: the same input always gives the same output (true here). It tells MySQL the function is predictable.

Now use it like any built-in function:

```sql
SELECT order_id, item_id, quantity, discount,
       fn_net_amount(quantity, list_price, discount) AS net_amount
FROM order_items
LIMIT 3;
```

**Output:**

| order_id | item_id | quantity | discount | net_amount |
|---|---|---|---|---|
| 1 | 1 | 1 | 0.20 | 479.99 |
| 1 | 2 | 2 | 0.07 | 3347.98 |
| 1 | 3 | 2 | 0.05 | 2943.10 |

A function with `IF`. Let us turn the price bands from the `CASE` in file 01 into a function:

```sql
DROP FUNCTION IF EXISTS fn_price_band;

DELIMITER $$

CREATE FUNCTION fn_price_band(p_price DECIMAL(10,2))
RETURNS VARCHAR(10)
DETERMINISTIC
BEGIN
    IF p_price < 500 THEN
        RETURN 'Budget';
    ELSEIF p_price < 2000 THEN
        RETURN 'Mid-range';
    ELSE
        RETURN 'Premium';
    END IF;
END$$

DELIMITER ;
```

```sql
SELECT fn_price_band(list_price) AS price_band, COUNT(*) AS products
FROM products
GROUP BY price_band
ORDER BY products DESC;
```

**Output:**

| price_band | products |
|---|---|
| Mid-range | 126 |
| Budget | 108 |
| Premium | 87 |

---

## Stored Procedures

A **procedure** is more powerful than a function: it can run many statements, return result sets, and change data. It has three kinds of parameters:

| Type | Meaning |
|---|---|
| `IN` | Goes in (input). The default |
| `OUT` | Comes out (output value) |
| `INOUT` | Both |

### Procedure 1: a customer's orders

```sql
DROP PROCEDURE IF EXISTS sp_customer_orders;

DELIMITER $$

CREATE PROCEDURE sp_customer_orders(IN p_customer_id INT)
BEGIN
    SELECT o.order_id,
           o.order_date,
           COUNT(*) AS items,
           ROUND(SUM(fn_net_amount(oi.quantity, oi.list_price, oi.discount)), 2) AS order_total
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.customer_id = p_customer_id
    GROUP BY o.order_id, o.order_date
    ORDER BY o.order_date;
END$$

DELIMITER ;
```

You run a procedure with `CALL`:

```sql
CALL sp_customer_orders(1);
```

**Output:**

| order_id | order_date | items | order_total |
|---|---|---|---|
| 599 | 2016-12-09 | 4 | 9199.15 |
| 1555 | 2018-04-18 | 5 | 13431.05 |
| 1613 | 2018-11-18 | 2 | 5257.97 |

For any customer you now just write `CALL sp_customer_orders(<id>);` with no need to retype the query. (The procedure also used our own function `fn_net_amount` inside.)

### Procedure 2: an OUT parameter

Return a store's Completed revenue through a variable:

```sql
DROP PROCEDURE IF EXISTS sp_store_revenue;

DELIMITER $$

CREATE PROCEDURE sp_store_revenue(IN p_store_id INT, OUT p_revenue DECIMAL(14,2))
BEGIN
    SELECT COALESCE(ROUND(SUM(oi.quantity * oi.list_price * (1 - oi.discount)), 2), 0)
    INTO p_revenue
    FROM orders AS o
    JOIN order_items AS oi ON o.order_id = oi.order_id
    WHERE o.store_id = p_store_id
      AND o.order_status = 4;
END$$

DELIMITER ;
```

`SELECT ... INTO variable` stores the result in a variable. `@rev` is a **user variable** (it starts with `@`) that lives for your session:

```sql
CALL sp_store_revenue(1, @rev);
SELECT @rev AS store_1_revenue;
```

**Output 1:**

| store_1_revenue |
|---|
| 1255491.65 |

### Procedure 3: changing data, with validation

For procedures that change data we use practice copies, so the real tables stay safe. These copies are also used by the triggers below:

```sql
DROP TABLE IF EXISTS price_log, sales_demo, stocks_demo, products_demo;

CREATE TABLE stocks_demo   LIKE stocks;
INSERT INTO stocks_demo   SELECT * FROM stocks;

CREATE TABLE products_demo LIKE products;
INSERT INTO products_demo SELECT * FROM products;

CREATE TABLE price_log (
    log_id     INT           NOT NULL AUTO_INCREMENT PRIMARY KEY,
    product_id INT           NOT NULL,
    old_price  DECIMAL(10,2),
    new_price  DECIMAL(10,2),
    changed_at DATETIME      NOT NULL DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE sales_demo (
    sale_id    INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    store_id   INT NOT NULL,
    product_id INT NOT NULL,
    quantity   INT NOT NULL
);
```

A procedure that adds stock. If the quantity is zero or negative, it raises its own error using `SIGNAL`:

```sql
DROP PROCEDURE IF EXISTS sp_restock;

DELIMITER $$

CREATE PROCEDURE sp_restock(IN p_store INT, IN p_product INT, IN p_qty INT)
BEGIN
    IF p_qty <= 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Restock quantity must be positive';
    END IF;

    UPDATE stocks_demo
    SET quantity = quantity + p_qty
    WHERE store_id = p_store AND product_id = p_product;

    SELECT store_id, product_id, quantity
    FROM stocks_demo
    WHERE store_id = p_store AND product_id = p_product;
END$$

DELIMITER ;
```

```sql
CALL sp_restock(1, 1, 5);
```

**Output:**

| store_id | product_id | quantity |
|---|---|---|
| 1 | 1 | 32 |

Store 1 had 27 units of product 1. After adding 5, it has 32. With bad input:

```sql
CALL sp_restock(1, 1, -3);
```

**Error:**

```text
ERROR 1644 (45000): Restock quantity must be positive
```

Inside a procedure you can also use variables (`DECLARE x INT; SET x = 5;`), `IF/ELSEIF/ELSE`, `CASE`, loops (`WHILE`, `REPEAT`, `LOOP`), and error handling (`DECLARE ... HANDLER`). That is enough for the basics.

### Function vs procedure

| | Function | Procedure |
|---|---|---|
| Result | Exactly **one value** | Anything: a result set, OUT values, nothing |
| How you call it | Inside a query (`SELECT fn(...)`) | `CALL` |
| Changes tables? | Usually not used for that | Yes |

---

## Triggers

A **trigger** is a program that runs **by itself** when something happens to a table. Two things define it:

- **Timing:** `BEFORE` (before the change) or `AFTER` (after it).
- **Event:** `INSERT`, `UPDATE` or `DELETE`.

Inside a trigger you get two special row references:

| Keyword | Meaning | Available in |
|---|---|---|
| `NEW.column` | The new value | `INSERT`, `UPDATE` |
| `OLD.column` | The old value | `UPDATE`, `DELETE` |

### Trigger 1: a price change audit log

Whenever the **price** of a product in `products_demo` changes, write the old and new price into `price_log`:

```sql
DROP TRIGGER IF EXISTS trg_products_price_log;

DELIMITER $$

CREATE TRIGGER trg_products_price_log
AFTER UPDATE ON products_demo
FOR EACH ROW
BEGIN
    IF NEW.list_price <> OLD.list_price THEN
        INSERT INTO price_log (product_id, old_price, new_price)
        VALUES (OLD.product_id, OLD.list_price, NEW.list_price);
    END IF;
END$$

DELIMITER ;
```

One update changes the price, another changes only `model_year`. Only the first should appear in the log:

```sql
UPDATE products_demo SET list_price = 399.99 WHERE product_id = 1;
UPDATE products_demo SET model_year = 2016 WHERE product_id = 2;

SELECT product_id, old_price, new_price FROM price_log;
```

**Output:**

| product_id | old_price | new_price |
|---|---|---|
| 1 | 379.99 | 399.99 |

`FOR EACH ROW` means the trigger runs once for every changed row. Because of the `IF`, nothing was logged when the price did not change.

### Trigger 2: reduce stock when a sale is recorded

When a new sale is inserted into `sales_demo`, the stock in `stocks_demo` should drop automatically:

```sql
DROP TRIGGER IF EXISTS trg_sales_reduce_stock;

DELIMITER $$

CREATE TRIGGER trg_sales_reduce_stock
AFTER INSERT ON sales_demo
FOR EACH ROW
BEGIN
    UPDATE stocks_demo
    SET quantity = quantity - NEW.quantity
    WHERE store_id = NEW.store_id
      AND product_id = NEW.product_id;
END$$

DELIMITER ;
```

Record a sale of 3 units of product 1 at store 2, and compare the stock before and after:

```sql
SELECT quantity AS stock_before FROM stocks_demo WHERE store_id = 2 AND product_id = 1;

INSERT INTO sales_demo (store_id, product_id, quantity) VALUES (2, 1, 3);

SELECT quantity AS stock_after FROM stocks_demo WHERE store_id = 2 AND product_id = 1;
```

**Output 1:**

| stock_before |
|---|
| 14 |

**Output 2:**

| stock_after |
|---|
| 11 |

We only inserted into `sales_demo`. The trigger reduced the stock on its own.

### Trigger 3: validation with a BEFORE trigger

A `BEFORE INSERT` trigger can stop bad data **before** it is stored:

```sql
DROP TRIGGER IF EXISTS trg_sales_check_qty;

DELIMITER $$

CREATE TRIGGER trg_sales_check_qty
BEFORE INSERT ON sales_demo
FOR EACH ROW
BEGIN
    IF NEW.quantity <= 0 THEN
        SIGNAL SQLSTATE '45000'
            SET MESSAGE_TEXT = 'Sale quantity must be greater than 0';
    END IF;
END$$

DELIMITER ;
```

```sql
INSERT INTO sales_demo (store_id, product_id, quantity) VALUES (2, 1, 0);
```

**Error:**

```text
ERROR 1644 (45000): Sale quantity must be greater than 0
```

The row was never inserted, and the `AFTER INSERT` stock trigger did not run either.

### Viewing routines and triggers

```sql
SHOW PROCEDURE STATUS WHERE Db = 'bike_store';
```

**Output:**

| Name | Type |
|---|---|
| sp_customer_orders | PROCEDURE |
| sp_restock | PROCEDURE |
| sp_store_revenue | PROCEDURE |

```sql
SHOW TRIGGERS WHERE `Table` LIKE '%demo';
```

**Output:**

| Trigger | Event | Timing | Table |
|---|---|---|---|
| trg_products_price_log | UPDATE | AFTER | products_demo |
| trg_sales_check_qty | INSERT | BEFORE | sales_demo |
| trg_sales_reduce_stock | INSERT | AFTER | sales_demo |

To remove them: `DROP PROCEDURE name;`, `DROP FUNCTION name;`, `DROP TRIGGER name;`. We clean all of these up at the end of file 11.

## When to use what

- **Functions:** a formula repeated in many places (net amount, price band).
- **Procedures:** several steps that always run together, or when an application should simply `CALL` one thing.
- **Triggers:** audit logs and automatic validation, as above. **Be careful:** a trigger is hidden logic. If a query does one thing but the data changes in another way, think of triggers first. Too many triggers make debugging and performance worse. When there is a lot of logic, keeping it in application code is often better.

## Common mistakes

- **Forgetting `DELIMITER`.** The procedure is never created and you get a syntax error.
- **Naming a parameter like a column.** `WHERE store_id = store_id` is always true. That is why we use the `p_` prefix.
- **Changing the same table the trigger is on** (it can loop forever, and MySQL gives an error).
- **Mixing up `BEFORE` and `AFTER`.** Validation goes in `BEFORE`. Work that follows the change (logs, stock) goes in `AFTER`.

## Practice

1. Create a function `fn_full_name(first, last)` that returns "First Last", and use it on 3 staff members.
2. Create a procedure `sp_top_products(IN p_n INT)` that shows the top `p_n` best-selling products (name and units). Run `CALL sp_top_products(3);`.

<details>
<summary>Answers</summary>

```sql
DROP FUNCTION IF EXISTS fn_full_name;

DELIMITER $$

CREATE FUNCTION fn_full_name(p_first VARCHAR(50), p_last VARCHAR(50))
RETURNS VARCHAR(101)
DETERMINISTIC
BEGIN
    RETURN CONCAT(p_first, ' ', p_last);
END$$

DELIMITER ;
```

```sql
SELECT staff_id, fn_full_name(first_name, last_name) AS full_name
FROM staffs
LIMIT 3;
```

**Output:**

| staff_id | full_name |
|---|---|
| 1 | Fabiola Jackson |
| 2 | Mireya Copeland |
| 3 | Genna Serrano |

```sql
DROP PROCEDURE IF EXISTS sp_top_products;

DELIMITER $$

CREATE PROCEDURE sp_top_products(IN p_n INT)
BEGIN
    SELECT p.product_name, SUM(oi.quantity) AS units_sold
    FROM order_items AS oi
    JOIN products AS p ON oi.product_id = p.product_id
    GROUP BY p.product_id, p.product_name
    ORDER BY units_sold DESC
    LIMIT p_n;
END$$

DELIMITER ;
```

```sql
CALL sp_top_products(3);
```

**Output:**

| product_name | units_sold |
|---|---|
| Surly Ice Cream Truck Frameset - 2016 | 167 |
| Electra Cruiser 1 (24-Inch) - 2016 | 157 |
| Electra Townie Original 7D EQ - 2016 | 156 |

</details>
