# Topic 13: Creating and Changing Tables (CREATE, ALTER, Constraints)

So far, the tables were already created. In this file, we will design a table ourselves: choose a **data type** for each column and use **constraints** (rules) to prevent invalid data. The new table will be created in `bike_store` and will store **product reviews** (a customer's 1-to-5 rating for a product).

---

## Data types

When creating a column, you must specify what kind of data it will contain. Choosing the right type saves space and prevents invalid data.

| Type | When to use it | Bike store example |
|---|---|---|
| `INT` | Whole numbers | `customer_id`, `quantity` |
| `TINYINT` | Very small numbers (-128 to 127) | `active` (0/1), rating |
| `DECIMAL(p, s)` | Exact decimals, for money/prices | `list_price` = `DECIMAL(10,2)` |
| `VARCHAR(n)` | Text up to `n` characters | `product_name`, `email` |
| `TEXT` | Long text | Long comments |
| `DATE` | Date only (`YYYY-MM-DD`) | `order_date` |
| `DATETIME` / `TIMESTAMP` | Date + time | Exact order time |

`DECIMAL(10,2)` means 10 digits total, with 2 after the decimal point. Never use `FLOAT`/`DOUBLE` for money because they are approximate:

```sql
SELECT CAST(0.1 AS DOUBLE) + CAST(0.2 AS DOUBLE) = 0.3 AS double_is_exact,
       CAST(0.1 AS DECIMAL(3,1)) + CAST(0.2 AS DECIMAL(3,1)) = 0.3 AS decimal_is_exact;
```

**Output:**

| double_is_exact | decimal_is_exact |
|---|---|
| 0 | 1 |

With `DOUBLE`, `0.1 + 0.2` is not exactly `0.3` (0 = false). With `DECIMAL`, it is (1 = true).

---

## CREATE TABLE

Syntax: `CREATE TABLE name (column type constraints, ..., table-level constraints);`

Create this table:

```sql
DROP TABLE IF EXISTS review_likes, product_reviews, wishlist;

CREATE TABLE product_reviews (
    review_id    INT          NOT NULL AUTO_INCREMENT,
    product_id   INT          NOT NULL,
    customer_id  INT          NOT NULL,
    rating       TINYINT      NOT NULL,
    review_text  VARCHAR(500),
    review_date  DATE         NOT NULL DEFAULT (CURRENT_DATE),

    PRIMARY KEY (review_id),
    UNIQUE KEY uq_customer_product (customer_id, product_id),
    CONSTRAINT chk_rating CHECK (rating BETWEEN 1 AND 5),
    CONSTRAINT fk_review_product  FOREIGN KEY (product_id)  REFERENCES products (product_id),
    CONSTRAINT fk_review_customer FOREIGN KEY (customer_id) REFERENCES customers (customer_id)
);
```

## Constraints: one by one

| Constraint | What it does |
|---|---|
| `PRIMARY KEY` | Uniquely identifies each row. No NULLs or duplicates |
| `AUTO_INCREMENT` | MySQL supplies the next number (1, 2, 3...) |
| `NOT NULL` | The column cannot be empty |
| `UNIQUE` | A value (or column pair) cannot be duplicated. Unlike a primary key, NULL is allowed |
| `DEFAULT` | Uses this value if none is supplied |
| `CHECK` | Requires a value to satisfy a condition |
| `FOREIGN KEY` | The value must exist in another table |

In this table, a customer can review a product **only once** (`UNIQUE (customer_id, product_id)`), the rating must be between 1 and 5 (`CHECK`), and the review must refer to a real product/customer (`FOREIGN KEY`). The constraints are named (`chk_rating`, `fk_review_product`) so their names appear in errors and they are easier to remove later.

View the structure:

```sql
DESCRIBE product_reviews;
```

**Output:**

| Field | Type | Null | Key | Default | Extra |
|---|---|---|---|---|---|
| review_id | int | NO | PRI | NULL | auto_increment |
| product_id | int | NO | MUL | NULL |  |
| customer_id | int | NO | MUL | NULL |  |
| rating | tinyint | NO |  | NULL |  |
| review_text | varchar(500) | YES |  | NULL |  |
| review_date | date | NO |  | curdate() | DEFAULT_GENERATED |

### Insert data

We omit `review_id` and `review_date`; they are generated automatically:

```sql
INSERT INTO product_reviews (product_id, customer_id, rating, review_text)
VALUES (1, 1, 5, 'Very smooth ride, worth the money.'),
       (1, 2, 4, 'Good bike, the seat is a little stiff.'),
       (4, 1, 3, 'It is okay.');

SELECT review_id, product_id, customer_id, rating, review_text
FROM product_reviews;
```

**Output:**

| review_id | product_id | customer_id | rating | review_text |
|---|---|---|---|---|
| 1 | 1 | 1 | 5 | Very smooth ride, worth the money. |
| 2 | 1 | 2 | 4 | Good bike, the seat is a little stiff. |
| 3 | 4 | 1 | 3 | It is okay. |

---

## How constraints work

Let's insert invalid data. Each error identifies the rule that was broken.

**CHECK:** rating 9.

```sql
INSERT INTO product_reviews (product_id, customer_id, rating) VALUES (4, 2, 9);
```

**Error:**

```text
ERROR 3819 (HY000): Check constraint 'chk_rating' is violated.
```

**NOT NULL:** an empty rating.

```sql
INSERT INTO product_reviews (product_id, customer_id, rating) VALUES (4, 2, NULL);
```

**Error:**

```text
ERROR 1048 (23000): Column 'rating' cannot be null
```

**UNIQUE:** the same customer is reviewing the same product again.

```sql
INSERT INTO product_reviews (product_id, customer_id, rating) VALUES (1, 1, 2);
```

**Error:**

```text
ERROR 1062 (23000): Duplicate entry '1-1' for key 'product_reviews.uq_customer_product'
```

**FOREIGN KEY:** a product that does not exist (9999).

```sql
INSERT INTO product_reviews (product_id, customer_id, rating) VALUES (9999, 2, 4);
```

**Error:**

```text
ERROR 1452 (23000): Cannot add or update a child row: a foreign key constraint fails (`bike_store`.`product_reviews`, CONSTRAINT `fk_review_product` FOREIGN KEY (`product_id`) REFERENCES `products` (`product_id`))
```

### A foreign key also blocks deletes

Foreign keys check **deletes** as well as inserts. Customer 1 has orders, so deleting that customer is blocked:

```sql
DELETE FROM customers WHERE customer_id = 1;
```

**Error:**

```text
ERROR 1451 (23000): Cannot delete or update a parent row: a foreign key constraint fails (`bike_store`.`product_reviews`, CONSTRAINT `fk_review_customer` FOREIGN KEY (`customer_id`) REFERENCES `customers` (`customer_id`))
```

This error is expected. It protects the real data (nothing was deleted). You can change this behavior with `ON DELETE`:

| `ON DELETE ...` | What happens to child rows when the parent row is removed |
|---|---|
| `RESTRICT` / `NO ACTION` (default) | The delete is blocked (as in the error above) |
| `CASCADE` | Child rows are deleted automatically too |
| `SET NULL` | The child's FK column becomes NULL (the column must allow NULL) |

A small `CASCADE` demo. Create a table for review likes that are removed along with the review:

```sql
CREATE TABLE review_likes (
    like_id   INT NOT NULL AUTO_INCREMENT PRIMARY KEY,
    review_id INT NOT NULL,
    CONSTRAINT fk_like_review FOREIGN KEY (review_id)
        REFERENCES product_reviews (review_id) ON DELETE CASCADE
);

INSERT INTO review_likes (review_id) VALUES (1), (1), (2);

SELECT COUNT(*) AS likes_before FROM review_likes;

DELETE FROM product_reviews WHERE review_id = 1;

SELECT COUNT(*) AS likes_after FROM review_likes;
```

**Output 1:**

| likes_before |
|---|
| 3 |

**Output 2:**

| likes_after |
|---|
| 1 |

When review 1 was deleted, its two likes were deleted automatically; only the like for review 2 remained.

---

## ALTER TABLE: change an existing table

You can change a table's structure after it has been created. Common `ALTER TABLE` operations:

```sql
-- 1. Naya column jodna
ALTER TABLE product_reviews ADD COLUMN is_verified TINYINT(1) NOT NULL DEFAULT 0;

-- 2. Change the column type/size (from 500 to 1000 characters)
ALTER TABLE product_reviews MODIFY COLUMN review_text VARCHAR(1000);

-- 3. Rename a column
ALTER TABLE product_reviews RENAME COLUMN review_text TO comment_text;

-- 4. Add a constraint (helpful votes cannot be negative)
ALTER TABLE product_reviews ADD COLUMN helpful_votes INT NOT NULL DEFAULT 0;
ALTER TABLE product_reviews ADD CONSTRAINT chk_votes CHECK (helpful_votes >= 0);

-- 5. Column hatana
ALTER TABLE product_reviews DROP COLUMN is_verified;
```

Now see how the structure changed:

```sql
DESCRIBE product_reviews;
```

**Output:**

| Field | Type | Null | Key | Default | Extra |
|---|---|---|---|---|---|
| review_id | int | NO | PRI | NULL | auto_increment |
| product_id | int | NO | MUL | NULL |  |
| customer_id | int | NO | MUL | NULL |  |
| rating | tinyint | NO |  | NULL |  |
| comment_text | varchar(1000) | YES |  | NULL |  |
| review_date | date | NO |  | curdate() | DEFAULT_GENERATED |
| helpful_votes | int | NO |  | 0 |  |

Notice: `comment_text` now has a size of 1000, `is_verified` was removed, and `helpful_votes` was added.

Other useful operations:

| Command | What it does |
|---|---|
| `ALTER TABLE t DROP CONSTRAINT chk_votes;` | Remove a named constraint |
| `ALTER TABLE t DROP FOREIGN KEY fk_name;` | Remove a foreign key |
| `RENAME TABLE old TO new;` | Rename a table |
| `DROP TABLE t;` | Permanently remove a table (file 07 explains `DELETE`/`TRUNCATE`/`DROP`) |

`ALTER` cannot be rolled back (see the DDL explanation in file 07). Back up large tables before running it.

## A few table design rules

- Give **every table a primary key**. The simplest option is `INT AUTO_INCREMENT`.
- Store **money in `DECIMAL`**, never `FLOAT`.
- Use **`NOT NULL` for values that must never be empty**. Adding this constraint later is difficult if the data already contains NULLs.
- The **foreign key data type should match the parent's primary key** (`INT` to `INT`).
- Use clear column names: `customer_id`, `order_date`. Avoid names such as `col1` or `data`.

## Common mistakes

- **Not naming a constraint.** MySQL then assigns a name such as `product_reviews_chk_1`, which makes errors harder to understand.
- **Making `VARCHAR` too small.** Longer values may cause an error (or be truncated). Choose the size carefully.
- **Forgetting the existing details in `ALTER ... MODIFY`.** `MODIFY` replaces the entire column definition. If you omit an existing `NOT NULL`/`DEFAULT`, it is removed.
- **Trying to `DROP` a parent table referenced by a foreign key.** MySQL blocks it. First remove the child table (the one that points to it).

## Practice

Create a `wishlist` table to record which products customers add to their wishlists. Requirements:
- `wishlist_id` is an auto-number primary key.
- `customer_id` and `product_id` both reference tables in `bike_store` (foreign keys).
- A customer can add a product to their wishlist only once.
- `added_on` is a date that defaults to today's date.

Then insert 2 rows and verify that duplicates are rejected.

<details>
<summary>Answers</summary>

```sql
CREATE TABLE wishlist (
    wishlist_id INT  NOT NULL AUTO_INCREMENT,
    customer_id INT  NOT NULL,
    product_id  INT  NOT NULL,
    added_on    DATE NOT NULL DEFAULT (CURRENT_DATE),
    PRIMARY KEY (wishlist_id),
    UNIQUE KEY uq_wish (customer_id, product_id),
    CONSTRAINT fk_wish_customer FOREIGN KEY (customer_id) REFERENCES customers (customer_id),
    CONSTRAINT fk_wish_product  FOREIGN KEY (product_id)  REFERENCES products (product_id)
);

INSERT INTO wishlist (customer_id, product_id) VALUES (5, 10), (5, 11);

SELECT wishlist_id, customer_id, product_id FROM wishlist;
```

**Output:**

| wishlist_id | customer_id | product_id |
|---|---|---|
| 1 | 5 | 10 |
| 2 | 5 | 11 |

If you try to insert the duplicate (customer 5, product 10) again:

```sql
INSERT INTO wishlist (customer_id, product_id) VALUES (5, 10);
```

**Error:**

```text
ERROR 1062 (23000): Duplicate entry '5-10' for key 'wishlist.uq_wish'
```

</details>

We will clean up the remaining practice tables (`product_reviews`, `review_likes`, `wishlist`, and the `*_practice` tables from file 07) together at the end of file 11.
