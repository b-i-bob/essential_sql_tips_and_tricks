# How to deal with NULL values.

## What is `NULL`?

SQL uses `NULL` to represent an unknown value. Once a value is `NULL` any comparison using =, &lt;, &gt;, etc. results in NULL. 

```sql
SELECT 1=1, 1=0, NULL=1, 1=NULL, NULL=NULL
```
*output: true	false	NULL	NULL	NULL*

You might expect this to return just `TRUE` and `FALSE` but equals in SQL returns `NULL` if either of the operands are `NULL`.
You need to decide what to do with NULLs if you want to treat them as ordinary `TRUE` or `FALSE`.

## Tip #1: Use `IS NULL` to explicitly test for `NULL` values

```sql
SELECT CASE WHEN Col1 IS NULL OR Col2 IS NULL THEN FALSE
            ELSE Col1=Col2
            END
FROM TABLE A
```
This spells it out pretty clearly but is verbose compared to the alternatives.

## Tip #2: Use `COALESCE()` to provide a default instead of NULL 

```sql
SELECT COALESCE(Col1=Col2, FALSE)
FROM TABLE A
```

```sql
SELECT COALESCE(1=1, FALSE)
```

## Tip #2: Use `IS NOT DISTRICT FROM` to treat NULL as a distinct value equal to itself

```sql
SELECT Col1 IS NOT DISTINCT FROM Col2
FROM TABLE A
```
