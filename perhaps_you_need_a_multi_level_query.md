# Perhaps You Need to Write A Multi-Level Query

A single-level SQL query of the form `SELECT ... FROM ...` has limited expressive power. [Multi-level queries](https://github.com/b-i-bob/essential_sql_tips_and_tricks/tree/master) expand the kinds of data transformations which can be performed. SQL permits as many levels as you need!

```sql
SELECT ... FROM
    (
    SELECT ... FROM
        (
        SELECT ... FROM A
        ) AS B
    ) AS C
```

Let's consider some multi-level queries and see if they can be written more simply as single-level queries.

### Tip #1: Who is the youngest user?

This multi-level query first computes the most recent birthday in a sub-select. 
That is the birthday of the youngest users.
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
Some SQL dialects use the syntax `SELECT TOP 1 ...` instead of `... LIMIT 1`.
An `ORDER BY` clause may be required so that the query always returns the same result when given the same source data. 

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
ORDER BY 1
LIMIT 1
```

Is there a way to get the one user result using a single-level query? Yes! 
Here the `ORDER BY` does double-duty. 
It puts the latest birthdays first and as a nested sort puts those users with the same birthday in alphabetical order.

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
ORDER BY 2 DESC, 1
LIMIT 1
```

### Tip #2: Who are the 2nd youngest users?

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
The innermost `SELECT` is finding the birthday of the youngest users. The `SELECT` in the middle is finding the birthday of the users who are second youngest users. The outermost `SELECT` is returning the name and birthday of the second youngest users.

There is an alternative which uses a window function and one less `SELECT`.
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

Is there a way to get the same result using a single-level query? Yes! Use a `COUNT DISTINCT` in the `HAVING` clause.

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

The logic of this query is group together the users who qualify as multiples (same birthday and same parent). The condition to show only multiples is in the `HAVING` clause instead of the `WHERE` clause because it uses an aggregate function, `COUNT(*)`. 

Its more likely that you will use this kind of query to detect if the same user has been erroneously entered multiple times rather than to detect twins. 

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
                U.username,
                count(*) OVER (PARTITION BY U.parent_id, U.date_of_birth) AS multiples
         FROM users AS U
     ) AS M
WHERE multiples > 1
ORDER BY 1, 2, 3
```

What's happening is that the window function, an aggreagte function with the `OVER` clause, does a `COUNT` over all users with the same parent and birthday. It adds that value we are calling `multiples` to every row. This is a little handier than the `GROUP BY` because extra information can be carried along with each row, here the username. The outer query adds a filter to return only the uses which are multiples.

### Tip #6: How many years have 1 user, 2 users, 3 users?

This is a histogram of the number of users within 1-year-wide bins. It requires a multi-level query.

The inner query counts the number of users for each year while the outer query counts the years for each number of users.

```sql
SELECT C.num_users, count(*) as years
from (
         SELECT DATE_PART('year', U.date_of_birth) AS y, count(*) AS num_users
         FROM users AS U
         WHERE U.date_of_birth IS NOT NULL
         GROUP BY 1
     ) AS C
GROUP BY 1
ORDER BY 1
```

# Conclusion

Sometimes a multi-level query can be avoided with a `LIMIT`, `HAVING`, `ORDER BY`, `DISTINCT` or some clever function of a function. However, a straightforward multi-level query is often simplest and may perform best. Expect to write mutli-level queries when single-level queries are insufficiently expressive.
