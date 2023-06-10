---
layout: post
title:  "Journeyman's Notes on Optimizing SQL Queries"
date:   2023-05-10 21:00:00 +0000
categories: [programming, sql]
---
### Introduction

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
We started the article hinting that improving query performance can boost your applications significantly. But not always.
Is it a problem that a query takes ten minutes? What if it runs once a day, does not block any other queries, and processes terabytes of data? Maybe not a problem.
What if it needs huge Snowflake cluster, and the ten minutes cost you a ton of money. Well, then it might be a problem.
But a query that takes 200 millisecond is surely not a problem, right? What if your application needs to run it a hundred times per second. Well...
Before you start optimizing, know the metric you need to improve. It can be cluster cost, database load, a response time of an endpoint, duration of some job, and many other things.
Only then, you can actually decide what to optimize.

Sometimes, the cost of your time might outweigh and potential cost savings. Other times, it might be more advantageous to get a larger database.
On some occasions, your best option is to start tweaking SQL queries. And that is the fun part.

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
Most techniques that we will cover will be useful for both OLTP and OLAP. In some cases, we will see that they work only on one type of database.
This is usually because OLAP database try to be smarter—since they don't intend respond within milliseconds, they can take more time to optimize query execution.
The positive of this is that a poorly written query can perform well on OLAP. The negative is that fine-tuning the query might not always work because the query 
optimizer might translate it to the same plan as the suboptimal one. After all, SQL is a declarative language (TODO: expand).

TODO: hint to possible future articles about differences in OLAP and OLTP principles

- Apache Druid
- Clickhouse


### Understand your data
Databases make assumptions about your data. Tons of smart people have spent years optimizing and testing databases
to make sure that these assumptions are accurate. But you can still do better than them.
While a database might guess that a join will produce something between 1 and 30 million rows, you might know that it will
be exactly one million. Or you might know that a filter removes 99 % rows in a table, so you might want to push it to a subquery.
Or you might know that a table includes duplicate join keys, so, you deduplicate it before joining.

### Clean up
A few years ago, I was put in charge of an application. Right from the outset I heard complaints that it is slower than it used to be
and customers are upset. After some profiling, I realized that fetching data was the bottleneck. The application was supposed to 
compute several metrics for products and what stroke me as odd was that there were a lot more products in the database than the company
actually sold. Indeed, most products in the database were inactive. When I deleted them, the critical endpoint got 3 times faster.
Deleting what you don't need is one of the easiest ways to speed up your queries. If you do not have automatic cleanup processes
in place, data tends to pile up. Since the queries need to run through more data, they get slower.
Delete obsolete data and setup processed that do it regularly for you.
There will be data that you can't delete–for legal reasons or to not upset your users. Sometimes, you might get around this
by creating snapshot tables or materialized views (see Use subtables in OLAP (+ Materialized views)). If that is not possible,
keep in mind that your application's performance will gradually degrade. That does not always have to be a problem. If the accumulation
is not too fast, the performance penalty may be something that you don't need to worry about for the lifespan of most applications.

### Use diagnostic tools
Databases usually have a way to show you how they decided to execute your query, i.e. display the query plan. Studying it can show you
what the bottlenecks are. When I am optimizing a query, it is usually a back and forth between checking the plan and tweaking the query.
I think execution plan visualizers are of great help here. While I might struggle to find the source of issues in the textual plan, it is 
often clear from the first look at the picture.
Some database providers embed visualizers to their service, but there are also free versions, e.g. for Postgres.

Use visualizers if available.
e.g. EXPLAIN in postgres

### WHERE to start
Start by fiddling with the `WHERE` clause. The biggest improvements in query performance I have ever seen were resulted from, often small, changes to filtering clause.
Let's look at an example query.
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
easier job if they work with fewer data.

NB: where clause that duplicates condition already included in an inner join might actually slow the query down.


### Aggregation vs. sorting
Sorting is expensive. If you can, you should probably avoid it.
I've seen quite a lot of code that sorts table only to get a maximum of some column, i.e. where simple group by would work
fine. Most often, such queries started out as more complex, and the sorting was originally necessary to get values
from some other column on the same line as our sorting column. Then, someone simplified but did not realize that
instead of sorting they can use aggregation.
Let's see how this can happen on an example: TODO

### Database design

A well-designed can save you a ton of troubles. While there are theoretical approaches that will lead you to the most
logically consistent structure, you should consider think about the performance you need to.
What data gets read and written the most, do you prefer to hold historical data or not having to sort to get the most recent snapshot.
What datatypes you need etc.
This is a much broader topic, and I hope to cover it in some of the following articles.

### Keep your queries simple
If you can, keep your queries simple. You will make it easier for query planners to find and efficient query plan.
Snowflake actually recommends exactly this—use simple queries and let the query planner do its job.
Of course, this is often easier said then done. You might actually need a more complex transformation, and it is not possible
to get the required data in a simple query. In such cases, Snowflake recommends using sub-tables (TODO: add source). These do come with
a danger of sacrificing data integrity, however. Moreover, in OLTP databases, you often can't afford to copy tons of data back and forth,
not to mention work with outdated data. Let's look at the alternatives, you have, if you want to keep you queries simple, but you need to
transform data in a non-trivial way.

