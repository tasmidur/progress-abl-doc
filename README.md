# ABL Best Practices and Common Mistakes for Database Queries, File Reuse, and External File Management

This document provides best practices, highlights common mistakes, and offers guidance on database queries, file reuse, data imports, and external file management in Progress Advanced Business Language (ABL). It includes code examples and references to the official Progress OpenEdge documentation for further details.

## Official Documentation Sources

- **Progress OpenEdge Documentation**: docs.progress.com (navigate to OpenEdge &gt; ABL Reference).
- **ABL Reference**: ABL Reference.
- **ABL Dojo**: ABL Dojo for interactive tutorials.
- **Progress Community**: Progress Community for forums and examples.

---

## 1. Database Queries

### Best Practices

1. **Use Appropriate Locking**:

   - Use `NO-LOCK` for read-only queries to avoid unnecessary locks and improve performance.
   - Use `EXCLUSIVE-LOCK` for updates, but limit its scope to minimize contention.
   - Use `SHARE-LOCK` only when shared read access with potential updates is needed.
   - **Example**:

     ```abl
     FOR EACH customer NO-LOCK WHERE customer.country = "USA":
         DISPLAY customer.name customer.balance.
     END.
     ```
   - **Reference**: Database Access.

2. **Leverage Indexes**:

   - Write `WHERE` clauses that align with database indexes to optimize query performance.
   - Use tools like `prostrct statistics` to analyze index usage.
   - **Example**:

     ```abl
     FOR EACH customer NO-LOCK WHERE customer.cust-num = 100:
         DISPLAY customer.name.
     END.
     ```
   - **Reference**: Index Usage.

3. **Use Transactions Sparingly**:

   - Wrap updates in explicit `DO TRANSACTION` blocks to control scope.
   - Avoid large transactions to prevent locking issues and transaction log overflow.
   - **Example**:

     ```abl
     DO TRANSACTION:
         FOR EACH customer EXCLUSIVE-LOCK WHERE customer.balance < 0:
             customer.balance = 0.
         END.
     END.
     ```
   - **Reference**: Transaction Management.

4. **Handle Errors**:

   - Use `NO-ERROR` with database operations and check `ERROR-STATUS:ERROR` to handle failures gracefully.
   - **Example**:

     ```abl
     FIND customer WHERE customer.cust-num = 999 NO-ERROR.
     IF NOT AVAILABLE customer THEN
         MESSAGE "Customer not found." VIEW-AS ALERT-BOX.
     ```
   - **Reference**: Error Handling.

5. **Use Query Tuning**:

   - Use `BY` clauses for sorting only when necessary, as they can impact performance.
   - Test queries with `QUERY-TUNING` options for optimization.
   - **Reference**: Query Tuning.

### Common Mistakes

1. **Overusing EXCLUSIVE-LOCK**:

   - Applying `EXCLUSIVE-LOCK` unnecessarily for read operations slows performance and causes contention.
   - **Mistake**:

     ```abl
     FOR EACH customer EXCLUSIVE-LOCK: /* Avoid for read-only */
         DISPLAY customer.name.
     END.
     ```
   - **Fix**: Use `NO-LOCK` for read-only queries.

2. **Ignoring Indexes**:

   - Writing `WHERE` clauses that don’t use indexes leads to full-table scans.
   - **Mistake**:

     ```abl
     FOR EACH customer NO-LOCK WHERE UPPER(customer.name) = "SMITH":
         DISPLAY customer.name.
     END.
     ```
   - **Fix**: Use indexed fields like `cust-num` or ensure `name` has an index.

3. **Unbounded Queries**:

   - Queries without `WHERE` clauses or with broad conditions can process excessive records.
   - **Mistake**:

     ```abl
     FOR EACH customer NO-LOCK:
         DISPLAY customer.name.
     END.
     ```
   - **Fix**: Add specific `WHERE` conditions.

4. **Missing Error Handling**:

   - Not using `NO-ERROR` can cause runtime errors to halt the program.
   - **Mistake**:

     ```abl
     FIND customer WHERE customer.cust-num = 999.
     DISPLAY customer.name.
     ```
   - **Fix**: Add `NO-ERROR` and check `AVAILABLE`.

