# Query from more than one table.

You are in luck if you can query from just one table. Someone has already done the work to decide what each row in the table
represents, what each column represents, which columns can have NULL values, which columns have unique values, etc. 
For multi-table queries you'll need to specify how to combine the tables, two a time, 
using sub-selects, `JOIN`s, and `UNION`s. 

This will require you to know a few things: which tables to query, how the tables relate to each other, 
what you want each row in the query output to represent, how you want to deal
with missing values, inconsistencies, and duplicate values. 

This is the hard part. Ask for written documentation sometimes called a data dictionary. 
Find out who knows the data best and ask them. Ask to see existing queries you can learn from. 
Query the data to verify any assumptions. Oh and the data is changing all the time. Good luck!

If the data you seek does not exist you need to resolve additional issues: How to source that missing data. What
proxy data you can use instead. How to bring the data you need into a system you can query. How to clean and monitor the data.

Even when you succeed there will be additional business questions, 
questions about the accuracy and completeness of the data, 
and questions about how the data was handled. 

## How many matches will there be for each row: 0, 1, N?

Let's assume you have two tables to query and you think you know how they are related. 

Look at the data and query the data to verify the relationship.
How many rows from the first and second table will match each other? Do any matching columns contain NULLs?
Do the values in the columns which should match actually match eachother? Are there suspcicious values such as outliers?
This is called data profiling. Do this to up to the point where it allows you to make the decisions you need to make. 

Categorize the relationship bewteen the two tables by how many rows in the first table match how many rows in the second table. The shorthand for these relationships are: 1:1, 1:0..1, 1:N, N:1, and N:M. This knowledged can be vizualized by making an entity-relationship diagram. Now you know enough to proceed.

## Summary

Tables can be combined in a variety of ways. SQL gives you some constructs to build from. You are writing code in SQL using these constructs to specify the logic you want. Here is a summary:

| construct | match 0 rows | match 1 row | match N rows |
| :--- | :--- | :--- | :--- |
| sub-select | `NULL` value | 1 value | query error |
| INNER JOIN | no row | 1 row | N rows |
| LEFT OUTER JOIN | values from second table will be `NULL` | 1 row | N rows |
| RIGHT OUTER JOIN | values from first table will be `NULL` | 1 row | N rows |
| FULL OUTER JOIN | values from non-matching table will be `NULL` | 1 row | N rows |
| UNION ALL | N/A | N/A | N/A |
| UNION | N/A | N/A | N/A |

The `INNER` and `OUTER` keywords are optional. Use them for clarity. Omit them for brevity. It's a personal preference.

## Tip #1: Use a sub-select to get a single value from a single column from another table

```sql
SELECT U.username,
       (SELECT L.zipcode FROM locations AS L WHERE U.location_id = L.id) AS zipcode,
       (SELECT L.city FROM locations AS L WHERE U.location_id = L.id) AS city    
FROM users AS U
```
An error will be thrown if the sub-select returns more than 1 value. No value will be returned as `NULL`.

## Tip #2: Use a `JOIN` to get 1 or more columns from another table

```sql
SELECT U.username, L.zipcode, L.city
FROM users AS U
JOIN locations AS L ON U.location_id = L.id
```

A `JOIN` is different from a sub-select in two important ways: 
(1) Duplication. Having multiple matches to the second table will output multiple
rows, one for each match in the second table. 
(2) Missing rows. If there is no match the values for that row from the first will not be output. It will be dropped from the result set.

## Tip #3: Use a `LEFT JOIN` to retain every row in the first table.

```sql
SELECT U.username, L.zipcode, L.city
FROM users AS U
LEFT JOIN locations AS L ON U.location_id = L.id
```

Didn't like how the `JOIN` result was missing some rows from the first table? Then `LEFT JOIN` is for you. The rows that failed to join will have NULL for the second table columns. In this case all `username`s will appear in the result set although the `zipcode` and `city` may be NULL.

The order of the tables in the `LEFT JOIN` matters. The first table is the driving table and the rows in the second table are matched to it. Put it another way, there may be unmatched rows in the second table which do not appear in the result set.

## Tip #4: Don't use `RIGHT JOIN`.

Rewrite your query using a `LEFT JOIN`.

## Tip #5: Don't use `NATURAL JOIN`.

The database interprests a `NATURAL JOIN` the database follows some rules to guess how to join the tables. For readability, it is better to use another `JOIN` type and explicitly specify the `ON` clause you want.

## Tip #6: Use `FULL OUTER JOIN` when you need to retain all rows from both tables.

A `FULL OUTER JOIN` is used when you want all rows in both the first and second table to appear in the result set even when they match no rows in the other table. Think of this as a `UNION` of a `LEFT OUTER JOIN` and a `RIGHT OUTER JOIN`. Better yet, don't.

A way to combine two tables into one result set is `SELECT * from T1 FULL OUTER JOIN T2 ON FALSE`. These are despirate measures.

A `CROSS JOIN` IS a `FULL OUTER JOIN` without an `ON` clause. You can think of it as 
`SELECT * FROM T1 FULL OUTER JOIN T2 ON TRUE`. It crosses an N row table with an M row table yielding an N\*M row table. The result set can be huge and it is not usually what you want. All the other joins yield subsets of the `CROSS JOIN`.

## Tip #7 Use `UNION` or `UNION ALL` to stack two tables which have the same schema.

If you have twelve identical tables, one for each month, you can `UNION` them together to combine them into one virtual table.
`UNION` sorts the result set and remove duplicates. `UNION ALL` is faster. Use it when de-duplication is not needed.

