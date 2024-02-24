---
layout: post
title:  "A Neat Trick for Compacting SQL Tables"
date:   2024-01-28 00:00:00 +0000
categories: [programming, sql]
---

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet">
<a href="{{ site.baseurl }}/index.html"><i class='fa fa-home'></i> Home</a>

### Introduction

Back in the days when I was writing a lot of ETL scripts, I learned a nice trick for compacting SQL tables.
As I am not writing that much SQL anymore, the muscle memory is fading away. So, I decided to write it down.
Hopefully, it will be useful for someone besides future me.

Imagine you have a table of items in stock that looks like this:

| date       | item\_name | count |
|:-----------|:-----------|:------|
| 2024-01-01 | apple      | 10    |
| 2024-01-01 | orange     | 4     |
| 2024-01-02 | apple      | 10    |
| 2024-01-02 | orange     | 4     |
| 2024-01-03 | apple      | 10    |
| 2024-01-03 | orange     | 4     |
| 2024-01-04 | apple      | 5     |
| 2024-01-04 | orange     | 4     |
| ...        | ...        | ...   |

There is `date`, `item_name`, and a value that changes over time (`count` in this case).

In a lot of rows, the count remains the same, and only the date changes.
The example table has only a few rows, but imagine that a larger table that has millions of products and thousands of
dates. If you can compact it and record only changes, you can save a ton of rows.

The compacted version can look something like this:

| item\_name | count | date\_from | date\_to |
| :--- | :--- | :--- | :--- |
| apple | 10 | 2024-01-01 | 2024-01-03 |
| orange | 4 | 2024-01-01 | 2024-01-05 |
| apple | 5 | 2024-01-04 | 2024-01-06 |
| orange | 7 | 2024-01-06 | 2024-01-06 |
| apple | 6 | 2024-01-07 | 2024-01-14 |
| orange | 6 | 2024-01-07 | 2024-01-07 |
| orange | 10 | 2024-01-08 | 2024-01-14 |

Apart from being smaller, it is also easier to navigate. At least for me.

### The trick
Let's actually create the table, so you can follow along.
All code below is for SQLite, but it should work in most other SQL databases as well.
There are some online SQLite editors (e.g. [here](https://sqliteonline.com/), 
so you can try it out without installing anything.

Create the original table:
```sql
create table stock
(
    date      date,
    item_name varchar(30),
    count     int
)
;
```

Populate it with some data:
```sql
insert into stock
    (date, item_name, count)
values ('2024-01-01', 'apple', 10),
       ('2024-01-01', 'orange', 4),
       ('2024-01-02', 'apple', 10),
       ('2024-01-02', 'orange', 4),
       ('2024-01-03', 'apple', 10),
       ('2024-01-03', 'orange', 4),
       ('2024-01-04', 'apple', 5),
       ('2024-01-04', 'orange', 4),
       ('2024-01-05', 'apple', 5),
       ('2024-01-05', 'orange', 4),
       ('2024-01-06', 'apple', 5),
       ('2024-01-06', 'orange', 7),
       ('2024-01-07', 'apple', 6),
       ('2024-01-07', 'orange', 6),
       ('2024-01-08', 'apple', 6),
       ('2024-01-08', 'orange', 10),
       ('2024-01-09', 'apple', 6),
       ('2024-01-09', 'orange', 10),
       ('2024-01-10', 'apple', 6),
       ('2024-01-10', 'orange', 10),
       ('2024-01-11', 'apple', 6),
       ('2024-01-11', 'orange', 10),
       ('2024-01-12', 'apple', 6),
       ('2024-01-12', 'orange', 10),
       ('2024-01-13', 'apple', 6),
       ('2024-01-13', 'orange', 10),
       ('2024-01-14', 'apple', 6),
       ('2024-01-14', 'orange', 10)
;
```
And here comes the trick:
```sql
create table stock_compacted as
with temp_change_capture as (select item_name,
                     date,
                     count,
                     row_number() over (partition by item_name order by date) -
                     row_number() over (partition by item_name, count order by date) as change_indicator
              from stock)
select item_name,
       count,
       min(date) as date_from,
       max(date) as date_to
from temp_change_capture
group by item_name, count, change_indicator
;
```
We partition the table by `item_name` and assign row numbers ordered by date.
We also partition by `item_name` and `count` and assign row numbers ordered by date.
The difference between these two row numbers changes when the `count` changes while the `item_name` does not change.
So, if we group by the difference and `item_name` and `count`, we get time periods when the `count` remains the same.

### How to get the original table back?
To get from the compacted table back to the original, you can just join the compacted table with a series of dates.
Generating the date series is probably the only database-specific part of this trick.
This StackOverflow [answer](https://stackoverflow.com/a/32987070) describes how to do it in SQLite.
Below, we are using a slight modification of that approach.

```sql
-- generate a series of dates
with recursive d(date) as (
  select min(date_from) from stock_compacted
  union all
  select date(date, '+1 day')
  from d
  where date <= (select max(date_to) from stock_compacted)
)
select d.date,
       sc.item_name,
       sc.count
from d
inner join stock_compacted sc
on d.date between sc.date_from and sc.date_to
order by d.date, sc.item_name
;
```


### Conclusion
This trick is quite computationally expensive, so if you are not tight on storage, you might want to think twice before using it.
Moreover, some database storages use compression, so the gain might not be as big as you'd expect.

Anyway, even if you might never use it, I hope you appreciate its simple elegance as much as I do.