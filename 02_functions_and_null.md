# Topic 5: Built-in Functions and NULL Handling

**A function** is a ready-made tool: you give it a value, and it returns a result. Functions work in `SELECT`, `WHERE`, and `ORDER BY`. This file covers three groups (string, number, and date), then NULL handling.

Functions work row by row. Functions that combine an entire table into one value (`SUM`, `COUNT`) are **aggregate functions**; they are covered in file 03.

---

## String functions

| Function | What it does |
|---|---|
| `CONCAT(a, b, ...)` | Joins text |
| `CONCAT_WS(sep, a, b, ...)` | Joins text with a separator |
| `UPPER(x)`, `LOWER(x)` | Converts to uppercase / lowercase |
| `LENGTH(x)` | Length in bytes (the same as characters for ordinary English text) |
| `SUBSTRING(x, start, len)` | Returns part of a string (start is 1-based) |
| `SUBSTRING_INDEX(x, sep, n)` | Returns the part before or after a separator |
| `REPLACE(x, old, new)` | Replaces text |
| `TRIM(x)` | Removes extra spaces from both ends |

Build each customer's full name:

```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name,
       UPPER(city) AS city_upper
FROM customers
LIMIT 5;
```

**Output:**

| full_name | city_upper |
|---|---|
| Debra Burks | ORCHARD PARK |
| Kasha Todd | CAMPBELL |
| Tameka Fisher | REDONDO BEACH |
| Daryl Spence | UNIONDALE |
| Charolette Rice | SACRAMENTO |

If any value passed to `CONCAT` is NULL, the entire result becomes NULL. This is a general rule for NULL, explained in detail below.

Extract the domain from an email. `SUBSTRING_INDEX(email, '@', -1)` splits at `@` and returns the **last** part (`-1` counts from the end):

```sql
SELECT email, SUBSTRING_INDEX(email, '@', -1) AS email_domain
FROM customers
LIMIT 5;
```

**Output:**

| email | email_domain |
|---|---|
| debra.burks@yahoo.com | yahoo.com |
| kasha.todd@yahoo.com | yahoo.com |
| tameka.fisher@aol.com | aol.com |
| daryl.spence@aol.com | aol.com |
| charolette.rice@msn.com | msn.com |

In file 03, we will use `GROUP BY` to count how often each domain appears.

An example using `TRIM`, `REPLACE`, and `SUBSTRING`:

```sql
SELECT TRIM('   Trek   ') AS trimmed,
       REPLACE('Road Bikes', 'Bikes', 'Bicycles') AS replaced,
       SUBSTRING('Mountain Bikes', 1, 8) AS first_8_chars;
```

**Output:**

| trimmed | replaced | first_8_chars |
|---|---|---|
| Trek | Road Bicycles | Mountain |

---

## Number functions

| Function | What it does |
|---|---|
| `ROUND(x, d)` | Rounds to `d` decimal places |
| `CEIL(x)` / `FLOOR(x)` | Rounds up / down to a whole number |
| `ABS(x)` | Returns the absolute value |
| `MOD(x, y)` or `x % y` | Returns the remainder |

Apply a 10% discount to the price and round it:

```sql
SELECT product_name, list_price,
       ROUND(list_price * 0.90, 2) AS price_after_10pct,
       CEIL(list_price) AS rounded_up,
       FLOOR(list_price) AS rounded_down
FROM products
LIMIT 5;
```

**Output:**

| product_name | list_price | price_after_10pct | rounded_up | rounded_down |
|---|---|---|---|---|
| Trek 820 - 2016 | 379.99 | 341.99 | 380 | 379 |
| Ritchey Timberwolf Frameset - 2016 | 749.99 | 674.99 | 750 | 749 |
| Surly Wednesday Frameset - 2016 | 999.99 | 899.99 | 1000 | 999 |
| Trek Fuel EX 8 29 - 2016 | 2899.99 | 2609.99 | 2900 | 2899 |
| Heller Shagamaw Frame - 2016 | 1320.99 | 1188.89 | 1321 | 1320 |

`ROUND(x, 0)` removes the decimal portion. `ROUND(x, -2)` rounds to the nearest hundred (for example, 1234 becomes 1200).

---

## Date functions

The dates in `orders` use the `DATE` type (`YYYY-MM-DD`), so you can extract many useful values from them.

| Function | What it does |
|---|---|
| `CURDATE()`, `NOW()` | Today's date / current date and time |
| `YEAR(d)`, `MONTH(d)`, `DAY(d)` | A part of a date |
| `DAYNAME(d)`, `MONTHNAME(d)` | The day / month name |
| `DATEDIFF(d1, d2)` | `d1 - d2` in days |
| `DATE_ADD(d, INTERVAL n unit)` | Adds days, months, or another interval to a date |
| `DATE_FORMAT(d, fmt)` | Formats a date as custom text |

