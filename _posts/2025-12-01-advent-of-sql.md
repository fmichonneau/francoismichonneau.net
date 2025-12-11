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

This year, the Advent of SQL was hosted by the Database School. I don't know anything about them except that they took over the Advent of SQL from last year. The challenges are in the forms of video tutorials with an associated playground. 

For me, it's an opportunity to brush up my DuckDB skills. I'm going to work from within R just in case I need to do any additional data manipulation or visualization, but my goal this year is to do everything using DuckDB SQL (and not use LLMs for help).

The data can be downloaded from the Database School, once you created an account.

## Day 1

It's a single table, containing messy wish list data. The goal is to find the most common wishes ordered in descending order.

```r
## Day 1

# Replace `BIGSERIAL` with INTEGER in wish_list table definition:
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