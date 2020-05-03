# Perhaps You Need to Write A Multi-Level Query

The expressiveness of a simple SQL query `SELECT ... FROM ... JOIN ...` only goes so far. 
SQL expressions can easily combine the values in a single row.
For values not in the same row you need to bring them to where you can address them, usually as additonal columns in the same virtual table..
You may need a separate query to prepare values which require complex transformations.
Multi-level queries are useful when you can break down the calculation into separate steps.

## HOW
There are a few ways to create multi-level queries in SQL.

### Sub-select in the FROM or JOIN clause.
### Common table expressions.
### Build a view.
### Materialize an intermediate result set into a table.

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

Can this be simplified to a single-level query? Yes! There are no duplicates using this version. 
I think you'll agree that this is simplier than the first query.
Use one-level queries when you find them simplier to understand. There are fewer places for bugs to hide.
```sql
SELECT MAX(Y.date_of_birth) FROM users AS Y
```

Note: the database will aggregate the result without a `GROUP BY` clause when all the expressions are aggregates like COUNT(), SUM(), MAX(), etc..

### Tip #2: Who is the youngest user?

This multi-level query first computes the most recent birthday in a sub-select, like before. 
Then the outer query returns the name of any user with that birthday. 
The names of all the youngest users will be returned in case of a tie.
```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
```

If you just want one youngest user you need to break the potential tie and return just one.  
Here we break the time by using alphabetical order and the `LIMIT` clause.  
Some SQL systems use `SELECT TOP 1 ...` instead of or in addition to `LIMIT`.
Avoid non-deterministic queries. 
Always include an `ORDER BY` clause so that the query returns the same result when given the same source data. 

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
WHERE U.date_of_birth = (SELECT MAX(Y.date_of_birth) FROM users AS Y)
ORDER BY 1
LIMIT 1
```

Is there a way to get the same tie-broken result using a one-level query? Yes! By sorting. 
Here the `ORDER BY` does double-duty. 
It puts the latest birthdays first and as a nested sort puts those users with the same birthday in alphabetical order.

```sql
SELECT U.username, U.date_of_birth
FROM users AS U
ORDER BY 2 DESC, 1
LIMIT 1
```

Is there a way to get the same result with possible ties using a one-level query? Yes! Using window functions.

```sql
SELECT U.username, FIRST_VALUE(U.date_of_birth) OVER (ORDER BY U.date_of_birth DESC) AS date_of_birth
FROM users AS U
```

### Tip #3: What is the range of user ages?
### Tip #4: What is the birthday of the 2nd youngest user?
### Tip #5: How many users are younger than the average user?
