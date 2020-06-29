# When the order of operations is important in SQL `SELECT`

##
The order of a select statement is shown below (graphic from 
@Apress "The Definitive Guide to SQLite").
![from where group by having select distinct order by limit](https://github.com/b-i-bob/essential_sql_tips_and_tricks/blob/master/Phases%20of%20Select%20in%20SQL%20Lite.jpg "Phases of Select in SQL")

The select statement is evaluated from the inside out starting with the `FROM` clause. The `FROM` clause specifies a table whose data is the input to process. Then each `JOIN` clause adds all the columns from another table. Then the `WHERE` clause filters out some rows. Then the `GROUP BY` clause aggregates rows with grouped by columns and aggregated values. Then the `HAVING` clause filters the rows based on aggregated values. Then, back to the beginning, the `SELECT` clause filters the columns and outputs them in the specified order. The `SELECT` can also transform values using calculations like `+` and functions like `LOWER()` or `SUM()`. Then `DISTINCT` is applied to remove duplicate rows. Almost done. Next, `ORDER BY` sorts the rows. Finally, `LIMIT` emits only the first N or a range of rows N to M.

The evaluation transforms the select statement into code (step by step instructions for the computer). When the code is executed it starts running the data from the `FROM` table through the joins-where-groupby-having-select-distinct-orderby-limit operations (in that order!). The output is your result set.

Often the order of evaluation does not matter. Let's see some examples when it does.

## Tip #1 GROUP BY cannot refer to column aliases defined with AS in the SELECT clause
SQL will not accept 
```sql
SELECT complex_calculation_here AS foo FROM table1 GROUP BY foo
```
The label `foo` is a column alias for a calculation. The alias appears as the name of the column in the header of the result set. This is really readable. However, SQL does not allow this. SQL evaluates the `GROUP BY` before it sees `foo` in the `SELECT` clause and does not look ahead. Instead you need to duplicate the complex calculation in the `GROUP BY` clause.
```sql

SELECT complex_calculation_here AS foo FROM table1 GROUP BY complex_calculation_here
```
Some SQL engines allow a `GROUP BY` to use a column *number* to avoid redundancies and reduce errors.
```sql
SELECT complex_calculation_here AS foo FROM table1 GROUP BY 1
```

A way to avoid repeating yourself which respects the SQL standard is by writing a nested query.
```sql
SELECT foo FROM
    (
    SELECT complex_calculation_here AS foo FROM table1
    ) AS X
GROUP BY foo
```

## Tip #2: Two clauses to filter results: WHERE and HAVING.
The `WHERE` clause is evaluated before the `GROUP BY`. `WHERE` filters rows before they are aggregated. The `HAVING` clause is evaluated after the `GROUP BY`. `HAVING` filters rows after they are aggregated. So if you want to filter the results of the `GROUP BY` the condition to filter by cannot be in the `WHERE` clause. SQL will not look ahead.

This example to show duplicate names will **not** work:
```sql
SELECT name, COUNT(*)
FROM table1
WHERE COUNT(*) > 1
GROUP BY name
```
It has to be
```sql
SELECT name, COUNT(*)
FROM table1
GROUP BY name
HAVING COUNT(*) > 1
```

### Conclusions
- To follow the logic of a query, read it from the inside out starting with FROM.
- Watch out for the SQL constructs which are order-sensitive.
