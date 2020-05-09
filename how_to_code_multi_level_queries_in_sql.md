# How to code multi-level queries in SQL

There are a few ways to code multi-level queries in SQL. Each makes different tradeoffs in readability, control, and performance. 

### Use a sub-select in the SELECT clause.
```sql
SELECT TableA.x, 
       (SELECT TableB.y FROM TableB WHERE TableA.b_id = TableB.id) AS y
FROM TableA
```
*This is appropriate when you just need a single value from another table. Column "y" in this example.*

### Use a sub-select in the FROM OR JOIN clause.
```sql
SELECT TableA.x, SS_B.y, SS_B.z
FROM TableA
    LEFT JOIN (SELECT id, y, z FROM TableB) AS SS_B ON TableA.b_id = SS_B.id
```
*This is best when you need multiple values from another table or to make another pass to reshape the data.*

### Use a common table expression.
```sql
WITH CTE_B AS
    (
    SELECT id, y, z FROM TableB
    )
SELECT TableA.x, CTE_B.y, CTE_B.z
FROM TableA
JOIN CTE_B ON TableA.b_id = CTE_B.id
```
*This is best when you want to use the CTE more than once in the query or make a more readable query than the sub-select.*

### Create a view. Use the view.
```sql
CREATE OR REPLACE VIEW V_B AS
    (
    SELECT id, y, z FROM TableB
    )
    
SELECT TableA.x, V_B.y, V_B.z
FROM TableA
JOIN V_B ON TableA.b_id = V_B.id
```
*This is best when you want to centralize logic in one place (in the view) to use across multiple queries.*

The view is a persistent, named, stored query. It can be used most places a table can used. 

### Materialize a result set into a table. Use the table.
```sql
CREATE OR REPLACE TABLE T_B AS
    (
    SELECT id, y, z FROM TableB
    )
    
SELECT TableA.x AS XfromA, T_B.y, T_B.z
FROM TableA
JOIN T_B ON TableA.b_id = T_B.id
```
*This is best when a view is too slow to query and when the table can be refreshed often enough for the use case.*

### Anything else?
Yes! There are more elaborate ways to stage intermediate results in a database. I'll mention a few here.
- Create materialized views. These are like tables but have additional support from the DBMS for refreshing them.
- Create data outside the database with another tool say into a CSV file. Then load that CSV file into a table.
- Create data outside the database with another tool. Move the file to a filesystem accessible by the DBMS. Use the file directly as a foreign table without loading it.
- Create a federated table that queries another DBMS for the data.
- Create user defined functions (UDFs) to do complex or domain-specific transformations. There are also user defined aggregate functions (UDAFs).
- Create stored procedures which can take parameters and contain multiple queries and also `CREATE`, `INSERT`, `UPDATE`, `DELETE`, `TRUNCATE`, `DROP` statements. Stored procedures give greater control with loops and ifs and transactions.
- Use a BI tool to join, transform, or blend data after it is fetched from the database.
