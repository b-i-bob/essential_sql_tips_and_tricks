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

If the data you seek does not exist you need to resolve additional issues: How to source that missing data or what
proxy data you can use. How to bring the data you need into a system you can query. How to clean and monitor the data.

Even when you succeed there will be additional business questions, 
questions about the accuracy and completeness of the data, 
and questions about how the data was handled. 

## How many matches will there be for each row: 0, 1, N?

Let's assume you have two tables to query and you know how they are related. 

Look at the data and query the data to verify the relationship.
How many rows from the first and second table will match each other? Do any matching columns contain NULLs?
Do the values in the columns which should match actually match eachother? Are there suspcicious values such as outliers?
This is called data profiling. Do this to the degree it allows you to make decisions you need to make. 
Although it can be fascinating, don't get lost in the data. You can come back and do more later.

Categorize the relationship bewteen the two tables by how many rows in the first table match how many rows in the second table.
The shorthand for these relationships are: 1:1, 1:0..1, 1:N, N:1, and N:M. Now you know enough to proceed.

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
(2) Missing rows. If there is no match the values for that row from the first will not be output.

## Tip #3: Summary

| method | match 0 rows | match 1 row | match N rows |
| :--- | :--- | :--- | :--- |
| sub-select | `NULL` value | 1 value | query error |
| INNER JOIN | no row | 1 row | N rows |
| LEFT OUTER JOIN | values from second table will be `NULL` | 1 row | N rows |
| RIGHT OUTER JOIN | values from first table will be `NULL` | 1 row | N rows |
| FULL OUTER JOIN | values from non-matching table will be `NULL` | 1 row | N rows |

The `INNER` and `OUTER` keywords are optional. Use them for clarity. Omit them for brevity. It's a personal preference.
A `CROSS JOIN` IS a `FULL OUTER JOIN` without an `ON` clause.



