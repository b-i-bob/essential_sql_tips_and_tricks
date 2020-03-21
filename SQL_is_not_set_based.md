# SQL is not set-based

SQL is described as a set-based language. 
`JOIN`s are shown as Venn diagrams implying they are intersection operators. 
A SQL query produces a result *set*.

You will find yourself adding `DISTINCT` to guard against unexpected duplicates. 
Your code will be an inefficient and more complex than necessary.

## Tip #1

SQL is based on multi-sets, not sets. Tables can contain duplicate values.

## Tip #2

A `JOIN` operator is a generalization of the intersection operator which can deal with duplicate values.

##  Where do duplicates come from?

Consider this query:

```sql
SELECT a.col1, b.col2
FROM table_a AS a
JOIN table_b AS b ON a.x = b.y
```

A SQL engine will generate code equivalent to "nested loops":

```
for a_row in table_a
    for b_row in table_b
        if a_row.x == b_row.y
        then emit(a_row.col1, b_row.col2)
    end
 end
```
In other words, it will emit one row for every pair of matching rows.

This is where duplicate rows come from.
In the extreme case where the join condition is always true, 
the result is a projection of columns from every a_row paired with every b_row.
The size of that result is N*M rows.

In contrast, an intersection operator generates at most M or N rows. 
In some cases, a `JOIN` does function like an intersection operator.
This happens when each a_row is matched to at most one b_row. 
This is so common it leads people to believe it is always true.

### How do I remove duplicates?

Do **NOT** put `DISTINCT` in every query just in case. Know your data! Be intentional!

Find out which tables and columns have unique values (no duplicates) 
and which tables are related to eachother by 1:1 relationships. 
Constructs like COUNT(a.col1) are faster than COUNT(DISTINCT a.col1).

To remove unwanted duplicates, when necessary, use `SELECT DISTINCT ...` or a `GROUP BY ...` clause.

