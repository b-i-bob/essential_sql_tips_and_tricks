# Perhaps You Need to Write A Multi-Level Query

A single-level SQL query of the form `SELECT ... FROM ...` has limited expressive power. Combining multiple queries into one multi-level query expands the kinds of data transformations which can be performed. SQL permits as many levels as you need!

```sql
SELECT ... FROM
    (
    SELECT ... FROM
        (
        SELECT ... FROM A
        ) AS B
    ) AS C
```

This article will show that sometimes multi-level queries are necessary. But first lets look at how to code them.

## HOW
There are a few ways to code multi-level queries in SQL. Each makes different tradeoffs in readability, control, and performance. 

### Use a sub-select in the SELECT clause.
```sql
SELECT TableA.x, 
       (SELECT TableB.y FROM TableB WHERE TableA.b_id = TableB.id) AS y
FROM TableA
```
*This is appropriate when you just need a single value from another table. Y in this example.*

### Use a sub-select in the FROM OR JOIN clause.
```sql
SELECT TableA.x, SS_B.y, SS_B.z
FROM TableA
    LEFT JOIN (SELECT id, y, z FROM TableB) AS SS_B ON TableA.b_id = SS_B.id
```
*This is best when you need multiple values from another table.*

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
*This is best when you want to use the CTE more than once in the query. Some think it is more readable than the sub-select.*

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

The view is a named stored query. It can be used most places a table name can used. 

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

There are more elaborate ways to stage intermediate results in SQL but they are unfortuantely beyond the scope of this article.

## WHEN
Let's consider some mutli-level queries and see if they can be written more simply as single-level queries.

### Tip #1: Who is the youngest user?

This multi-level query first computes the most recent birthday in a sub-select. 
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

The multi-level query is necessary here because the window function FIRST_VALUE() cannot be directly used in the `WHERE` clause of the outer query. The Postgresql documentation says:
> Window functions are permitted only in the SELECT list and the ORDER BY clause of the query. They are forbidden elsewhere, such as in GROUP BY, HAVING and WHERE clauses. This is because they logically execute after the processing of those clauses. Also, window functions execute after regular aggregate functions.

Is there a way to get the same result using a single-level query? No.

If you just want one youngest user you need to break the potential tie and return just one name.  
Here we break the tie by using alphabetical order and the `LIMIT` clause.  
Some SQL systems use `SELECT TOP 1 ...` instead of or in addition to `LIMIT`.
An `ORDER BY` clause may be required so that the query returns the same result when given the same source data. 

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
ORDER BY 1
LIMIT 1
```

Is there a way to get the one user result using a one-level query? Yes! By sorting. 
Here the `ORDER BY` does double-duty. 
It puts the latest birthdays first and as a nested sort puts those users with the same birthday in alphabetical order.

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
ORDER BY 2 DESC, 1
LIMIT 1
```

### Tip #2: What is the name and birthday of the 2nd youngest users?

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
The innermost `SELECT` is finding the birthday of the youngest users. The `SELECT` in the middle is finding the birthday of the users who are second younges users. The outermost `SELECT` is returning the name and birthday of the second youngest users.

There is an alternative which uses window functions and one less `SELECT`.
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

### Tip #3: In which years were users born in every month?

The inner query gets 1 row for each year and month there was a birthday. The outer query selects the years in which 12 months have birthdays.
```sql
SELECT V.y
FROM
    (
    SELECT DATE_PART('year',U.date_of_birth) as y, DATE_PART('month',U.date_of_birth) as m
    FROM users AS U
    GROUP BY 1,2
    ) AS V
GROUP BY 1
HAVING COUNT(*) = 12
ORDER BY 1
```

Is there a way to get the same result using a single-level query? Yes! A `COUNT DISTINCT` in the `HAVING` clause replaces the inner query and the `COUNT`. Return the years which have 12 distinct birth months.

```sql
SELECT DATE_PART('year',U.date_of_birth) as y
FROM users AS U
GROUP BY 1
HAVING COUNT(DISTINCT DATE_PART('month',U.date_of_birth)) = 12
ORDER BY 1
```

### Tip #4: Which parents have multiples (twins, triplets, quadruplets, ...)?

The inner query counts the number of children born on the same day for each parent. 
The outer query shows only those parents and the birthday which has more than 1 child.

```sql
SELECT *
FROM
    (
    SELECT U.parent_id, U.date_of_birth as dbay, count(*) as people
    FROM users AS U
    GROUP BY 1,2
    )
WHERE people > 1
ORDER BY 1,2
```

We can get the same result with a single-level query.

```sql
SELECT U.parent_id, U.date_of_birth as dbay, count(*) as people
FROM users AS U
GROUP BY 1,2
HAVING COUNT(*) > 1
ORDER BY 1,2
```

The logic of this query is to show every birthday for every parent which has multiple children born that day. The condition to show only multiples and not single births is in the `HAVING` clause instead of the `WHERE` clause because it uses aggregate functions, the `COUNT(*)`. 

Its more likely that you will use this kind of query to detect if the same child has been erroneously entered more than once than to detect twins. 

### Tip #5: Which users are the multiples (twins, triplets, quadruplets, ...)?

A multi-level query is required to show the names of the twins.

```sql
SELECT U2.parent_id, U2.date_of_birth, U2.username
FROM users AS U2
JOIN (
    SELECT U.parent_id, U.date_of_birth as dbay, count(*) as people
    FROM users AS U
    GROUP BY 1,2
    HAVING COUNT(*) > 1
) AS M ON U2.parent_id = M.parent_id
ORDER BY 1,2,3
```

Another way to do it is with a window function. A multi-level query is still needed.

```sql
SELECT *
FROM (
         SELECT U.parent_id,
                U.date_of_birth,
                U.username
                count(*) OVER (PARTITION BY U.account_id, U.date_of_birth) AS multiples
         FROM users AS U
     ) AS M
WHERE multiples > 1
ORDER BY 1, 2, 3
```
### Tip #6: What is the distribution of the number of users born in any year?
