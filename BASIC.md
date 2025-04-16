# Progress ABL: Basic and Advanced Documentation

This document provides a comprehensive guide to **Progress Advanced Business Language (ABL)**, covering both **basic** and **advanced** concepts. It is designed for beginners learning ABL syntax and experienced developers exploring advanced features like object-oriented programming, web development, and performance optimization. The document includes code examples, best practices, common mistakes, and references to official Progress OpenEdge documentation.

## Official Documentation Sources
- **Progress OpenEdge Documentation**: [docs.progress.com](https://docs.progress.com) (OpenEdge > ABL Reference).
- **ABL Reference**: [ABL Reference](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/abl-reference.html).
- **ABL Dojo**: [ABL Dojo](https://www.progress.com/openedge/abl-dojo) for interactive tutorials.
- **Progress Community**: [Progress Community](https://community.progress.com) for forums, examples, and updates.

---

## 1. Basic ABL Concepts

### 1.1 Program Structure
ABL programs are typically stored in `.p` (procedure) or `.w` (window) files. They consist of statements, procedures, and optional database interactions.

**Example**:
```abl
/* HelloWorld.p */
DISPLAY "Hello, World!" WITH FRAME f1 TITLE "Welcome".
```

- **Description**: Outputs "Hello, World!" in a framed window. ABL is case-insensitive and uses `/* */` for comments.
- **Documentation**: [Introducing ABL](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/introducing-abl.html).

### 1.2 Variables and Data Types
Variables are declared with `DEFINE VARIABLE`, specifying a data type such as:
- `CHARACTER` (string)
- `INTEGER` (whole number)
- `DECIMAL` (floating-point)
- `DATE` (date)
- `LOGICAL` (true/false)

**Syntax**:
```abl
DEFINE VARIABLE var-name AS data-type [NO-UNDO] [INITIAL value].
```

**Example**:
```abl
DEFINE VARIABLE cName AS CHARACTER NO-UNDO INITIAL "Alice".
DEFINE VARIABLE iAge AS INTEGER NO-UNDO INITIAL 30.
DEFINE VARIABLE dSalary AS DECIMAL NO-UNDO INITIAL 50000.50.
DISPLAY cName iAge dSalary.
```

- **Best Practice**: Use `NO-UNDO` for non-transactional variables to reduce overhead.
- **Common Mistake**: Omitting `NO-UNDO`, causing unnecessary transaction logging.
- **Documentation**: [Data Types](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/data-types.html).

### 1.3 Control Structures
ABL supports conditionals and loops for flow control.

#### IF-THEN-ELSE
**Syntax**:
```abl
IF condition THEN
    statement.
[ELSE
    statement.]
```

**Example**:
```abl
DEFINE VARIABLE iAge AS INTEGER NO-UNDO INITIAL 20.
IF iAge >= 18 THEN
    DISPLAY "Adult".
ELSE
    DISPLAY "Minor".
```

#### FOR EACH (Database Loop)
**Syntax**:
```abl
FOR EACH table-name [WHERE condition] [NO-LOCK]:
    statement.
END.
```

**Example**:
```abl
FOR EACH customer NO-LOCK WHERE customer.country = "USA":
    DISPLAY customer.name customer.balance.
END.
```

#### DO (General Loop)
**Syntax**:
```abl
DO [variable = start TO end]:
    statement.
END.
```

**Example**:
```abl
DO i = 1 TO 5:
    DISPLAY i.
END.
```

- **Best Practice**: Use `NO-LOCK` for read-only database queries to avoid locking overhead.
- **Common Mistake**: Using `EXCLUSIVE-LOCK` for read operations, causing contention.
- **Documentation**: [ABL Statements](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/abl-statements.html).

### 1.4 Database Operations
ABL excels in database integration, particularly with Progress databases like `sports2000`.

#### Querying Data
**Example**:
```abl
FIND FIRST customer WHERE customer.cust-num = 100 NO-LOCK NO-ERROR.
IF AVAILABLE customer THEN
    DISPLAY customer.name.
ELSE
    DISPLAY "Customer not found".
```

#### Updating Data
**Example**:
```abl
DO TRANSACTION:
    FOR EACH customer EXCLUSIVE-LOCK WHERE customer.balance < 0:
        customer.balance = 0.
    END.
END.
```

- **Best Practice**: Use explicit `DO TRANSACTION` blocks for updates to control scope.
- **Common Mistake**: Not using `NO-ERROR` with `FIND`, causing runtime errors.
- **Documentation**: [Database Access](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/database-access.html).

### 1.5 Output and Frames
ABL supports console and GUI output using `DISPLAY` and `MESSAGE`.

**Example**:
```abl
DEFINE FRAME f1
    customer.name LABEL "Name"
    customer.balance LABEL "Balance"
    WITH 10 DOWN TITLE "Customers" CENTERED.

FOR EACH customer NO-LOCK:
    DISPLAY customer.name customer.balance WITH FRAME f1.
END.

MESSAGE "Query complete." VIEW-AS ALERT-BOX.
```

- **Best Practice**: Use frames to organize output; specify `FORMAT` for alignment.
- **Common Mistake**: Overusing `MESSAGE` for non-critical output, cluttering the UI.
- **Documentation**: [User Interface](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/user-interface.html).

### 1.6 Procedures and Functions
ABL supports modular programming.

#### Procedure
**Example**:
```abl
PROCEDURE greet:
    DEFINE INPUT PARAMETER cName AS CHARACTER NO-UNDO.
    DISPLAY "Hello, " + cName.
END PROCEDURE.

RUN greet("Bob").
```

#### Function
**Example**:
```abl
FUNCTION addNumbers RETURNS INTEGER (iNum1 AS INTEGER, iNum2 AS INTEGER):
    RETURN iNum1 + iNum2.
END FUNCTION.

DISPLAY addNumbers(5, 3). /* Outputs 8 */
```

- **Best Practice**: Use procedures for reusable logic; store in `.p` files for modularity.
- **Common Mistake**: Defining functions without clear return types, causing errors.
- **Documentation**: [Procedures and Functions](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/procedures-and-functions.html).

---

## 2. Advanced ABL Concepts

### 2.1 Object-Oriented Programming (OOP)
Since OpenEdge 11.0, ABL supports OOP, allowing classes, inheritance, and interfaces.

**Example** (Class Definition):
```abl
/* Customer.cls */
CLASS Customer:
    DEFINE PUBLIC PROPERTY CustNum AS INTEGER NO-UNDO
        GET.
        PRIVATE SET.
    
    DEFINE PUBLIC PROPERTY Name AS CHARACTER NO-UNDO
        GET.
        SET.
    
    CONSTRUCTOR PUBLIC Customer (INPUT iCustNum AS INTEGER, INPUT cName AS CHARACTER):
        CustNum = iCustNum.
        Name = cName.
    END CONSTRUCTOR.
    
    METHOD PUBLIC DECIMAL GetBalance():
        FIND customer WHERE customer.cust-num = CustNum NO-LOCK NO-ERROR.
        IF AVAILABLE customer THEN
            RETURN customer.balance.
        RETURN 0.0.
    END METHOD.
END CLASS.
```

**Example** (Using the Class):
```abl
/* UseCustomer.p */
DEFINE VARIABLE oCustomer AS Customer NO-UNDO.
oCustomer = NEW Customer(100, "Acme Sports").
DISPLAY oCustomer:Name oCustomer:GetBalance().
DELETE OBJECT oCustomer.
```

- **Best Practice**: Use classes for complex business logic; encapsulate data with properties.
- **Common Mistake**: Not deleting objects (`DELETE OBJECT`), causing memory leaks.
- **Documentation**: [Object-Oriented Programming](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/object-oriented-programming.html).

### 2.2 Web Development with SpeedScript
SpeedScript is an ABL subset for web applications, integrating with HTML, JavaScript, and CSS.

**Example**:
```abl
/* CustomerList.w */
<HTML>
<HEAD><TITLE>Customer List</TITLE></HEAD>
<BODY>
<H1>Customers</H1>
<TABLE BORDER="1">
<TR><TH>ID</TH><TH>Name</TH><TH>Balance</TH></TR>
<SCRIPT LANGUAGE="SpeedScript">
FOR EACH customer NO-LOCK:
    OUTPUT TO "WEB".
    {&OUT} "<TR><TD>" customer.cust-num "</TD><TD>" customer.name "</TD><TD>" customer.balance "</TD></TR>".
END.
</SCRIPT>
</TABLE>
</BODY>
</HTML>
```

- **Best Practice**: Use `{&OUT}` for dynamic HTML output; separate logic into `.p` files.
- **Common Mistake**: Embedding complex logic in SpeedScript, reducing maintainability.
- **Documentation**: [SpeedScript](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/speedscript.html).

### 2.3 File Management and Imports
ABL supports reading/writing external files and importing data.

#### Importing CSV
**Example**:
```abl
DEFINE TEMP-TABLE ttCustomer
    FIELD custNum AS INTEGER
    FIELD custName AS CHARACTER
    FIELD balance AS DECIMAL.

INPUT FROM "customers.csv".
REPEAT:
    CREATE ttCustomer.
    IMPORT DELIMITER "," ttCustomer NO-ERROR.
    IF ERROR-STATUS:ERROR THEN
        NEXT.
END.
INPUT CLOSE.

FOR EACH ttCustomer:
    CREATE customer.
    ASSIGN
        customer.cust-num = ttCustomer.custNum
        customer.name = ttCustomer.custName
        customer.balance = ttCustomer.balance.
END.
```

- **Best Practice**: Use temp-tables for staging; validate data with `NO-ERROR`.
- **Common Mistake**: Not closing streams (`INPUT CLOSE`), causing resource leaks.
- **Documentation**: [IMPORT Statement](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/import-statement.html).

#### File Existence Check
**Example**:
```abl
FILE-INFO:FILE-NAME = "data.txt".
IF FILE-INFO:FULL-PATHNAME = ? THEN
    MESSAGE "File not found." VIEW-AS ALERT-BOX.
```

- **Documentation**: [FILE-INFO Handle](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/file-info-handle.html).

### 2.4 Performance Tuning
ABL programs can be optimized for speed and resource usage.

**Best Practices**:
1. **Use Indexes**: Align `WHERE` clauses with database indexes.
   ```abl
   FOR EACH customer NO-LOCK WHERE customer.cust-num = 100:
       DISPLAY customer.name.
   END.
   ```
2. **Minimize Transactions**: Use small, explicit `DO TRANSACTION` blocks.
3. **Profile Code**: Use `PROFILER` or ProTop to identify bottlenecks.
4. **Optimize Queries**: Avoid `BY` clauses unless sorting is required.

**Common Mistakes**:
- Writing unindexed queries (e.g., `WHERE UPPER(name) = "SMITH"`).
- Using large transactions, causing lock contention.

**Documentation**: [Performance Tuning](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/performance-tuning.html).

### 2.5 Error Handling
ABL provides robust error handling with `NO-ERROR` and `ERROR-STATUS`.

**Example**:
```abl
FIND customer WHERE customer.cust-num = 999 NO-ERROR.
IF ERROR-STATUS:ERROR OR NOT AVAILABLE customer THEN
    MESSAGE "Customer not found." VIEW-AS ALERT-BOX.
ELSE
    DISPLAY customer.name.
```

- **Best Practice**: Always use `NO-ERROR` for database operations; log errors for debugging.
- **Common Mistake**: Not checking `ERROR-STATUS`, leading to unhandled errors.
- **Documentation**: [Error Handling](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/error-handling.html).

### 2.6 File Reuse with Include Files
Include files (`.i`) store reusable code, such as constants or procedures.

**Example**:
```abl
/* common.i */
DEFINE VARIABLE cLogFile AS CHARACTER NO-UNDO INITIAL "app.log".
PROCEDURE logMessage:
    DEFINE INPUT PARAMETER cMsg AS CHARACTER NO-UNDO.
    OUTPUT TO VALUE(cLogFile) APPEND.
    PUT cMsg SKIP.
    OUTPUT CLOSE.
END PROCEDURE.
```

```abl
/* main.p */
{common.i}
RUN logMessage("Application started").
```

- **Best Practice**: Use include files for shared logic; store in a central directory.
- **Common Mistake**: Hardcoding values instead of using include files.
- **Documentation**: [Include Files](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/include-files.html).

---

## 3. Best Practices (Basic and Advanced)
1. **Naming Conventions**:
   - Use descriptive names (e.g., `cCustomerName`, `iCustNum`) for clarity.
2. **Modular Design**:
   - Break code into procedures, functions, or classes; store reusable code in `.p` or `.i` files.
3. **Error Handling**:
   - Use `NO-ERROR` and `ERROR-STATUS` for robust applications.
4. **Performance**:
   - Optimize queries with indexes; use `NO-UNDO` for non-transactional variables.
5. **Documentation**:
   - Comment code and maintain external documentation for complex logic.
6. **Version Control**:
   - Use Git or similar for `.p`, `.w`, and `.i` files to track changes.

---

## 4. Common Mistakes
1. **Database**:
   - Using `EXCLUSIVE-LOCK` for read-only queries.
   - Writing unindexed queries (e.g., `FOR EACH customer NO-LOCK` without `WHERE`).
2. **Variables**:
   - Omitting `NO-UNDO`, increasing transaction overhead.
3. **File Management**:
   - Hardcoding file paths (e.g., `INPUT FROM "C:/data.txt"`).
   - Not closing streams (`INPUT CLOSE`, `OUTPUT CLOSE`).
4. **OOP**:
   - Not deleting objects (`DELETE OBJECT`), causing memory leaks.
5. **SpeedScript**:
   - Mixing heavy logic with HTML, reducing maintainability.

---

## 5. Example Program (Combining Basic and Advanced Concepts)
This program imports customer data, updates the database using a class, and generates a web report.

```abl
/* CustomerManager.p */
{config.i} /* Defines cDataDir = "C:/data/" */

/* Class for customer operations */
CLASS CustomerManager:
    DEFINE PRIVATE TEMP-TABLE ttCustomer
        FIELD custNum AS INTEGER
        FIELD custName AS CHARACTER
        FIELD balance AS DECIMAL.
    
    METHOD PUBLIC VOID ImportCustomers(INPUT cFile AS CHARACTER):
        FILE-INFO:FILE-NAME = cFile.
        IF FILE-INFO:FULL-PATHNAME = ? THEN
            RETURN ERROR "File not found".
        
        INPUT FROM VALUE(cFile).
        REPEAT:
            CREATE ttCustomer.
            IMPORT DELIMITER "," ttCustomer NO-ERROR.
            IF ERROR-STATUS:ERROR THEN
                NEXT.
        END.
        INPUT CLOSE.
    END METHOD.
    
    METHOD PUBLIC VOID UpdateDatabase():
        DO TRANSACTION:
            FOR EACH ttCustomer:
                FIND customer WHERE customer.cust-num = ttCustomer.custNum NO-ERROR.
                IF NOT AVAILABLE customer THEN
                    CREATE customer.
                ASSIGN
                    customer.cust-num = ttCustomer.custNum
                    customer.name = ttCustomer.custName
                    customer.balance = ttCustomer.balance.
            END.
        END.
    END METHOD.
END CLASS.

/* Web report */
OUTPUT TO "WEB".
{&OUT} "<HTML><HEAD><TITLE>Customer Report</TITLE></HEAD><BODY>".
{&OUT} "<TABLE BORDER='1'><TR><TH>ID</TH><TH>Name</TH><TH>Balance</TH></TR>".

FOR EACH customer NO-LOCK WHERE customer.balance > 1000:
    {&OUT} "<TR><TD>" customer.cust-num "</TD><TD>" customer.name "</TD><TD>" customer.balance "</TD></TR>".
END.

{&OUT} "</TABLE></BODY></HTML>".
OUTPUT CLOSE.

/* Main block */
DEFINE VARIABLE oManager AS CustomerManager NO-UNDO.
oManager = NEW CustomerManager().
oManager:ImportCustomers(cDataDir + "customers.csv").
oManager:UpdateDatabase().
DELETE OBJECT oManager.
```

### Explanation
- **Basic**: Uses variables, database queries (`FOR EACH`), and file I/O (`IMPORT`).
- **Advanced**: Implements a class (`CustomerManager`), uses SpeedScript for web output, and manages errors.
- **Best Practices**: Modular design, error handling, dynamic file paths, and object cleanup.
- **Documentation**: Combines [OOP](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/object-oriented-programming.html), [SpeedScript](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/speedscript.html), and [File I/O](https://documentation.progress.com/output/ua/OpenEdge_latest/index.html#page/abl/input-from-statement.html).

---

## 6. Additional Resources
- **ABL Dojo**: Test code interactively at [ABL Dojo](https://www.progress.com/openedge/abl-dojo).
- **Progress Community**: Find examples and support at [Progress Community](https://community.progress.com).
- **Whitepapers**: Download from [www.progress.com](https://www.progress.com) for topics like ABL performance or cloud deployment.
- **Tutorials**: Check [riptutorial.com](https://riptutorial.com/progress-4gl) for beginner guides.

This document provides a foundation for ABL development and a springboard for advanced applications. For specific topics (e.g., .NET integration, cloud deployment, or detailed performance tuning), consult the referenced documentation or request tailored examples.
