---
title: "Advent of SQL 2025 with DuckDB and R"
excerpt: "An annotated list of solutions to the Advent of SQL challenges"
layout: "single"
date: "2025-12-11"
type: "post"
published: true
categories: ["Hacking"]
tags: ["r", "duckdb"]
keep-yaml: true
validate-yaml: false
header:
  overlay_image: "/images/2024-12-01-bw-winter.jpg"
  overlay_filter: 0.5
  caption: "Photo by [Cristina Gottardi](https://unsplash.com/@cristina_gottardi?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash) on [Unsplash](https://unsplash.com/photos/pine-tree-covered-with-snow-4L-AyDJM-yM?utm_content=creditCopyText&utm_medium=referral&utm_source=unsplash)"
toc: true
---

## Intro

This year, the Advent of SQL is hosted by the Database School. I don't know anything about them except that they took over the Advent of SQL from last year. There will be only 10 challenges this year (with 25 challenges last year, it felt a little long, so this is a welcome change). The spirit of the challenges seem to remain the same: using SQL to solve Christmas theme puzzles. The delivery format is however different as it uses the Database School platform and format. You need to create an account, and log in to access the challenges and their associated data. Each challenge is in the form of a video tutorial with an associated playground. 

I'm going to use these challenges as an opportunity to brush up my SQL skills, using DuckDB. I'm going to work from R (just in case I need to do any additional data manipulation or visualization), but my goal this year is to do everything using DuckDB SQL (and not use LLMs for help, just searching and reading the docs the old fashion way). I might use LLMs to propose more elegant/alternative solutions once I have a working solution. 

I'll post my solutions daily (or as often as I can manage) below. The data can be downloaded from the Database School website once you created an account. 

## Day 1

It's a single table, containing messy wish list data. The goal is to find the most common wishes ordered in descending order.

```r
# Replace `BIGSERIAL` with `INTEGER` in `wish_list` table definition.

# Create DuckDB database with:
#  duckdb ./data_duckdb/advent_day_01.duckdb < ./data_sql/day1-wish-list.sql
# Be patient, these single inserts take a while to run in duckdb (about 90s)

con <- DBI::dbConnect(duckdb::duckdb(), "data_duckdb/advent_day_01.duckdb")
DBI::dbGetQuery(
  con,
  "
  SELECT wish, count(wish) AS n
    FROM (SELECT lower(trim(raw_wish)) AS wish FROM 'wish_list')
    GROUP BY wish
    ORDER BY n DESC;
"
)
```

## Day 2

With day 2, we have two tables: `snowball_inventory` and `snowball_categories`. The goal is to find the total quantity of items in inventory for each category, ordered by total quantity ascending. Only items with quantity > 0 should be included. You need to watch the video to understand the challenge as some information is not included in the challenge description itself.

```r
# Create DuckDB database with (no need to edit the file):
# duckdb ./data_duckdb/advent_day_02.duckdb < ./data_sql/day2-inserts.sql

con <- DBI::dbConnect(duckdb::duckdb(), "data_duckdb/advent_day_02.duckdb")
DBI::dbGetQuery(
  con,
  "
  SELECT category_name, SUM(quantity) AS total_quantity
  FROM (
       SELECT i.category_name, i.status, i.quantity, o.*
       FROM snowball_inventory i
       JOIN snowball_categories o
      ON (i.category_name = o.official_category AND quantity > 0)
    )
    GROUP BY category_name
    ORDER BY total_quantity ASC;
  "
)
```

## Day 3

Copy and paste from the challenge:

> Using the hotline_messages table, update any record that has "sorry" (case insensitive) in the transcript and doesn't currently have a status assigned to have a status of "approved".
> Then delete any records where the tag is "penguin prank", "time-loop advisory", "possible dragon", or "nonsense alert" or if the caller's name is "Test Caller".
> After updating and deleting the records as described, write a final query that returns how many messages currently have a status of "approved" and how many still need to be reviewed (i.e., status is `NULL`).

```r
# Create DuckDB database with (no need to edit the file):
# duckdb ./data_duckdb/advent_day_03.duckdb < ./data_sql/day3-inserts.sql

con <- DBI::dbConnect(duckdb::duckdb(), "data_duckdb/advent_day_03.duckdb")

DBI::dbExecute(
  con,
  "
  UPDATE hotline_messages
  SET status = 'approved' 
  WHERE LOWER(transcript) LIKE '%sorry%'
    AND status IS NULL;
  "
)

DBI::dbExecute(
  con,
  "
  DELETE FROM hotline_messages
  WHERE tag IN (
    'penguin prank',
    'time-loop advisory',
    'possible dragon',
    'nonsense alert'
    )
    OR caller_name = 'Test Caller';
  "
)

DBI::dbGetQuery(
  con,
  "
  SELECT clean_status, COUNT(clean_status)
  FROM (
    SELECT
    status, 
      CASE 
        WHEN status IS NULL THEN 'TBD'
        ELSE 'approved'
      END as clean_status
    FROM hotline_messages
    )
  GROUP BY clean_status;
"
)
```

## Day 4 

Copy and paste from the challenge:

> Using the official_shifts and last_minute_signups tables, create a combined de-duplicated volunteer list.
> Ensure the list has standardized role labels of Stage Setup, Cocoa Station, Parking Support, Choir Assistant, Snow Shoveling, Handwarmer Handout.
> Make sure that the timeslot formats follow John's official shifts format.

I used the snake case format for the role, it looks like the challenge actually asked for title case. I left the 'ELSE TBD' clauses in there as I used them when building the queries to make sure I caught all the cases. I had also checked for unique values in the time slots and given there were just a few I went for a CASE WHEN approach rather than something more sophisticated.

```r
# Create DuckDB database with (no need to edit the file):
# duckdb ./data_duckdb/advent_day_04.duckdb < ./data_sql/day4-inserts.sql

con <- DBI::dbConnect(duckdb::duckdb(), "data_duckdb/advent_day_04.duckdb")

DBI::dbGetQuery(
  con,
  "SELECT * FROM official_shifts"
)

DBI::dbGetQuery(
  con,
  "SELECT
  volunteer_name,
  CASE
    WHEN assigned_task ILIKE '%choir%' THEN 'choir_assistant'
    WHEN assigned_task ILIKE '%stage%' THEN 'stage_setup'
    WHEN assigned_task ILIKE '%cocoa%' THEN 'cocoa_station'
    WHEN assigned_task ILIKE '%parking%' THEN 'parking_support'
    WHEN assigned_task ILIKE '%shovel%' THEN 'snow_shoveling'
    WHEN assigned_task ILIKE '%hand%' THEN 'handwarmer_handout'
    ELSE 'TBD'
  END as role,
  CASE
    WHEN (time_slot='10AM' OR time_slot ='10 am') THEN '10:00 AM'
    WHEN (time_slot='2 PM' OR time_slot='2 pm') THEN '2:00 PM'
    WHEN time_slot = 'noon' THEN '12:00 PM'
    ELSE 'TBD'
  END as shift_time
  FROM last_minute_signups
 
  UNION

  SELECT volunteer_name,
         role,
         shift_time
  FROM official_shifts
  ORDER BY volunteer_name;
"
)
```