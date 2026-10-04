# Procedure-SQL 

**Source:** [techTFQ - Procedure Tutorial in SQL | SQL Stored Procedure](https://www.youtube.com/watch?v=yLR1w4tZ36I)

---

## 1. Overview of SQL Stored Procedures

### **What is a Stored Procedure?**
A **stored procedure** is a named, reusable block of SQL code stored directly inside the database system. It combines standard SQL commands (SELECT, INSERT, UPDATE, DELETE, DDL, TCL) with procedural programming constructs like:
* Variables
* Conditional Logic (`IF-ELSE`, `CASE`)
* Loops (`WHILE`, `FOR`, `LOOP`)
* Cursors & Collections
* Exception Handling

### **Why Use Stored Procedures?**
* **Complex Business Logic:** Executes multi-step business transactions and data validations on the server side.
* **Performance:** Reduces network traffic by bundling multiple SQL commands into a single database execution.
* **Security & Control:** Restricts direct table access by exposing controlled procedures with execution permissions.
* **Code Reusability:** Prevents code duplication across applications interacting with the database.

---

## 2. Basic Syntax Comparison Across 4 Major RDBMS

### **PostgreSQL Syntax**
```sql
CREATE OR REPLACE PROCEDURE pr_name(p_param VARCHAR)
LANGUAGE plpgsql
AS $$
DECLARE
    v_var INT;
BEGIN
    -- Business logic here
    RAISE NOTICE 'Execution complete';
END;
$$;
```
* **Key Details:**
  * Uses `LANGUAGE plpgsql` to specify the procedural language extension.
  * Uses `$$` dollar-quoting to enclose the body without escaping single quotes.
  * Explicit `DECLARE` block for local variables.
  * `RAISE NOTICE` outputs messages to the console.
  * Executed using: `CALL pr_name('value');`

---

### **Oracle Database Syntax**
```sql
CREATE OR REPLACE PROCEDURE pr_name(p_param IN VARCHAR2)
AS
    v_var NUMBER;
BEGIN
    -- Business logic here
    DBMS_OUTPUT.PUT_LINE('Execution complete');
END;
/
```
* **Key Details:**
  * No `LANGUAGE` or `$$` required.
  * Variables are declared between `AS` and `BEGIN` (no explicit `DECLARE` keyword needed).
  * `DBMS_OUTPUT.PUT_LINE()` outputs console messages (requires enabling `DBMS_OUTPUT`).
  * Executed using: `EXEC pr_name('value');` or `CALL pr_name('value');`

---

### **Microsoft SQL Server (T-SQL) Syntax**
```sql
CREATE OR ALTER PROCEDURE pr_name
    @p_param VARCHAR(50)
AS
BEGIN
    DECLARE @v_var INT;
    -- Business logic here
    PRINT 'Execution complete';
END;
```
* **Key Details:**
  * Uses `CREATE OR ALTER` syntax.
  * All parameters and variables MUST begin with `@` (e.g., `@p_param`, `@v_var`).
  * Explicit `DECLARE @v_var INT;` syntax for variables.
  * `PRINT` outputs console messages.
  * Executed using: `EXEC pr_name @p_param = 'value';`

---

### **MySQL Syntax**
```sql
DELIMITER $$

DROP PROCEDURE IF EXISTS pr_name$$

CREATE PROCEDURE pr_name(IN p_param VARCHAR(50))
BEGIN
    DECLARE v_var INT;
    -- Business logic here
    SELECT 'Execution complete' AS message;
END$$

DELIMITER ;
```
* **Key Details:**
  * Requires changing statement `DELIMITER` (e.g., `$$`) to prevent SQL standard `;` from closing the procedure definition early.
  * Does not support `CREATE OR REPLACE`; requires `DROP PROCEDURE IF EXISTS` beforehand.
  * Variables declared inside `BEGIN` using `DECLARE v_var INT;`.
  * Executed using: `CALL pr_name('value');`

---

## 3. Real-World Use Case & Database Schema

The tutorial builds a real-world inventory & sales management system using two core tables:

### **`products` Table Schema**
* `product_code` (VARCHAR): Unique SKU code (e.g., `'P1'`, `'P2'`)
* `product_name` (VARCHAR): Descriptive product name (e.g., `'iPhone 13 Pro Max'`)
* `price` (NUMERIC): Selling price per unit
* `quantity_remaining` (INT): Inventory count available
* `quantity_sold` (INT): Cumulative count sold

### **`sales` Table Schema**
* `order_id` (INT): Auto-increment primary key
* `order_date` (DATE): System date of transaction
* `product_code` (VARCHAR): Foreign key referencing `products`
* `quantity` (INT): Quantity purchased in the order
* `total_price` (NUMERIC): `quantity * unit_price`

---

## 4. Simple Stored Procedure (No Parameters)

### **Objective**
Automatically process a fixed order for `'iPhone 13 Pro Max'`:
1. Fetch unit price and product code.
2. Insert a new record into `sales`.
3. Update `quantity_remaining` and `quantity_sold` in `products`.

### **PostgreSQL Implementation**
```sql
CREATE OR REPLACE PROCEDURE pr_buy_products()
LANGUAGE plpgsql
AS $$
DECLARE
    v_product_code VARCHAR(20);
    v_price FLOAT;
BEGIN
    -- 1. Query price and code into variables
    SELECT product_code, price 
    INTO v_product_code, v_price 
    FROM products 
    WHERE product_name = 'iPhone 13 Pro Max';

    -- 2. Insert sales transaction record
    INSERT INTO sales (order_date, product_code, quantity, total_price)
    VALUES (CURRENT_DATE, v_product_code, 1, (v_price * 1));

    -- 3. Update product inventory quantities
    UPDATE products 
    SET quantity_remaining = (quantity_remaining - 1),
        quantity_sold = (quantity_sold + 1)
    WHERE product_code = v_product_code;

    -- 4. Print success notification
    RAISE NOTICE 'Product sold successfully!';
END;
$$;
```

### **Line-by-Line Explanation**
1. `CREATE OR REPLACE PROCEDURE pr_buy_products()`: Creates or replaces procedure `pr_buy_products` without input parameters.
2. `LANGUAGE plpgsql`: Defines procedural language engine for PostgreSQL.
3. `AS $$`: Opens body string block with double-dollar quotes.
4. `DECLARE`: Starts local variable declaration section.
5. `v_product_code VARCHAR(20);`: Declares variable to store SKU string.
6. `v_price FLOAT;`: Declares variable to store numeric price.
7. `BEGIN`: Starts procedure execution block.
8. `SELECT product_code, price INTO v_product_code, v_price FROM products WHERE product_name = 'iPhone 13 Pro Max';`: Selects `product_code` and `price` for the specified phone, populating local variables using the `INTO` keyword.
9. `INSERT INTO sales (...) VALUES (CURRENT_DATE, v_product_code, 1, (v_price * 1));`: Records a new sales order row with current date, product code, unit quantity 1, and calculated total price.
10. `UPDATE products SET quantity_remaining = (quantity_remaining - 1), quantity_sold = (quantity_sold + 1) WHERE product_code = v_product_code;`: Decrements remaining stock by 1 and increments sold count by 1 for the matching product.
11. `RAISE NOTICE 'Product sold successfully!';`: Logs a notice message to console output.
12. `END; $$;`: Closes the `BEGIN` block and dollar-quoted procedure definition.

---

## 5. Parameterized Stored Procedure (Dynamic Execution)

### **Objective**
Enhance the procedure to accept **dynamic parameters** (`p_product_name`, `p_quantity`), validating if sufficient stock exists prior to processing orders.

### **PostgreSQL Implementation**
```sql
CREATE OR REPLACE PROCEDURE pr_buy_products(
    p_product_name VARCHAR,
    p_quantity INT
)
LANGUAGE plpgsql
AS $$
DECLARE
    v_product_code VARCHAR(20);
    v_price FLOAT;
    v_count INT;
BEGIN
    -- Check if product exists with sufficient stock
    SELECT COUNT(1) 
    INTO v_count
    FROM products 
    WHERE product_name = p_product_name 
      AND quantity_remaining >= p_quantity;

    IF v_count > 0 THEN
        -- Fetch product details
        SELECT product_code, price 
        INTO v_product_code, v_price 
        FROM products 
        WHERE product_name = p_product_name;

        -- Record sale
        INSERT INTO sales (order_date, product_code, quantity, total_price)
        VALUES (CURRENT_DATE, v_product_code, p_quantity, (v_price * p_quantity));

        -- Update inventory
        UPDATE products 
        SET quantity_remaining = (quantity_remaining - p_quantity),
            quantity_sold = (quantity_sold + p_quantity)
        WHERE product_code = v_product_code;

        RAISE NOTICE 'Product sold successfully!';
    ELSE
        RAISE NOTICE 'Insufficient quantity or product unavailable!';
    END IF;
END;
$$;
```

### **Line-by-Line Explanation**
1. `p_product_name VARCHAR, p_quantity INT`: Defines input parameters passed dynamically at execution time.
2. `v_count INT;`: Declares variable to hold availability check result (1 if valid, 0 if invalid).
3. `SELECT COUNT(1) INTO v_count FROM products WHERE product_name = p_product_name AND quantity_remaining >= p_quantity;`: Queries whether matching stock is greater than or equal to requested purchase quantity `p_quantity`.
4. `IF v_count > 0 THEN`: Evaluates if sufficient inventory is available.
5. `(v_price * p_quantity)`: Dynamically computes total purchase cost based on requested quantity.
6. `quantity_remaining = (quantity_remaining - p_quantity)`: Subtracts requested quantity from stock.
7. `quantity_sold = (quantity_sold + p_quantity)`: Adds requested quantity to sold balance.
8. `ELSE ... END IF;`: Handles stock shortage gracefully by throwing an alert notice instead of failing silently or crashing.

---

## 6. Execution Examples across Dialects

### **PostgreSQL Execution**
```sql
CALL pr_buy_products('AirPods Pro', 2);
```

### **Oracle Execution**
```sql
EXEC pr_buy_products('AirPods Pro', 2);
```

### **SQL Server Execution**
```sql
EXEC pr_buy_products @p_product_name = 'AirPods Pro', @p_quantity = 2;
```

### **MySQL Execution**
```sql
CALL pr_buy_products('AirPods Pro', 2);
```

---

## Summary Syntax Reference Matrix

| Feature / Dialect | PostgreSQL | Oracle | MS SQL Server | MySQL |
| :--- | :--- | :--- | :--- | :--- |
| **Creation Keyword** | `CREATE OR REPLACE` | `CREATE OR REPLACE` | `CREATE OR ALTER` | `DROP ... + CREATE` |
| **Parameter Prefix** | None | None / `IN` / `OUT` | `@` | `IN` / `OUT` / `INOUT` |
| **Variable Prefix** | None | None | `@` | None |
| **Console Output** | `RAISE NOTICE` | `DBMS_OUTPUT.PUT_LINE` | `PRINT` | `SELECT ...` |
| **Body Delimiter** | `$$` block | `/` term | `BEGIN...END` | `DELIMITER $$` |
| **Execution Command**| `CALL` | `EXEC` / `CALL` | `EXEC` | `CALL` |
