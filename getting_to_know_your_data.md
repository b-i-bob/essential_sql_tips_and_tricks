# Getting to know your data.

Before you can write a useful query you need to know the data you have. This is called *data profiling*.
There are tools to profile data. They tell you basic things about each column in each table. 
Here's how to do that using SQL. The advantage with SQL you can go beyond the basics.

### Tip #1: A database is not a spreadsheet.

Spreadsheets are great for small data sets. They become unwieldy for larger data sets. 
For data sets over 1 million rows you are going to need something the enforces more structure. 
You're going to need a database.

A relational database management system (RDBMS) manages data in named databases and named tables 
corresponding to a spreadsheet's workbooks and sheets. 
Other database objects include: views, materialized views, stored procedures, indexes, sequences, operations, and user-defined functions. 

Separate databases typically are for data different stages of development or access permissions or under control of different departments.

Separate tables typically represents a collection of one kind of entity: users, bank accounts, etc. This discipline means your queries will often need to join the data squirreled away in multiple tables.

A table has named columns whereas a spreadsheet has columns named A,B,C,... AA,AB,AC,... ZZ.

Each column in a table holds one type of data whereas a spreadsheet allows any values anywhere. 
Column data types can be an integer or real number or datetime or string of characters. 
Advanced data types include JSON, XML, a vector to hold multiple values, geographic points, etc. A column cannot contain another table. A column can contain a value which can be used to relate that row to rows in other tables.

Tables have a changing number of rows as data is inserted and deleted. 
To uniquely identify each row the contents of one or more columns act as a so called primary key.
Each row represents one user, one bank account, one observation of an event, etc. 

A RDBMS has a standard query language (SQL) to get result sets with SELECT, 
manage rows of data with INSERT, UPDATE, and DELETE, 
and manage other objects with CREATE, ALTER, and DROP. 
SQL is a declarative written language. 
You declare what you want in SQL and the RDBMS writes the machine code and executes it.

Many RDBMSes allow concurrent access by multiple users over multiple connections from multiple remote hosts.

This additional structure creates a flexible and efficent service to persist data at any scale.

The structure of a RDBMS is also its downside. 
If the elements in the data change frequently a fixed table is a poor fit.
A spreadsheet has a graphical user interface which can be more user friendly in layout and for direct manipulation. 
A spreadsheet can have different types data and formulas intermixed willy-nilly.

### Tip #2: What do you need to know about the data to write a query?

You need to know the inputs and outputs.

Your inputs will be sourced from database tables. You'll need to know which tables to query,
how the tables relate to each other,
whether there are missing values, inconsistencies, or duplicate values. This is the hard part. The source of most trouble is misunderstood, missing, or bad data.

You also need to know the outputs: what you want each row in the query output to represent and what data items you need. If you have incomplete information about your requirements, proceed with what you know and iterate.

## Tip #3: How to discover the contents of a database?

Apply the Pareto Principle to find the 20% of the tables in a database which will be queried 80% of the time.
- Ask for written documentation, sometimes called a data dictionary. 
- Find out who knows the data best. Ask them. 
- Ask to see existing queries you can learn from. 
- Query the data to verify any assumptions.

## Tip #4: How to profile the contents of a database table?

L@@k at the data:
```sql
SELECT * FROM tablename
```

L@@K at the table structure. What columns are there? Is there a unique row identifier? 
Are there columns that reference rows in other tables?

What are the values in a column? Are they what you expect? Are there typos?
```sql
SELECT DISTINCT columnname FROM tablename
```

What is the distribution of values in a column:
What are the values in a column:
```sql
SELECT columnname, count(*) FROM tablename GROUP BY 1 ORDER BY 1
```

What are the most frequently occurring values in a column:
```sql
SELECT columnname, count(*) FROM tablename GROUP BY 1 ORDER BY 2 DESC
```

What are the most frequently occurring values in a column, as a percent of total, as a bar chart, and as a cumulative percent of total:
```sql
SELECT columnname, 
       count(*) AS occurrences,
       ROUND(100*count(*)/sum(count(*)) over (),0) AS pct,
       REPEAT('#'::TEXT, (50*count(*)/sum(count(*)) over ())::INT) AS bar_chart,
       ROUND(100*sum(count(*)) over (order by count(*) desc) / sum(count(*)) over (),0) AS cume_pct
       FROM tablename 
GROUP BY 1
ORDER BY 2 DESC
```

How many NULL and non-NULL values are in a column and as a percent of total: 
```sql
SELECT columnname IS NOT NULL AS has_columnname,
       count(*) AS occurrences,
       ROUND(100*count(*)/sum(count(*)) over (),0) AS pct
       FROM tablename 
GROUP BY 1
ORDER BY 1
```

When did the contents of a column begin to be populated?
```sql
SELECT columnname IS NOT NULL AS has_columnname,
       count(*) AS occurrences,
       ROUND(100*count(*)/sum(count(*)) over (),0) AS pct,
       min(created_at) as earliest_create_at_date,
       max(created_at) as latest_create_at_date
       FROM tablename 
GROUP BY 1
ORDER BY 1
```

# Tip #5: When to stop looking at data?

Profile data up to the point where you know enough to make the decisions you need to make. 

# Tip #6: Do not plan to have a column for each month's sales

Take advantage of the asymmetry between rows (easy to insert and delete) and columns (harder to insert and delete, essentially fixed). The columns names act as an interface to processes which use the table. Minimizing the changes to the columns will minimize changes to all processes which use the table. 

Because planning spreadsheets have columns for each month (January 2020, February 2020, March 2020, etc.) it seems logical that the database table should have those same columns. But the next year 12 additional columns would be needed. All processes which use this data would need to be updated or need to be much more dynamic. This is possible, but goes against the grain.

If instead the columns are sales_month and sales_amount a new row can be added each month. For human consumption data can be pivoted to show monthly columns after retrieval from the database.

