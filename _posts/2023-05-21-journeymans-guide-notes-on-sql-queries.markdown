---
layout: post
title:  "Journeyman's Notes on Optimizing SQL Queries"
date:   2023-05-10 21:00:00 +0000
categories: programming
---

Whether you are running a webpage, an analytic platform, or a data processing job, how you query your database crucially impacts how your application performs.
In this article, we will look at some examples of optimization techniques that I use when database performance becomes an issue.
I certainly don't know all the tricks out there. We will cover those that actually helped me solve problems in production applications in my several years of data engineering. 
For now, we will focus only on tabular databases that support SQL as they are the most common group. 
If you want to run the examples yourself, go to this (TODO: add link) repository and follow the README.

We limited the scope of this post to 'tabular databases that support SQL'. Such a long-winded definition might seem unnecessary, so let's unpack it.
Tabular databases are databases that store data in tables. SQL support means that we can use standard SQL (ANSI???) to query them.
This is not the same thing as relational databases—for example Snowflake falls under our definition, but it does not let you define relations. Generally, databases for
analytics tend to use SQL and be tabular but non-relational. On the other hand, several modern databases support a subset of SQL syntax but are not tabular (e.g. CosmosDB).
Yet another group uses tables as an underlying storage but does not support SQL queries (e.g. EdgeDB).

### Premature optimization

### Horses for courses (OLAP vs. OLTP)
Are you working with the right type of database?
Even though, you can just copy-paste most of your Postgres queries and run them in Snowflake, these databases are useful for entirely different purposes.
While Postgres is optimized for quickly fetching individual rows, Snowflake shows its strength on large data aggregations.
On the flipside, you probably shouldn't build a data lake on Postgres, and you should definitely avoid using Snowflake for low-latency applications.
I used to joke that every query takes a couple of seconds in Snowflake—no matter whether you are retrieving one row or performing an aggregation over millions of rows.
Of course, it works up to a point, once you get to really large table, even Snowflake queries take minutes or hours.
More generally, Postgres is an online transactional processing (OLTP) database, and Snowflake is an online analytics processing (OLAP) database.
Use OLTP if you need low latency, and OLAP if you need heavy aggregations.

TODO: examples

Our examples will work either with Postgres or DuckDB. I am planning to add more database examples in the database playground repository in the future.
Most techniques we will cover will be useful for both OLTP and OLAP. In some cases, we will see that they work only on one type of database.
This is usually because OLAP database try to be smarter—since they don't intend respond within milliseconds, they can take more time to optimize query execution.
The positive of this is that a poorly written query can perform well on OLAP. The negative is that finetuning the query might not always work because the query 
optimizer might translate it to the same plan as the suboptimal one. After all, SQL is a declarative language (TODO: expand).


### WHERE to start
Start by fiddling with the `WHERE` clause. The biggest improvements in query performance I have ever seen were resulted from, often small, changes to filtering clause.
Lets look at an example query.
```sql
select 
    u.name,
    count(*) / extract epoch from (max(p.time_created) - min(p.time_created))) as posts_per_second_active
from post p
inner join user u on p.user_id = u.id
where lower(u.name) like '%helmut%' 
  and p.time_created is not null 
group by u.name
having max(p.time_created) > '2023-01-01'
and max(p.time_created) != min(p.time_created)
order by time_per_post
limit 10
;
```
SQL query engines evaluate query parts in the following *logical* order:
- `FROM` and `JOIN` - determining the source data
- `WHERE` - filtering
- `GROUP BY` - splitting into groups
- `HAVING` - filtering aggregated results
- `ORDER BY` - sorting
- `LIMIT` - limiting output rows

The actual order in which the SQL engine evaluates clauses may differ, but it must ensure that if you
write your query as if it used the logical order, it will not fail due to different execution order.
For example, we cannot get a zero division error in the query above even if the query engine decides
to calculate `posts_per_second_active` before removing rows where `max(p.time_created) = min(p.time_created)`.

In reality, the filtering (`WHERE` clause), or parts of it, might get executed even before tables are joined.
`WHERE` clause reduces the amount of data we are working with from the start, and all subsequent steps have
easier job if they work with less data.

NB: where clause that duplicates condition already included in an inner join might actually slow the query down.


### Understand your data
Databases make assumptions about your data. Tons of smart people have spent years optimizing and testing databases
to make sure that these assumptions are accurate. But you can still do better than them.
While a database might guess that a join will produce something between 1 and 30 million rows, you might know that it will
be exactly one million. Or you might know that a filter removes 99 % rows in a table, so you might want to push it to a subquery.
Or you might know that a table includes duplicate join keys, so, you deduplicate it before joining.

