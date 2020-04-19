# Query from more than one table.

## Preparing to write multi-table queries

You have some idea of what you want in the result set. Be clear about what you want the grain of the result set to be. Is each row one user, one day, one group of users? The grain may be the same as or different from any of the base tables.

The next step is to focus on how to combine the tables. Combining tables in SQL happens before anything else when the query executes.

## How many matches will there be for each row?

Let's assume you have two base tables to query, the first with N rows and the second with M rows, and you know which columns you want to use to relate the tables. 

A relationship between two tables can be characterized by the of number rows in the second table which will match each row in the first table. 

The interesting matching possibilities are:

relationship | matching rows | min rows | max rows | missing rows? | duplicate rows?
--- | --- | --- | --- | --- | ---
N:1 | exactly 1 row | N | N | No | No
N:0..1 | at most 1 row | 0 | N | Maybe | No
N:M | any number of rows | 0 | N\*M | Maybe | Maybe

Notice that the type of relationship bounds the range of rows you get. 

In the N:1 case there is never missing data nor are there ever duplications. In the N:0..1 cases you may not match some rows but no duplicates will be formed. In the N:M case you may need to deal with missing data or duplications resulting from combining the tables.

What if you are not sure which columns relate the two tables? Look at the data in both tables, make a guess or phone a friend, and query the data to verify the data relationship. See the note on [data profiling](https://github.com/b-i-bob/essential_sql_tips_and_tricks/blob/master/getting_to_know_your_data.md).

What if the data does not have a simple relationship due to missing rows, inconsistencies, invalid values, or have mismatched types? See the note on [cleaning data](ADD LINK HERE).

## SQL constructs for combining data from two tables.

Sub-selects, `JOIN`s, and `UNION`s are the SQL building blocks for combining tables. They give you different ways to control for missing data and duplications in the result set.

Here is a summary of those SQL constructs. 

| construct | match 0 rows | match 1 row | match N rows |
| :--- | :--- | :--- | :--- |
| sub-select | `NULL` value | 1 value | query error |
| INNER JOIN | no row | 1 row | N rows |
| LEFT OUTER JOIN | 1 row with `NULL` values from the second table | 1 row | N rows |
| RIGHT OUTER JOIN | 1 row with `NULL` values from the first table | 1 row | N rows |
| FULL OUTER JOIN | 1 row with `NULL` values from the non-matching table | 1 row | N rows |
| UNION ALL | N/A | N/A | N/A |
| UNION | N/A | N/A | N/A |

Note: The `INNER` and `OUTER` keywords are optional. Use them for clarity. Omit them for brevity. It's a personal preference.

The following tips show how these constructs work and when to use them.

## Tip #1: Use a sub-select to get a single value from a single column from another table

```sql
SELECT U.username,
       (SELECT L.zipcode FROM locations AS L WHERE U.location_id = L.id) AS zipcode,
       (SELECT L.city FROM locations AS L WHERE U.location_id = L.id) AS city    
FROM users AS U
```
In this example the zipcode and the city are retrieved from the locations table using sub-selects in the select clause. The sub-selects are written as select statements inside parentheses. They need an `AS` clause to specify a column name. The `SELECT` clause in the sub-select chooses one value. The `WHERE` clause in the sub-select specifies how to find the rows in the second table and usually refers to some columns in the first table. 

Sub-selects are expected to return a single value, perfect for this use case. The relationship used here is the location_id in the users table matches an id in the locations table. An error will be thrown if a sub-select returns more than 1 value. A `NULL` will be returned for rows where the sub-select does not find any values.

## Tip #2: Use a `JOIN` to get 1 or more columns from another table

```sql
SELECT U.username, L.zipcode, L.city
FROM users AS U
JOIN locations AS L ON U.location_id = L.id
```
This is another way to get values from another table. It uses the same relationship between the users and locations tables.

A `JOIN` is different from a sub-select in two important ways: 
(1) Duplication. Having multiple matches to the second table will output multiple
rows, one for each match in the second table duplicating the values in the first table. 
(2) Missing rows. If there is no match the values for that row from the first will not be output. It will be dropped from the result set.

## Tip #3: Use a `LEFT JOIN` to retain every row, avoid missing rows, from the first table.

```sql
SELECT U.username, L.zipcode, L.city
FROM users AS U
LEFT JOIN locations AS L ON U.location_id = L.id
```

The rows in the first table that fail to `LEFT JOIN` to any rows in the second table will return NULL for any column in the second table. Put it another way, all rows from the first table will appear in the result set; unmatched rows in the second table will **not** appear in the result set. In this case all `username`s will appear in the result set although the `zipcode` and `city` may be NULL.

The order of the tables in the `LEFT JOIN` matters. The first table is the base table and the rows in the second table are matched to it. 

## Tip #4: Don't use `RIGHT JOIN`.

Rewrite your query using a `LEFT JOIN`.

## Tip #5: Don't use `NATURAL JOIN`.

The database processes a `NATURAL JOIN` by guessing which columns to use to join the tables. For readability, it is better to use another `JOIN` type and explicitly specify the columns to join in the `ON` clause.

## Tip #6: Use `FULL OUTER JOIN` when you need to retain all rows from both tables.

A `FULL OUTER JOIN` is used when you want all rows in both the first and second table to appear in the result set even when they match no rows in the other table. Think of this as a `UNION` of a `LEFT OUTER JOIN` and a `RIGHT OUTER JOIN`. Better yet, don't.

A `CROSS JOIN` IS a `FULL OUTER JOIN` without an `ON` clause. You can think of it as 
`SELECT * FROM T1 FULL OUTER JOIN T2 ON TRUE`. It crosses an N row table with an M row table yielding an N\*M row table. The result set can be huge and it is not usually what you want. All the other joins yield subsets of the `CROSS JOIN`.

## Tip #7: Use `UNION` or `UNION ALL` to stack two tables which have the same columns.

If you have twelve identical tables, one for each month, you can `UNION` them together to combine them into one virtual table.
`UNION` sorts the result set and remove duplicates. `UNION ALL` is faster becasue it avoids sorting. Use it when de-duplication is not needed.

## Tip #8: Some queries require a self-join.

A self-join is using a join from one table to itself. An example would help:

This selects all child and parent name pairs
```sql
SELECT C.username AS child, P.username AS parent
FROM users AS C
JOIN users AS P ON C.parent_id = P.id
```

Another case is when you want to compare rows to some other rows. This selects only the oldest child for each parent. 

```sql
SELECT C.username AS oldest_child, P.username AS parent
FROM users AS C
JOIN users AS P ON C.parent_id = P.id
LEFT JOIN users AS OLDER ON C.parent_id = OLDER.parent_id AND C.date_of_birth > OLDER.date_of_birth
WHERE OLDER.id IS NULL
```
It works by filtering out any children who have older siblings. 

# Tip #9: Deduplicating result sets.

One simple way to remove duplicates is to use `DISTINCT`. This example shows which birthdays are in the users table.

```sql
SELECT DISTINCT C.date_of_birth
FROM users AS C
```

Another way is to `GROUP BY` every column in the `SELECT` clause. The `ORDER BY` clause is optional.

```sql
SELECT C.date_of_birth
FROM users AS C
GROUP BY 1
```

One advantage of the `GROUP BY` solution is that you can create some aggreated measures at the same time. This example shows the number of user who where born on the same day. This `ORDER BY` clause puts the birthday shared by the most users at the top of the result set. Without the `ORDER BY` clause you get what you get.

```sql
SELECT C.date_of_birth, count(*) as number_of_users
FROM users AS C
GROUP BY 1
ORDER BY 2 DESC
```

# Tip #10: Why create such a confusing way to combine data?

Partly, I blame it on Codd. He invented this relational algebra which was turned into the SQL we know today. He was a mathematician. Can you tell? 

The other part is flexibility and control. There are many potential ways to combine data from two tables which yield different result sets. The DBMS does not guess how you want to combine tables each time. You need to tell it each and every time as part of the query.

SQL has many advantages. It is highly expressive and compact. SQL can be quickly and automatically translated into fast, parallel code optimized to the size and content of the data sets. It has saved many person-years of writing tedious code. It has withstood the test of time as a standard. Many companies and universities continue to make investments in SQL keeping it useful for the foreseeable future. It is a technical marvel well worth your time learning.


