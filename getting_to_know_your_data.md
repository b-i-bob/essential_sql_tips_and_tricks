# Getting to know your data.

Before you can write a useful query you need to know the data you have. This is called *data profiling*.
There are tools to profile data. They tell you basic things about each column in each table. 
Here's how to do that using SQL. The advantage with SQL you can go beyond the basics.

### Tip #1: A database is not one giant spreadsheet.

Spreadsheets are great for small data sets. They become unwieldy for larger data sets. 
For complex data sets over 1 million rows you are going to need something the enforces more structure. 
You're going to need a database.

A relational database management system (RDBMS) manages data in named databases and named tables 
similar to speadsheet's workbooks and sheets. 
Other database objects include: views, materialized views, stored procedures, indexes, sequences, operations, and user-defined functions. 

Each database can be backed-up and restored independently ensuring the data in the collection of tables is consistent. 
Typical uses of separate databases are for data from different sources, with different access permissions, or different stage of development.

Each table has named columns whereas a spreadsheet has columns A,B,C,...AA,AB,AC,...etc.
Each table has rows which are unnamed and unordered. Often, one column is used to hold a unique sequence number for each row. 
Each table has a set of named columns. 
Each table typically represents a collection of one kind of entity: users, bank accounts, etc. Each row represents one user, one bank account, etc.
Each column holds one type of data. Data types can be an integer or real number or datetime or strings of characters. 
Advanced datatypes include JSON or XML or a vector or a buckets of bytes known as a BLOB.

A RDBMS has a standard query language (SQL) to get result sets with SELECT, 
manage rows of data with INSERT, UPDATE, and DELETE, 
and manage other objects with CREATE, ALTER, and DROP. 
SQL is a declarative language. 
You declare what you want in SQL and the RDBMS writes the code and executes it.

Many RDBMSes allow concurrent access by multiple users over multiple connections from multiple remote hosts.

This additional structure creates a flexible and efficent service to persist data at any scale.

The structure of a RDBMS is also its downside. 
If the content in the data changes frequently a fixed table structure is a poor fit.
A spreadsheet can be modified rapidly using direct manipulation, can be more user friendly in layout and format, 
can have different types data and formulas intermixed.

### Tip #2: What do you need to know about the data to write a query?

You need to know the inputs and outputs.

The details about inputs are which tables to query,
how the tables relate to each other,
whether there are missing values, inconsistencies, or duplicate values

This is the hard part. The source of most trouble is misunderstood, missing, or bad data.

You also need to know the outputs: what you want each row in the query output to represent and what data items you need.

If the data you seek does not exist you need to resolve additional issues: How to source that missing data. What
proxy data you can use instead. How to bring the data you need into a system you can query. How to clean and monitor the data.

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

L@@K at the table structure. What columns are there? Is there a unique row identifier, sometimes called `id` or `slug`? 
Are there columns that reference rows in other tables?

What are the values in a column? Are their near duplicates?
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

What are the most frequently occurring values in a column and as a percent of total:
```sql
SELECT columnname, 
       count(*) AS occurrances,
       ROUND(100*count(*)/sum(count(*)) over (),0) as pct
       FROM tablename 
GROUP BY 1
ORDER BY 2 DESC
```

How many NULL and non-NULL values are in a column and as a percent of total: 
```sql
SELECT columnname IS NOT NULL AS has_columnname,
       count(*) AS occurrances,
       ROUND(100*count(*)/sum(count(*)) over (),0) as pct
       FROM tablename 
GROUP BY 1
ORDER BY 1
```

When did the contents of a column begin to be populated?
```sql
SELECT columnname IS NOT NULL AS has_columnname,
       count(*) AS occurrances,
       ROUND(100*count(*)/sum(count(*)) over (),0) as pct,
       min(created_at) as earliest_create_at_date,
       max(created_at) as latest_create_at_date
       FROM tablename 
GROUP BY 1
ORDER BY 1
```

# Tip #5: 
