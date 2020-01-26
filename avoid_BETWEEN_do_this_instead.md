# Avoid BETWEEN. Do this instead.

## Tip

How many events occurred in January? You may be tempted to write
```sql
SELECT count(*)
FROM events AS e
WHERE e.created_at BETWEEN '2019-01-01' AND '2019-01-31'
```

It is best to use code like this snippet which is always correct
```sql
SELECT count(*)
FROM events AS e
WHERE e.created_at >= '2019-01-01'
  AND e.created_at <  '2019-02-01'
```

## Why

Surprise! The first snippet may miss events on '2019-01-31' after midnight. It <b>will be</b> correct are when the e.created_at column type is `DATE`. Otherwise, you are gambling that only dates at midnight are present.

To avoid surprises, write code which is correct if it looks correct. `BETWEEN` can look correct but be incorrect. 

## Alternatives (not advised)

There are alternatives which require type casting or date manipulation functions. You will regret using these.

Adding time like `'2019-01-31 23:59:59'` is gambling that no events occur in the last second of the day.

Using the next date `'2019-02-01'` with BETWEEN is gambling that no events occur exactly at midnight on the next day.

Casting using `DATE(e.created_at`) or `e.created_at::DATE` is too easy to forget.

Truncating the column value using `DATE_TRUNC('month',e.created_at) = '2019-01-01'` is less efficient.

Converting the column value to characters `TO_CHAR(e.created_at,'YYYY-MM') = '2019-01'` is less efficient.