---

## 2. File Reuse

### Best Practices

1. **Use Include Files for Shared Code**:

   - Store reusable code (e.g., constants, procedures) in `.i` include files to avoid duplication.
   - **Example**:

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
     RUN logMessage("Starting application").
     ```
   - **Reference**: Include Files.

2. **Modularize with Procedures**:

   - Use internal or external procedures to encapsulate logic, stored in `.p` files.
   - **Example**:

     ```abl
     /* utilities.p */
     PROCEDURE formatCurrency:
         DEFINE INPUT PARAMETER dAmount AS DECIMAL NO-UNDO.
         DEFINE OUTPUT PARAMETER cFormatted AS CHARACTER NO-UNDO.
         cFormatted = STRING(dAmount, ">>>,>>9.99").
     END PROCEDURE.
     ```

     ```abl
     /* main.p */
     RUN utilities.p PERSISTENT SET hUtil.
     RUN formatCurrency IN hUtil (1234.56, OUTPUT cResult).
     DISPLAY cResult. /* Outputs: 1,234.56 */
     ```
   - **Reference**: Procedures.

3. **Use Persistent Procedures**:

   - Run external procedures with `PERSISTENT` to keep them in memory for repeated calls.
   - **Example** (as above, using `PERSISTENT SET hUtil`).
   - **Reference**: Persistent Procedures.

4. **Centralize Configuration**:

   - Store database connections, file paths, or settings in a single include file or procedure.
   - **Example**:

     ```abl
     /* config.i */
     DEFINE VARIABLE cDbName AS CHARACTER NO-UNDO INITIAL "sports2000".
     CONNECT VALUE(cDbName) -H "localhost" -S 1234.
     ```

5. **Version Control**:

   - Use version control systems (e.g., Git) to manage `.p`, `.i`, and `.w` files, ensuring reusable code is tracked and updated systematically.

### Common Mistakes

1. **Hardcoding Values**:

   - Embedding constants like file paths or database names in multiple files makes maintenance difficult.
   - **Mistake**:

     ```abl
     CONNECT "sports2000" -H "localhost" -S 1234.
     ```
   - **Fix**: Use a configuration include file (`config.i`).

2. **Duplicating Code**:

   - Repeating logic across files increases maintenance overhead.
   - **Mistake**:

     ```abl
     /* main1.p */
     OUTPUT TO "app.log" APPEND.
     PUT "Starting" SKIP.
     OUTPUT CLOSE.
     /* main2.p */
     OUTPUT TO "app.log" APPEND.
     PUT "Starting" SKIP.
     OUTPUT CLOSE.
     ```
   - **Fix**: Use an include file or procedure (`common.i`).

3. **Not Using Persistent Procedures**:

   - Repeatedly loading external procedures without `PERSISTENT` causes performance overhead.
   - **Mistake**:

     ```abl
     RUN utilities.p.
     RUN formatCurrency (1234.56, OUTPUT cResult).
     ```
   - **Fix**: Use `PERSISTENT SET`.

---

## 3. Importing Data

### Best Practices

1. **Use IMPORT Statement for Text Files**:

   - Import data from CSV or text files using `IMPORT` with proper delimiters.
   - **Example**:

     ```abl
     /* import-customers.p */
     DEFINE TEMP-TABLE ttCustomer
         FIELD custNum AS INTEGER
         FIELD custName AS CHARACTER
         FIELD balance AS DECIMAL.
     
     INPUT FROM "customers.csv".
     REPEAT:
         CREATE ttCustomer.
         IMPORT DELIMITER "," ttCustomer.
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
   - **Reference**: IMPORT Statement.

2. **Validate Input Data**:

   - Check for invalid or missing data before processing.
   - **Example**:

     ```abl
     IMPORT DELIMITER "," ttCustomer NO-ERROR.
     IF ERROR-STATUS:ERROR THEN
         MESSAGE "Invalid data at line " + STRING(INPUT LINE-NUMBER) VIEW-AS ALERT-BOX.
     ```

3. **Use Temp-Tables for Staging**:

   - Stage imported data in temp-tables to validate and transform before updating the database.
   - **Example** (as above, using `ttCustomer`).

