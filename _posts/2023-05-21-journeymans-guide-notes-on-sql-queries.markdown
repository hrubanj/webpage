---
layout: post
title:  "Journeyman's Notes on Optimizing SQL Queries"
date:   2023-05-10 21:00:00 +0000
categories: programming
---


```sql
select * from some_table
```

### Horses for courses (OLAP vs. OLTP)
Some databases will are not made for fetching data quickly, others are not made for
complex analytical queries. Do not use Snowlake for low-latency applications, and do not run nested aggregation queries on MariaDB.

### Understand your data
Your database might not infer frequencies of some values properly, but you can take them into account when designing queries.

### Use diagnostic tools
e.g. EXPLAIN in postgres

### WHERE to start
Always work with the least amount of data necessary. The WHERE clause is the firs place where (pun intended)
you limit the amount of data. Use it to the max.

### Subqueries
Can reduce the amount of data and speed up further processing. But the time that the subquery takes is often not worth it.

### Grouping vs. Window functions for deduplication
If you can, use grouping.

### Good design helps you avoid deduplication

### Keep your queries simple
Snowflake optimization engine.

### Use subtables in OLAP

### Memory vs. speed tradeoff

### Use correct datatypes
+ avoid unnecessary conversions

### Indexes
The treacherous nature of Snowflake indexes. Covered queries.

### Experiment