```sql
SELECT order_id, order_date,
       YEAR(order_date) AS yr,
       MONTHNAME(order_date) AS month_name,
       DAYNAME(order_date) AS weekday
FROM orders
LIMIT 5;
```

**Output:**

| order_id | order_date | yr | month_name | weekday |
|---|---|---|---|---|
| 1 | 2016-01-01 | 2016 | January | Friday |
| 2 | 2016-01-01 | 2016 | January | Friday |
| 3 | 2016-01-02 | 2016 | January | Saturday |
| 4 | 2016-01-03 | 2016 | January | Sunday |
| 5 | 2016-01-03 | 2016 | January | Sunday |

Find how many days it took an order to ship (write `DATEDIFF(later, earlier)`):

```sql
SELECT order_id, order_date, shipped_date,
       DATEDIFF(shipped_date, order_date) AS days_to_ship
FROM orders
WHERE shipped_date IS NOT NULL
ORDER BY days_to_ship DESC
LIMIT 5;
```

**Output:**

| order_id | order_date | shipped_date | days_to_ship |
|---|---|---|---|
| 14 | 2016-01-09 | 2016-01-12 | 3 |
| 12 | 2016-01-06 | 2016-01-09 | 3 |
| 13 | 2016-01-08 | 2016-01-11 | 3 |
| 5 | 2016-01-03 | 2016-01-06 | 3 |
| 9 | 2016-01-05 | 2016-01-08 | 3 |

Add 7 days to the required date and apply a custom format:

```sql
SELECT order_date,
       DATE_ADD(order_date, INTERVAL 7 DAY) AS plus_7_days,
       DATE_FORMAT(order_date, '%d %b %Y') AS pretty_date
FROM orders
LIMIT 3;
```

**Output:**

| order_date | plus_7_days | pretty_date |
|---|---|---|
| 2016-01-01 | 2016-01-08 | 01 Jan 2016 |
| 2016-01-01 | 2016-01-08 | 01 Jan 2016 |
| 2016-01-02 | 2016-01-09 | 02 Jan 2016 |

Some common `DATE_FORMAT` codes: `%Y` (2018), `%m` (01), `%d` (05), `%b` (Jan), `%M` (January), `%W` (Monday).

> The output of `CURDATE()` depends on today's date on your computer, so no sample output is shown here. Try it yourself: `SELECT CURDATE(), NOW();`

---

## Handling NULL

**NULL** means "no value / unknown." It is not 0 and it is not empty text (`''`). It has three important rules:

1. Arithmetic or `CONCAT` with NULL produces a NULL result.
2. A comparison (`=`, `<>`, `>`) with NULL evaluates to "unknown," so the row is not returned by `WHERE`. Always check with `IS NULL` / `IS NOT NULL`.
3. Aggregate functions (`COUNT(col)`, `AVG`, `SUM`) **ignore** NULL values (see file 03).

```sql
SELECT 10 + NULL AS arithmetic,
       CONCAT('abc', NULL) AS joined,
       NULL = NULL AS null_equals_null,
       NULL IS NULL AS is_null_check;
```

**Output:**

| arithmetic | joined | null_equals_null | is_null_check |
|---|---|---|---|
| NULL | NULL | NULL | 1 |

### IFNULL and COALESCE: a default value in place of NULL

- `IFNULL(x, default)`: returns `default` when x is NULL.
- `COALESCE(a, b, c, ...)`: returns the **first non-NULL** value in the list. It is a more flexible version of `IFNULL` and accepts any number of arguments.

```sql
SELECT first_name, phone,
       IFNULL(phone, 'Not provided') AS phone_display
FROM customers
LIMIT 5;
```

**Output:**

| first_name | phone | phone_display |
|---|---|---|
| Debra | NULL | Not provided |
| Kasha | NULL | Not provided |
| Tameka | NULL | Not provided |
| Daryl | NULL | Not provided |
| Charolette | (916) 381-6003 | (916) 381-6003 |

Label NULL `shipped_date` values in pending or processing orders:

```sql
SELECT order_id, order_date, shipped_date,
       COALESCE(shipped_date, 'Not shipped yet') AS shipping_info
FROM orders
WHERE order_status IN (1, 2)
LIMIT 3;
```

**Output:**

| order_id | order_date | shipped_date | shipping_info |
|---|---|---|---|
| 1479 | 2018-04-01 | NULL | Not shipped yet |
| 1480 | 2018-04-01 | NULL | Not shipped yet |
| 1481 | 2018-04-01 | NULL | Not shipped yet |

Note: `shipped_date` is a DATE column and the fallback is text. MySQL converts both to text for the result, so this is fine for display but should not be used for later date calculations.