4. **Handle Large Files Efficiently**:

   - Process files in batches using `REPEAT` or `FOR EACH` to avoid memory issues.
   - Use `BUFFER-COPY` for efficient data transfer to database tables.
   - **Reference**: BUFFER-COPY.

5. **Log Import Results**:

   - Log successes and errors to a file or table for auditing.
   - **Example**:

     ```abl
     OUTPUT TO "import.log" APPEND.
     PUT "Imported " + STRING(RECID(ttCustomer)) + " at " + STRING(NOW) SKIP.
     OUTPUT CLOSE.
     ```

### Common Mistakes

1. **Not Handling Errors**:

   - Failing to use `NO-ERROR` or validate input leads to crashes on bad data.
   - **Mistake**:

     ```abl
     IMPORT DELIMITER "," ttCustomer.
     ```
   - **Fix**: Add `NO-ERROR` and error checking.

2. **Ignoring Delimiters**:

   - Assuming default delimiters (space) when importing CSV files causes data misalignment.
   - **Mistake**:

     ```abl
     IMPORT ttCustomer. /* Expects spaces, not commas */
     ```
   - **Fix**: Specify `DELIMITER ","`.

3. **Direct Database Updates**:

   - Importing directly into database tables without validation risks data corruption.
   - **Mistake**:

     ```abl
     IMPORT customer.
     ```
   - **Fix**: Use temp-tables for staging.

---

## 4. External File Management

### Best Practices

1. **Use Dynamic File Paths**:

   - Store file paths in variables or include files to support portability.
   - **Example**:

     ```abl
     /* config.i */
     DEFINE VARIABLE cDataDir AS CHARACTER NO-UNDO INITIAL "C:/data/".
     ```

     ```abl
     INPUT FROM VALUE(cDataDir + "customers.csv").
     ```

2. **Check File Existence**:

   - Use `FILE-INFO` or `SEARCH` to verify files exist before accessing.
   - **Example**:

     ```abl
     FILE-INFO:FILE-NAME = "customers.csv".
     IF FILE-INFO:FULL-PATHNAME = ? THEN
         MESSAGE "File not found." VIEW-AS ALERT-BOX.
     ```
   - **Reference**: FILE-INFO Handle.

3. **Close Streams Properly**:

   - Always close `INPUT` and `OUTPUT` streams to prevent resource leaks.
   - **Example**:

     ```abl
     INPUT FROM "data.txt".
     REPEAT:
         IMPORT cLine.
     END.
     INPUT CLOSE.
     ```
   - **Reference**: INPUT FROM.

4. **Use Secure File Access**:

   - Validate file paths to prevent unauthorized access.
   - Use relative paths or environment variables for sensitive files.
   - **Reference**: Security Considerations.

5. **Backup Critical Files**:

   - Before modifying external files, create backups programmatically.
   - **Example**:

     ```abl
     COPY-LOB FROM FILE "data.txt" TO FILE "data.bak".
     ```

### Common Mistakes

1. **Hardcoding File Paths**:

   - Absolute paths break portability across environments.
   - **Mistake**:

     ```abl
     INPUT FROM "C:/data/customers.csv".
     ```
   - **Fix**: Use variables or include files.

2. **Not Closing Streams**:

   - Leaving `INPUT` or `OUTPUT` streams open causes resource leaks.
   - **Mistake**:

     ```abl
     INPUT FROM "data.txt".
     REPEAT:
         IMPORT cLine.
     END. /* Missing INPUT CLOSE */
     ```
   - **Fix**: Add `INPUT CLOSE`.

3. **Ignoring File Existence**:

   - Attempting to read non-existent files causes errors.
   - **Mistake**:

     ```abl
     INPUT FROM "data.txt".
     ```
   - **Fix**: Check with `FILE-INFO`.

---

## 5. General Best Practices

1. **Use Meaningful Names**:
   - Name variables, procedures, and files descriptively (e.g., `cCustomerName` instead of `c1`).
2. **Comment Code**:
   - Add comments to explain complex logic or reusable components.
   - **Example**:

     ```abl
     /* Updates customer balance to zero if negative */
     FOR EACH customer EXCLUSIVE-LOCK WHERE customer.balance < 0:
         customer.balance = 0.
     END.
     ```
