---
layout: post
title:  "A Neat Trick for Compacting SQL Tables"
date:   2024-01-28 00:00:00 +0000
categories: [programming, sql]
---

<link href="https://maxcdn.bootstrapcdn.com/font-awesome/4.2.0/css/font-awesome.min.css" rel="stylesheet">
<a href="{{ site.baseurl }}/index.html"><i class='fa fa-home'></i> Home</a>

### Introduction

Back when I was writing a lot of ETL scripts, I learned a nice trick for making SQL tables smaller. 
As I am not writing that much SQL anymore, the muscle memory is fading away. 
So, I decided to write it down. I hope that someone besides future me will find it useful.

Suppose you have a table of items in stock that looks like this:

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

In many rows, the `count` remains the same, and only the `date` changes. 
This example has only a few rows. 
Imagine a more realistic scenario–a table with millions or billions of rows. 
If you compact it, and keep only changes, you'll save a ton of space.

The compacted version can look something like this:

| item\_name | count | date\_from | date\_to   |
|:-----------|:------|:-----------|:-----------|
| apple      | 10    | 2024-01-01 | 2024-01-03 |
| orange     | 4     | 2024-01-01 | 2024-01-05 |
| apple      | 5     | 2024-01-04 | 2024-01-06 |
| orange     | 7     | 2024-01-06 | 2024-01-06 |
| apple      | 6     | 2024-01-07 | 2024-01-14 |
| orange     | 6     | 2024-01-07 | 2024-01-07 |
| orange     | 10    | 2024-01-08 | 2024-01-14 |

Apart from being smaller, it is also easier to navigate. At least for me.

### The trick
Let's first create the table, so you can follow along.
All code below is for SQLite, but most of it should work in any SQL database.
There are some online SQLite editors, e.g. [here](https://sqliteonline.com/), 
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

Populate it:
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
You partition the table by `item_name` and assign row numbers ordered by date. 
You also partition it by `item_name` and `count` and assign row numbers ordered by date. 
Then, you compute the difference between these row numbers (`change_indicator`). 
The `change_indicator` changes when the `count` changes and the `item_name` remains unchanged. .

### How to get the original table back?
If you join the compacted table with a series of dates, you get the original table back.

Generating the date series is the only database-specific part of this trick.
This StackOverflow [answer](https://stackoverflow.com/a/32987070) describes how to do it in SQLite.
Below example uses a slight modification of that approach.

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
-- join it with the compacted table
inner join stock_compacted sc
on d.date between sc.date_from and sc.date_to
order by d.date, sc.item_name
;
```


### Conclusion
This trick is quite computationally expensive. If you are not tight on storage, you might want to think twice before using it.
Moreover, some database storages use compression, so the gain might not be as big as you'd expect.

Anyway, even if you might never use it, I hope you appreciate its simple elegance as much as I do.