```sql
create database sql_practice;
use sql_practice;
CREATE TABLE customers (
    customer_id INT PRIMARY KEY,
    customer_name VARCHAR(100),
    phone_no VARCHAR(15),
    alternative_phone_no VARCHAR(15),
    email VARCHAR(100),
    alternative_email VARCHAR(100)
);
```

```sql
INSERT INTO customers
(customer_id, customer_name, phone_no, alternative_phone_no, email, alternative_email)
VALUES
(1, 'Amit Sharma', '9876543210', '9123456780', 'amit@gmail.com', 'amit.work@gmail.com'),
(2, 'Priya Verma', NULL, '9234567890', 'priya@gmail.com', NULL),
(3, 'Rahul Singh', '9345678901', NULL, NULL, 'rahul.work@gmail.com'),
(4, 'Neha Joshi', NULL, NULL, 'neha@gmail.com', NULL),
(5, 'Vikas Mehta', '9456789012', '9567890123', NULL, NULL),
(6, 'Pooja Rathore', NULL, '9678901234', NULL, 'pooja.work@gmail.com'),
(7, 'Rohit Kumar', '9789012345', NULL, 'rohit@gmail.com', NULL),
(8, 'Anjali Gupta', NULL, NULL, NULL, NULL),
(9, 'Suresh Patel', '9890123456', '9901234567', 'suresh@gmail.com', NULL),
(10, 'Kavita Jain', NULL, '9012345678', 'kavita@gmail.com', 'kavita.work@gmail.com'),
(11, 'Manish Yadav', '9123456701', NULL, NULL, NULL),
(12, 'Simran Kaur', NULL, NULL, 'simran@gmail.com', NULL),
(13, 'Deepak Saini', '9234567012', '9345678012', NULL, 'deepak.work@gmail.com'),
(14, 'Ritu Choudhary', NULL, '9456780123', NULL, NULL),
(15, 'Arjun Malhotra', '9567801234', NULL, 'arjun@gmail.com', 'arjun.work@gmail.com');
```

```sql
SELECT 
    *,
    COALESCE(phone_no,
            alternative_phone_no,
            email,
            alternative_email,
            'na'
	)
FROM
    customers;
```
  
### NULLIF: return NULL when two values are equal

`NULLIF(a, b)` means: return NULL if `a = b`; otherwise return `a`. It is most useful for avoiding division by zero:

```sql
SELECT 100 / NULLIF(0, 0) AS safe_division;
```

**Output:**

| safe_division |
|---|
| NULL |

Without `NULLIF`, MySQL also returns NULL for `100 / 0` (with a warning), but other databases (such as PostgreSQL) return an error. Using this pattern is a good habit.

---

## Common mistakes

- **Using `WHERE YEAR(order_date) = 2018`.** It works but can be slow on large tables. File 09 (indexes) explains why and shows a better approach.
- **Treating `LENGTH` as a character count.** For text such as Hindi or emoji, `LENGTH` counts bytes. Use `CHAR_LENGTH` to count characters.
- **Reversing the arguments to `DATEDIFF`.** `DATEDIFF(order_date, shipped_date)` returns a negative number.

## Practice

1. Show each staff member's `full_name` (first + last) and email domain.
2. Show the `order_id` values for orders shipped after `required_date` (late shipments), along with the number of days late. Show only 5.
3. Replace `customers.phone` with 'N/A' using `COALESCE`, and show `city` in uppercase (show only 5).

<details>
<summary>Answers</summary>

```sql
SELECT CONCAT(first_name, ' ', last_name) AS full_name,
       SUBSTRING_INDEX(email, '@', -1) AS domain
FROM staffs;
```

**Output:**

| full_name | domain |
|---|---|
| Fabiola Jackson | bikes.shop |
| Mireya Copeland | bikes.shop |
| Genna Serrano | bikes.shop |

_(The first 3 of 10 total rows are shown.)_

```sql
SELECT order_id, required_date, shipped_date,
       DATEDIFF(shipped_date, required_date) AS days_late
FROM orders
WHERE shipped_date > required_date
ORDER BY days_late DESC
LIMIT 5;
```

**Output:**

| order_id | required_date | shipped_date | days_late |
|---|---|---|---|
| 65 | 2016-02-09 | 2016-02-11 | 2 |
| 43 | 2016-01-28 | 2016-01-30 | 2 |
| 15 | 2016-01-10 | 2016-01-12 | 2 |
| 42 | 2016-01-28 | 2016-01-30 | 2 |
| 21 | 2016-01-16 | 2016-01-18 | 2 |

```sql
SELECT COALESCE(phone, 'N/A') AS phone, UPPER(city) AS city
FROM customers
LIMIT 5;
```

**Output:**

| phone | city |
|---|---|
| N/A | ORCHARD PARK |
| N/A | CAMPBELL |
| N/A | REDONDO BEACH |
| N/A | UNIONDALE |
| (916) 381-6003 | SACRAMENTO |

</details>