### Subqueries
Subqueries and common table expressions (CTEs) are often necessary unless you decide to create a temporary table or  view.
Occasionally, they are useful even if you could theoretically avoid them. For example, instead of performing complicated cross join
and filtering its results, you may be better off de-duplicating tables first, and then joining them without having to filter the result.
The caveats are: query planner might be smart enough to optimize the original (non-nested) query, and using subqueries can actually
confuse it. Hence, this is something you have to test—there is not theoretical rule that will reliably tell whether using subquery is
a good idea in particular case. What usually works for me is checking the size of the table in subquery before and after applying
the subquery filter. If it is large and becomes significantly smaller after filtering, putting the filter into subquery might work.
Otherwise, it usually won't.
Let's explore when subqueries help with some examples.
TODO: add examples where subqueries work and where they harm

### Use subtables in OLAP (+ Materialized views)
I've seen countless examples of tables named like order_**advanced**, order_**enriched**, order_**v2** order_**extra**.
These boastful suffixes usually indicate that it is just the original order table with a few columns joined from another table.
I've seen hardcore programmers cry in pain when they heard of such duplication and violation of data integrity.
The truth is that tables like these are common in analytical workflows, and, I daresay, even useful.

Subtable is not an official term. TODO: explain  / maybe use derived table instead

Imagine you have a team of ten analysts, each of them analyzing some aspect of sales. They all need data about orders and transportation costs,
which is exactly what the order_advanced table gives them. Of course, they could join the transportation costs themselves
but that might make they query too slow, and they would need to duplicate the joining code creating a potential for mistakes.
If they don't expect near real-time data, the inconsistency is not such a big problem. I.e. they cannot expect to join order data
to, for example, website traffic data for today, and expect a reliable result. If the order_advanced is refreshed once a day, they should just discard today's data.

Another case where you might reach for a subtable is an OLAP job. Instead of crafting complicated nested queries, you just create
temporary table simplifying query planner's job, and delete them when transformation finished.

A better alternative to non-temporary subtable is a materialized view, which is basically a table that knows how to
refresh its data. Materialized view is read-only—you can write only to underlying tables. On refresh, it runs the queries
by which it was created.
However, making sure that the refresh happens when it should is a discipline of its own. (I might write another article on that).
The difference between view and materialized view is that materialized view runs queries on refresh, and holds the data,
i.e. it is a 'derived table', whereas normal view runs its queries when you query the view.

I would argue that using a materialized view is better than using persistent subtables. Materialized view clearly indicates
that is read only and can be refreshed with one command.

On the other hand, you might have good reasons to use subtables, e.g.,  for compatibility with exporters to other systems
or to make sure that your code lives somewhere else and not in the database.

Some examples where materialized view can help: TODO

The same thing with subtable TODO:


### Indices, clustering keys, partitioning keys

You definitely want to hit an index. Indices are the most common tool databases give you to speed up fetching or filtering records.
When your query hits an index, database does not need to scan the whole table, and this needs to read much fewer data.
Indices work primarily in OLTP databases.

Be careful when you read about indices in OLAP. Snowflake, for example, lets you define a primary key. You might think
that this creates a unique index as a well-behaved database would do. Instead, it only adds a non-null constraint on the columns.
Snowflake says in their documentation that they use indices mainly as a documentation feature, but I am sure I was not the
only person who was confused by duplicates in a primary key column.

In OLAP databases, you might encounter clustering, partitioning, or some other keys. These are columns that usually divide table into
chunks. If we take our `post` table from previous examples and set clustering key to `updated_at`, the database
will ideally create chunks that cover mutually exclusive time intervals of `updated_at` (TODO: check). 
Hitting a clustering or partitioning key amounts to similar jackpot as hitting an index—the database needs to process
only the partition with the key. If you do not hit the key, it needs to check all partitions.

Clustering or partitioning is primarily useful on larger tables (think > 1TB); with smaller ones, the overhead of shuffling data
might not be worth it. If you are paying for the time that database does something, as in Snowflake, you should also consider
that partitioning large table takes a lot of time and needs to happen regularly, otherwise the partitions won't be balanced,
which will worsen query performance.

TODO: examples of both indices and partitioning

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

That is what I did, hoping to see some performance improvement. Now, the sub-query looked like this:
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
So, I ended up with seemingly nonsensical query:
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
But you should be careful when optimizing, because query planner can sometimes get hints from things that almost seem like a programmer's mistake.


### Conclusion: Question and experiment
Linear thinking won't always get you there. Most of the time, optimizing query performance is a rigorous process—you rearrange clauses, trim off unnecessary data,
choose the most efficient functions, measure changes, and repeat.
But sometimes, queries will be slow, no matter how much love and care you give them.

Then, you have to think more broadly. Can you get the data from somewhere else, split the query into parts, or redefine the database structure?
Do you actually need to return all data in this query? Who uses it? What for?

Always question your assumptions. Don't be afraid to trust your gut and try crazy things. This is more of a general life advice, so, we might as well end the article here.
