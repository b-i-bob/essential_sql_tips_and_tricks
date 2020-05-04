# Perhaps You Need to Write A Multi-Level Query

A simple SQL query of the form `SELECT ... FROM ...` only goes so far. Its expressive power can be expanded by writing a query where the `FROM` clause is another query. SQL permits as many levels as you need!

```sql
SELECT ... FROM
    (
    SELECT ... FROM
        (
        SELECT ... FROM A ...
        ) AS B
    ) AS C
```

## HOW
There are a few ways to create multi-level queries in SQL. Each makes different tradeoffs in readability, control, and performance to accomplish roughly the same thing. 

### Use a sub-select in the SELECT clause.
```sql
SELECT TableA.X sd XfromA, 
       (SELECT TableB.Y FROM TableB WHERE TableA.b_id = TableB.id) as YfromB
FROM TableA
```
*This is appropriate when you just need a single value from another table.*

### Use a sub-select in the FROM OR JOIN clause.
```sql
SELECT TableA.X sd XfromA, SS_B.YfromB
FROM TableA
    LEFT JOIN (SELECT TableB.id, TableB.Y as YfromB FROM TableB) AS SS_B ON TableA.b_id = SS_B.id
```
*This is best when you need multiple values from another table.*

### Use a common table expression.
```sql
WITH CTE_B AS
    (
    SELECT TableB.id, TableB.Y as YfromB FROM TableB
    )
SELECT TableA.X AS XfromA, CTE_B.YfromB
FROM TableA
JOIN CTE_B ON TableA.b_id = CTE_B.id
```
*This is best when you want to use the CTE more than once in the query. Some think it is more readable than the sub-select.*

### Create a view. Use the view.
```sql
CREATE OR REPLACE VIEW V_B AS
    (
    SELECT TableB.id, TableB.Y as YfromB FROM TableB
    )
    
SELECT TableA.X AS XfromA, V_B.YfromB
FROM TableA
JOIN V_B ON TableA.b_id = V_B.id
```
*This is best when you want to centralize logic in one place (the view) to use across multiple queries.*
The view is a named stored query. It can be used most places a table can used for convenience. 

### Materialize a result set into a table. Use the table.
```sql
CREATE OR REPLACE TABLE T_B AS
    (
    SELECT TableB.id, TableB.Y as YfromB FROM TableB
    )
    
SELECT TableA.X AS XfromA, T_B.YfromB
FROM TableA
JOIN T_B ON TableA.b_id = T_B.id
```
*This is best when a view is too slow to query and when the table can be refreshed often enough for the use case.*

There are even more elaborate ways to create a table. Some databases support materialied view creation which is similar to creating your own table. You can create temporary tables which are automatically deleted when the database session ends. You can incrementally maintain a table instead of rebulding it from scratch each time. You can compute data sets using other tools such as python and load the data into a table.

## WHEN
Let's consider some example mutli-level queries and see if they can be written more simply as single-level queries.

### Tip #1: What is the birthday of the youngest user?
This multi-level query first computes the most recent birthday in a sub-select. 
Then the outer query returns that birthday for every user with that birthday. 
The birthday will be repeated in case of a tie.
```sql
SELECT U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
```

If you just want to return one birthday you can use the `LIMIT` clause.  

```sql
SELECT U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
LIMIT 1
```

Can this be simplified to a single-level query? Yes! Just use the inner query. 
```sql
SELECT MAX(Y.date_of_birth) FROM users AS Y
```
I think you'll agree that this is simplier than the first query.
Use one-level queries when you find them simplier to understand. There are fewer places for bugs to hide.

Note: the database will aggregate the result without a `GROUP BY` clause when all the expressions are aggregates like COUNT(), SUM(), MAX(), etc..

### Tip #2: Who is the youngest user?

This multi-level query first computes the most recent birthday in a sub-select, like before. 
Then the outer query returns the name of any user who has that birthday. 
The names of all the youngest users will be returned in case of a tie.
```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
```

There is another alternative which uses window functions.
```sql
SELECT X.username, X.date_of_birth
FROM (
         SELECT U.username,
                U.date_of_birth,
                FIRST_VALUE(U.date_of_birth)
                    OVER (ORDER BY U.date_of_birth DESC
                         ) AS youngest_dob
         FROM users AS U
     ) AS X
WHERE X.date_of_birth = youngest_dob
```

If you just want one youngest user you need to break the potential tie and return just one.  
Here we break the time by using alphabetical order and the `LIMIT` clause.  
Some SQL systems use `SELECT TOP 1 ...` instead of or in addition to `LIMIT`.
Avoid non-deterministic queries. 
An `ORDER BY` clause may be required so that the query returns the same result when given the same source data. 

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
ORDER BY 1
LIMIT 1
```

Is there a way to get the same result using a one-level query? Yes! By sorting. 
Here the `ORDER BY` does double-duty. 
It puts the latest birthdays first and as a nested sort puts those users with the same birthday in alphabetical order.

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
ORDER BY 2 DESC, 1
LIMIT 1
```

Is there a way to get the same result with possible ties using a single-level query? No.

### Tip #3: What is the name and birthday of the 2nd youngest users?

Let me clarify by saying its the users who are youngest among those not born on the same day as the youngest users.

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth =
    (
    SELECT MAX(U.date_of_birth)
    FROM users AS U
    WHERE U.date_of_birth < (SELECT MAX(Y.date_of_birth) FROM users AS Y)
    )
```

There is another alternative which uses window functions.
```sql
SELECT X.username, X.date_of_birth
FROM (
         SELECT U.username,
                U.date_of_birth,
                NTH_VALUE(U.date_of_birth, 2)
                    OVER (ORDER BY U.date_of_birth DESC
                          RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                         ) AS second_youngest_dob
         FROM users AS U
     ) AS X
WHERE X.date_of_birth = second_youngest_dob
```

Is there a way to get the same result with possible ties using a single-level query? No.

### Tip #4: In which years were users born in every month?
### Tip #5: Which users are multiples (twins, triplets, quadruplets, ...)?
### Tip #6: Which users share their date of birth with another user?
### Tip #7: What is the distribution of the number of users born in any year?
