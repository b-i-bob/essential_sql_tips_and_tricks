# How to deal with NULL values.

## What is `NULL`?

SQL uses `NULL` to represent an unknown value of any type.

```sql
SELECT A, B, A=B AS equals
FROM
(VALUES (1,1), (1,0), (NULL,1), (1,NULL), (NULL,NULL)) AS T(A,B)
```
*output:*
a|b|equals
---|---|---
1|1|true
1|0|false
NULL|1|NULL
1|NULL|NULL
NULL|NULL|NULL

You might expect `A=B` to return just `TRUE` and `FALSE`. However, equals in SQL returns `NULL` if either of the operands are `NULL`.
You need to decide what to do with `NULL`s if you want to treat them as ordinary two-valued logic `TRUE` or `FALSE`.

## Tip #1: Use `IS NULL` to explicitly test for `NULL` values

```sql
SELECT A, B, CASE
               WHEN A IS NULL AND B IS NULL THEN TRUE
               WHEN A IS NULL OR B IS NULL THEN FALSE
               ELSE A=B
              END as verbose
FROM (VALUES (1,1), (1,0), (NULL,1), (1,NULL), (NULL,NULL)) AS T(A,B)
```
*output:*
| a | b | verbose |
| :--- | :--- | :--- |
| 1 | 1 | true |
| 1 | 0 | false |
| NULL | 1 | false |
| 1 | NULL | false |
| NULL | NULL | true |

This spells it out pretty clearly but is verbose compared to the alternatives.

## Tip #2: Use `COALESCE()` to provide a default instead of NULL 

```sql
SELECT A, B, A=B AS equals, COALESCE(A=B, FALSE) AS coalesce_false
FROM (VALUES (1,1), (1,0), (NULL,1), (1,NULL), (NULL,NULL)) AS T(A,B)
```
*output:*
a|b|equals|coalesce_false
---|---|---|---
1|1|true|true
1|0|false|false
NULL|1|NULL|false
1|NULL|NULL|false
NULL|NULL|NULL|false

This is usually sufficient. If you want NULL=NULL to be `TRUE` you can do something fancier:
```sql
SELECT A, B, A=B AS equals, COALESCE(A=B, FALSE) AS coalesce_false,
       COALESCE(A=B, A IS NULL AND B IS NULL) AS coalesce_fancy
FROM (VALUES (1,1), (1,0), (NULL,1), (1,NULL), (NULL,NULL)) AS T(A,B)
```
*output:*
| a | b | equals | coalesce\_false | coalesce\_fancy |
| :--- | :--- | :--- | :--- | :--- |
| 1 | 1 | true | true | true |
| 1 | 0 | false | false | false |
| NULL | 1 | NULL | false | false |
| 1 | NULL | NULL | false | false |
| NULL | NULL | NULL | false | true |

## Tip #2: Use `IS NOT DISTINCT FROM` to treat NULL as a distinct value equal to itself

```sql
SELECT A, B, A IS NOT DISTINCT FROM B AS slick
FROM (VALUES (1,1), (1,0), (NULL,1), (1,NULL), (NULL,NULL)) AS T(A,B)
```
*output:*
| a | b | slick |
| :--- | :--- | :--- |
| 1 | 1 | true |
| 1 | 0 | false |
| NULL | 1 | false |
| 1 | NULL | false |
| NULL | NULL | true |

If `IS NOT DISTINCT FROM` is not availale in your SQL dialect use one of the other alternatives.