### Use diagnostic tools
Databases usually have a way to show you how they decided to execute your query, i.e. display the query plan. Studying it can show you
what the bottlenecks are. When I am optimizing a query, it is usually a back and forth between checking the plan and tweaking the query.
I think execution plan visualizers are of great help here. While I might struggle to find the source of issues in the textual plan, it is 
often clear as mountain water from the first look at the picture.
Some database providers embed visualizers to their service, but there are also free versions, e.g. for Postgres.

Use visualizers if available.
e.g. EXPLAIN in postgres


### Subqueries
Can reduce the amount of data and speed up further processing. But the time that the subquery takes is often not worth it.

### Grouping vs. Window functions for deduplication
If you can, use grouping.

### Good design helps you avoid deduplication

### Database structure
table + table_advanced

### Keep your queries simple
Snowflake optimization engine.

### Use subtables in OLAP (+ Materialized views)

### Memory vs. speed tradeoff

### Use correct datatypes
+ avoid unnecessary conversions

### Indexes, clustering keys
The treacherous nature of Snowflake indexes. Covered queries. Index hinting. Too many indexes.

### Hinting
Sometimes, you might boost query performance by tweaks that make no apparent sense. Let's see how I made a query almost two hundred times slower
by removing an unnecessary join.
The query was supposed to extract the first content block of articles published on the website (TODO: possibly reproduce) and it looked like this:
```sql
select e2.id,
       mac.field_module_text,
       e3.handle
from matrixcontent_articlecontent mac
         inner join elements e on e.id = mac."elementId"
         inner join matrixblocks mb on e.id = mb."id"
         inner join entries e2 on mb."ownerId" = e2.id
         inner join entrytypes e3 on e2."typeId" = e3.id
where e."dateDeleted" is null
  and e.enabled
  and not e.archived
  and mb."sortOrder" = 1 -- the first article block
  and e2."postDate" is not null
;
```
It was a subquery in a larger query, but I will not copy the whole thing here as it would be hard to see the important parts.

While refactoring the code, I realized that the `handle` column is not used anywhere, so, I removed it. The output no longer included any data from the `entrytypes` table.
You might think that the `entrytypes` table still did some filtering since we are performing inner join. But it does not filter anything, even though you cannot see it from the query alone.
I am joining `entrytypes` on `entries` on `entries.typeId` = `entrytypes.id`. `entries.typeId` is a non-nullable column with a foreign key to `entrytypes.id`, and
`entrytypes.id` is a primary key. Hence, if I remove the excessive join, the output will be identical.

That is what I did hoping to see some performance improvement. Now, the sub-query looked like this:
```sql
select e2.id,
       mac.field_module_text
from matrixcontent_articlecontent mac
         inner join elements e on e.id = mac."elementId"
         inner join matrixblocks mb on e.id = mb."id"
         inner join entries e2 on mb."ownerId" = e2.id
where e."dateDeleted" is null
  and e.enabled
  and not e.archived
  and mb."sortOrder" = 1 -- the first article block
  and e2."postDate" is not null
;
```
The execution time went from roughly 300 ms to 51 s. I was not happy.
When I kept the join there, the performance remained similar to the original.
I took the subquery out and started looking into it, but then came another surprise—in isolation, the execution time did not worsen without the join.
It turned out that when it was a part of a larger query, the join provided some kind of hint to the query optimized, and it chose a better plan.
TODO: dissect query plan
So, I ended up with seemingly non-sensical query:
```sql
select e2.id,
       mac.field_module_text
from matrixcontent_articlecontent mac
         inner join elements e on e.id = mac."elementId"
         inner join matrixblocks mb on e.id = mb."id"
         inner join entries e2 on mb."ownerId" = e2.id
         inner join entrytypes e3 on e2."typeId" = e3.id -- does not filter anything, but hints query planner
where e."dateDeleted" is null
  and e.enabled
  and not e.archived
  and mb."sortOrder" = 1 -- the first article block
  and e2."postDate" is not null
;
```
I am not suggesting that you should go crazy adding unnecessary joins to your queries and hope that it will speed them up.
But you should be careful when optimizing, because query planner can sometimes get hints from things that almost seem like a programmer' mistake.



### Question and experiment
Linear thinking won't always get you there. Most of the time, optimizing query performance is a rigorous process—you rearrange clauses, trim off unnecessary data,
choose the most efficient functions, measure changes, and repeat.
But sometimes, queries will be slow, no matter how much love and care you give them.

Then, you have to think more broadly. Can you get the data from somewhere else, split the query into parts, or redefine the database structure?
Do you actually need to return all data in this query? Who uses it? What for?

Always question your assumptions. Don't be afraid to trust your gut and try crazy things. This is more of a general life advice, so, we might as well end the article here.


