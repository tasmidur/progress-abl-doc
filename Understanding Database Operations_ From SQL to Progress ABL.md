# Understanding Database Operations: From SQL to Progress ABL

Databases are the backbone of modern applications, storing and managing the data that powers everything from simple websites to complex enterprise systems. Whether you're a beginner just starting out or an experienced developer looking to expand your skill set, understanding how to interact with databases is crucial. In this post, we'll explore common database operations, comparing how they're performed in SQL (specifically PostgreSQL) and Progress ABL (Advanced Business Language), a language used in OpenEdge applications.

We'll start with the basics and gradually move to more advanced topics, ensuring you have a solid foundation before diving into complex concepts. By the end, you'll have a clear understanding of how these operations work in both SQL and ABL, and you'll be equipped to apply this knowledge in your own projects.

## Table of Contents

1. [Introduction to Database Operations](#introduction-to-database-operations)
2. [Beginner Operations](#beginner-operations)
   - [CREATE: Building the Foundation](#create-building-the-foundation)
   - [INSERT: Adding Data](#insert-adding-data)
   - [SELECT: Retrieving Data](#select-retrieving-data)
   - [UPDATE: Modifying Data](#update-modifying-data)
   - [DELETE: Removing Data](#delete-removing-data)
3. [Intermediate Operations](#intermediate-operations)
   - [ORDER BY: Sorting Results](#order-by-sorting-results)
   - [Aggregate Functions: Summarizing Data](#aggregate-functions-summarizing-data)
   - [JOIN: Combining Tables](#join-combining-tables)
4. [Advanced Operations](#advanced-operations)
   - [Transactions: Ensuring Data Integrity](#transactions-ensuring-data-integrity)
   - [Pattern Matching: Finding Specific Data](#pattern-matching-finding-specific-data)
   - [CASE Statements: Adding Conditional Logic](#case-statements-adding-conditional-logic)
   - [Nested Queries: Queries Within Queries](#nested-queries-queries-within-queries)
   - [Renaming: Aliases for Clarity](#renaming-aliases-for-clarity)
5. [Additional Operations](#additional-operations)
   - [TRUNCATE: Emptying Tables](#truncate-emptying-tables)
   - [DROP: Deleting Tables](#drop-deleting-tables)
   - [RENAME: Changing Names](#rename-changing-names)
6. [Conclusion](#conclusion)

## Introduction to Database Operations

Database operations are the actions we perform to manage and manipulate data stored in a database. These operations can be broadly categorized into four types:

- **Data Definition Language (DDL)**: Operations that define or modify the structure of the database, such as creating or dropping tables.
- **Data Manipulation Language (DML)**: Operations that manipulate the data itself, such as inserting, updating, or deleting records.
- **Data Query Language (DQL)**: Operations that retrieve data from the database, primarily using the `SELECT` statement.
- **Transaction Control Language (TCL)**: Operations that manage transactions to ensure data integrity.

In this post, we'll focus on DML and DQL operations, with a brief look at DDL and TCL where relevant. We'll compare how these operations are performed in SQL, a widely-used language for relational databases, and Progress ABL, a procedural language used in OpenEdge applications.

SQL is declarative, meaning you specify what you want, and the database figures out how to get it. ABL, on the other hand, is procedural and record-oriented, requiring you to specify how to access and manipulate data step by step.

Let's start with the basics.

## Beginner Operations

### CREATE: Building the Foundation

**SQL (PostgreSQL)**:
In SQL, you create a table using the `CREATE TABLE` statement, defining columns, data types, and constraints.

```sql
CREATE TABLE customers (
    customer_id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    city VARCHAR(50),
    balance NUMERIC(10, 2)
);
```

This creates a table named `customers` with four columns: `customer_id`, `name`, `city`, and `balance`. The `customer_id` is an auto-incrementing primary key.

**Progress ABL**:
In ABL, permanent tables are defined in the Data Dictionary tool, not in code. However, for temporary in-memory tables (temp-tables), you use the `DEFINE TEMP-TABLE` statement:

```abl
DEFINE TEMP-TABLE ttCustomers NO-UNDO
    FIELD customer_id AS INTEGER
    FIELD name AS CHARACTER FORMAT "x(100)"
    FIELD city AS CHARACTER FORMAT "x(50)"
    FIELD balance AS DECIMAL FORMAT "->>,>>9.99".
```

Temp-tables are session-specific and don't persist data beyond the program's execution. Unlike SQL, ABL doesn't support constraints like `PRIMARY KEY` in code; you must enforce uniqueness programmatically if needed.

**Key Takeaway**: SQL allows you to define table structures and constraints directly in code, while ABL relies on the Data Dictionary for permanent tables and offers temp-tables for in-memory data manipulation.

### INSERT: Adding Data

**SQL (PostgreSQL)**:
To add a new record, use the `INSERT INTO` statement:

```sql
INSERT INTO customers (name, city, balance) 
VALUES ('John Doe', 'London', 500.00);
```

This inserts a new row into the `customers` table with the specified values.

**Progress ABL**:
In ABL, you create a new record using the `CREATE` statement and then assign values to its fields:

```abl
CREATE customers.
ASSIGN 
    customers.name = "John Doe"
    customers.city = "London"
    customers.balance = 500.00.
```

Alternatively, you can use direct assignments:

```abl
CREATE customers.
customers.name = "John Doe".
customers.city = "London".
customers.balance = 500.00.
```

**Key Takeaway**: SQL's `INSERT` is a single statement that adds a row, while ABL's `CREATE` and `ASSIGN` handle record creation and field assignment separately.

### SELECT: Retrieving Data

**SQL (PostgreSQL)**:
The `SELECT` statement retrieves data from one or more tables:

```sql
SELECT name, city 
FROM customers 
WHERE balance > 1000;
```

This query selects the `name` and `city` of customers with a balance greater than 1000.

**Progress ABL**:
ABL uses a `FOR EACH` loop to iterate over records that match a condition:

```abl
FOR EACH customers NO-LOCK WHERE customers.balance > 1000:
    DISPLAY customers.name customers.city.
END.
```

The `NO-LOCK` keyword ensures read-only access, similar to PostgreSQL's default behavior.

**Key Takeaway**: SQL's `SELECT` is declarative, specifying what data to retrieve, while ABL's `FOR EACH` is procedural, iterating over each matching record.

### UPDATE: Modifying Data

**SQL (PostgreSQL)**:
The `UPDATE` statement modifies existing records:

```sql
UPDATE customers 
SET balance = balance + 100 
WHERE city = 'New York';
```

This increases the balance by 100 for all customers in New York.

**Progress ABL**:
ABL updates records within a loop, using `EXCLUSIVE-LOCK` to allow modifications:

```abl
FOR EACH customers EXCLUSIVE-LOCK WHERE customers.city = "New York":
    customers.balance = customers.balance + 100.
END.
```

Each record is updated individually.

**Key Takeaway**: SQL updates multiple rows in a single statement, while ABL requires iterating over each record to apply changes.

### DELETE: Removing Data

**SQL (PostgreSQL)**:
The `DELETE` statement removes records that match a condition:

```sql
DELETE FROM customers 
WHERE balance < 0;
```

This deletes all customers with a negative balance.

**Progress ABL**:
ABL deletes records one at a time within a loop:

```abl
FOR EACH customers EXCLUSIVE-LOCK WHERE customers.balance < 0:
    DELETE customers.
END.
```

**Key Takeaway**: Similar to `UPDATE`, SQL's `DELETE` is set-based, while ABL's `DELETE` is record-based.

## Intermediate Operations

### ORDER BY: Sorting Results

**SQL (PostgreSQL)**:
The `ORDER BY` clause sorts the result set:

```sql
SELECT name, city 
FROM customers 
WHERE country = 'USA' 
ORDER BY city ASC, name DESC;
```

This sorts customers from the USA by city in ascending order and name in descending order.

**Progress ABL**:
ABL uses the `BY` keyword within `FOR EACH` to sort records:

```abl
FOR EACH customers NO-LOCK 
    WHERE customers.country = "USA"
    BY customers.city 
    BY customers.name DESCENDING:
    DISPLAY customers.name customers.city.
END.
```

Multiple `BY` clauses chain the sort order.

**Key Takeaway**: Both SQL and ABL support sorting, but ABL integrates sorting directly into the loop construct.

### Aggregate Functions: Summarizing Data

**SQL (PostgreSQL)**:
Aggregate functions like `COUNT`, `SUM`, etc., summarize data:

```sql
SELECT city, COUNT(*) AS customer_count, SUM(balance) AS total_balance
FROM customers
GROUP BY city
HAVING COUNT(*) > 5;
```

This groups customers by city, counts them, and sums their balances, showing only cities with more than 5 customers.

**Progress ABL**:
ABL lacks direct aggregate functions, so you use procedural logic with `BREAK BY`:

```abl
DEFINE VARIABLE customerCount AS INTEGER NO-UNDO.
DEFINE VARIABLE totalBalance AS DECIMAL NO-UNDO.

FOR EACH customers NO-LOCK 
    BREAK BY customers.city:
    IF FIRST-OF(customers.city) THEN
        ASSIGN customerCount = 0
               totalBalance = 0.
    ASSIGN customerCount = customerCount + 1
           totalBalance = totalBalance + customers.balance.
    IF LAST-OF(customers.city) AND customerCount > 5 THEN
        DISPLAY customers.city customerCount totalBalance.
END.
```

`BREAK BY` groups records, and `FIRST-OF` and `LAST-OF` help manage group boundaries.

**Key Takeaway**: SQL's aggregate functions are concise, while ABL requires manual accumulation and grouping.

### JOIN: Combining Tables

**SQL (PostgreSQL)**:
The `JOIN` clause combines data from multiple tables:

```sql
SELECT c.name, o.order_date 
FROM customers c 
JOIN orders o ON c.customer_id = o.customer_id 
WHERE c.city = 'Boston';
```

This retrieves names and order dates for customers in Boston.

**Progress ABL**:
ABL doesn't support SQL-style joins, so you nest `FOR EACH` loops:

```abl
FOR EACH customers NO-LOCK WHERE customers.city = "Boston",
    EACH orders NO-LOCK WHERE orders.customer_id = customers.customer_id:
    DISPLAY customers.name orders.order_date.
END.
```

The comma acts like an implicit join condition.

**Key Takeaway**: SQL's `JOIN` is declarative, while ABL's nested loops are procedural but achieve similar results.

## Advanced Operations

### Transactions: Ensuring Data Integrity

**SQL (PostgreSQL)**:
Transactions ensure multiple operations succeed or fail together:

```sql
BEGIN;
UPDATE customers SET balance = balance - 100 WHERE customer_id = 1;
UPDATE accounts SET balance = balance + 100 WHERE account_id = 101;
COMMIT;
```

If both updates succeed, the transaction commits; otherwise, it rolls back.

**Progress ABL**:
ABL uses `DO TRANSACTION` blocks:

```abl
DO TRANSACTION:
    FIND customers EXCLUSIVE-LOCK WHERE customers.customer_id = 1.
    customers.balance = customers.balance - 100.
    
    FIND accounts EXCLUSIVE-LOCK WHERE accounts.account_id = 101.
    accounts.balance = accounts.balance + 100.
END.
```

The block ensures atomicity; if an error occurs, changes are rolled back.

**Key Takeaway**: Both SQL and ABL support transactions, but ABL's syntax is more procedural.

### Pattern Matching: Finding Specific Data

**SQL (PostgreSQL)**:
The `LIKE` operator performs pattern matching:

```sql
SELECT name, city 
FROM customers 
WHERE name LIKE 'J%';
```

This finds names starting with 'J'.

**Progress ABL**:
ABL uses `BEGINS` for prefix matching or `MATCHES` for patterns:

```abl
FOR EACH customers NO-LOCK WHERE customers.name BEGINS "J":
    DISPLAY customers.name customers.city.
END.
```

For more complex patterns, use `MATCHES`:

```abl
FOR EACH customers NO-LOCK WHERE customers.name MATCHES "J*":
    DISPLAY customers.name customers.city.
END.
```

**Key Takeaway**: SQL's `LIKE` is flexible with wildcards, while ABL's `BEGINS` and `MATCHES` offer similar functionality.

### CASE Statements: Adding Conditional Logic

**SQL (PostgreSQL)**:
The `CASE` statement adds conditional logic to queries:

```sql
SELECT name, 
       CASE 
           WHEN balance > 1000 THEN 'High'
           WHEN balance > 500 THEN 'Medium'
           ELSE 'Low'
       END AS balance_category
FROM customers;
```

This categorizes customers based on their balance.

**Progress ABL**:
ABL uses `IF-ELSE` logic within loops:

```abl
FOR EACH customers NO-LOCK:
    DEFINE VARIABLE balanceCategory AS CHARACTER NO-UNDO.
    IF customers.balance > 1000 THEN
        balanceCategory = "High".
    ELSE IF customers.balance > 500 THEN
        balanceCategory = "Medium".
    ELSE
        balanceCategory = "Low".
    DISPLAY customers.name balanceCategory.
END.
```

**Key Takeaway**: SQL's `CASE` is declarative and part of the query, while ABL's conditional logic is procedural.

### Nested Queries: Queries Within Queries

**SQL (PostgreSQL)**:
Nested queries (subqueries) allow queries within queries:

```sql
SELECT name, city 
FROM customers 
WHERE customer_id IN (
    SELECT customer_id 
    FROM orders 
    WHERE order_date > '2024-01-01'
);
```

This finds customers who placed orders after January 1, 2024.

**Progress ABL**:
ABL doesn't support subqueries directly, so you use nested loops or temp-tables:

```abl
FOR EACH customers NO-LOCK:
    DEFINE VARIABLE hasOrder AS LOGICAL NO-UNDO.
    hasOrder = FALSE.
    FOR EACH orders NO-LOCK 
        WHERE orders.customer_id = customers.customer_id 
        AND orders.order_date > DATE("01/01/2024"):
        hasOrder = TRUE.
        LEAVE.
    END.
    IF hasOrder THEN
        DISPLAY customers.name customers.city.
END.
```

Alternatively, use a temp-table to store intermediate results.

**Key Takeaway**: SQL's subqueries are powerful and concise, while ABL requires more code to achieve similar results.

### Renaming: Aliases for Clarity

**SQL (PostgreSQL)**:
Aliases rename columns or tables for clarity:

```sql
SELECT c.name AS customer_name, c.city AS customer_city
FROM customers c
WHERE c.balance > 1000;
```

This renames columns and uses a table alias `c`.

**Progress ABL**:
ABL doesn't support column aliases in the same way, but you can use `LABEL` in `DISPLAY`:

```abl
FOR EACH customers NO-LOCK WHERE customers.balance > 1000:
    DISPLAY 
        customers.name LABEL "Customer Name"
        customers.city LABEL "Customer City".
END.
```

For programmatic use, assign to variables:

```abl
FOR EACH customers NO-LOCK WHERE customers.balance > 1000:
    DEFINE VARIABLE customerName AS CHARACTER NO-UNDO.
    DEFINE VARIABLE customerCity AS CHARACTER NO-UNDO.
    ASSIGN 
        customerName = customers.name
        customerCity = customers.city.
    DISPLAY customerName customerCity.
END.
```

**Key Takeaway**: SQL's aliases are straightforward, while ABL offers display labels or variable assignments for similar effects.

## Additional Operations

### TRUNCATE: Emptying Tables

**SQL (PostgreSQL)**:
`TRUNCATE` quickly removes all rows from a table:

```sql
TRUNCATE TABLE customers;
```

**Progress ABL**:
ABL has no direct `TRUNCATE`; you must delete records one by one:

```abl
FOR EACH customers EXCLUSIVE-LOCK:
    DELETE customers.
END.
```

For temp-tables, use `EMPTY TEMP-TABLE ttCustomers.`.

**Key Takeaway**: SQL's `TRUNCATE` is efficient, while ABL's loop is slower for large tables.

### DROP: Deleting Tables

**SQL (PostgreSQL)**:
`DROP TABLE` deletes a table and its data:

```sql
DROP TABLE customers;
```

**Progress ABL**:
Permanent tables are managed via the Data Dictionary, not code. For temp-tables, they expire at session end.

**Key Takeaway**: SQL allows table deletion in code, while ABL requires administrative tools for permanent tables.

### RENAME: Changing Names

**SQL (PostgreSQL)**:
`ALTER TABLE` renames tables or columns:

```sql
ALTER TABLE customers RENAME TO clients;
ALTER TABLE customers RENAME COLUMN city TO town;
```

**Progress ABL**:
Renaming isn't supported in code; use the Data Dictionary for permanent tables or redefine temp-tables.

**Key Takeaway**: SQL provides direct renaming capabilities, while ABL does not.

## Conclusion

Understanding database operations is essential for any developer working with data. While SQL and Progress ABL approach these operations differently—SQL being declarative and set-based, ABL being procedural and record-oriented—both languages provide the tools needed to manage and manipulate data effectively.

By starting with basic operations like `CREATE`, `INSERT`, `SELECT`, `UPDATE`, and `DELETE`, and progressing to more advanced concepts like transactions, pattern matching, and nested queries, you've gained a comprehensive overview of how these operations translate between SQL and ABL.

Remember, practice is key. Try implementing these operations in your own projects to solidify your understanding. Whether you're working with SQL or ABL, the principles remain the same: define your data, manipulate it carefully, and ensure its integrity.

Happy coding!