3. **Test Incrementally**:
   - Use ABL Dojo or a development environment to test code snippets before deployment.
4. **Profile Performance**:
   - Use OpenEdge tools like ProTop or `PROFILER` to identify slow queries or procedures.
   - **Reference**: Performance Tuning.
5. **Follow Coding Standards**:
   - Adopt consistent indentation, naming conventions, and modular design as per Progress guidelines.

## 6. Common General Mistakes

1. **Not Using NO-UNDO**:
   - Omitting `NO-UNDO` for non-transactional variables increases overhead.
   - **Mistake**:

     ```abl
     DEFINE VARIABLE cTemp AS CHARACTER.
     ```
   - **Fix**: Add `NO-UNDO`.
2. **Overcomplicating Logic**:
   - Writing verbose code when ABL’s high-level statements suffice.
   - **Mistake**:

     ```abl
     DEFINE VARIABLE i AS INTEGER.
     DO i = 1 TO NUM-ENTRIES(cList):
         IF ENTRY(i, cList) = cValue THEN ...
     END.
     ```
   - **Fix**: Use `LOOKUP(cValue, cList)`.

---

## Example Program Combining Best Practices

This program demonstrates best practices for querying, file reuse, importing, and external file management.

```abl
/* CustomerImportAndReport.p */
/* Purpose: Import customers from CSV, update database, and generate report */

{config.i} /* Includes cDataDir, cDbName */
{common.i} /* Includes logMessage procedure */

/* Define temp-table for import */
DEFINE TEMP-TABLE ttCustomer
    FIELD custNum AS INTEGER
    FIELD custName AS CHARACTER
    FIELD balance AS DECIMAL.

/* Import data */
PROCEDURE importCustomers:
    DEFINE INPUT PARAMETER cFile AS CHARACTER NO-UNDO.
    FILE-INFO:FILE-NAME = cFile.
    IF FILE-INFO:FULL-PATHNAME = ? THEN DO:
        RUN logMessage("File " + cFile + " not found").
        RETURN.
    END.
    
    INPUT FROM VALUE(cFile).
    REPEAT:
        CREATE ttCustomer.
        IMPORT DELIMITER "," ttCustomer NO-ERROR.
        IF ERROR-STATUS:ERROR THEN DO:
            RUN logMessage("Invalid data at line " + STRING(INPUT LINE-NUMBER)).
            NEXT.
        END.
    END.
    INPUT CLOSE.
END PROCEDURE.

/* Update database */
PROCEDURE updateDatabase:
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
    RUN logMessage("Updated " + STRING(RECID(customer)) + " records").
END PROCEDURE.

/* Generate report */
DEFINE FRAME fReport
    customer.cust-num LABEL "ID"
    customer.name LABEL "Name"
    customer.balance LABEL "Balance"
    WITH 10 DOWN TITLE "Customer Report" CENTERED.

FOR EACH customer NO-LOCK WHERE customer.balance > 1000:
    DISPLAY
        customer.cust-num
        customer.name
        customer.balance
        WITH FRAME fReport.
END.

/* Main block */
RUN importCustomers(cDataDir + "customers.csv").
RUN updateDatabase.
```

### Explanation

- **File Reuse**: Uses `config.i` and `common.i` for shared settings and logging.
- **Database Query**: Uses `NO-LOCK` for reporting and `EXCLUSIVE-LOCK` (implicit in transaction) for updates.
- **Importing**: Imports CSV into a temp-table, validates data, and logs errors.
- **External Files**: Checks file existence and uses dynamic paths.
- **Error Handling**: Includes `NO-ERROR` and logging.
- **Modularity**: Separates logic into procedures.

---

## Additional Resources

- **Tutorials**: ABL Dojo for hands-on practice.
- **Community**: Progress Community for peer support and examples.
- **Whitepapers**: Download from www.progress.com for advanced topics like performance tuning.

This document covers the essentials for writing robust ABL code. For specific use cases or deeper dives into topics like object-oriented ABL or web integration, consult the referenced documentation or request tailored examples.

