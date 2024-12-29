---
title: "Advent of SQL with DuckDB and R"
excerpt: "An annoted list of solutions to the Advent of SQL challenges"
layout: "single"
date: "2024-12-01"
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


## Quick Overview

Advent of Code is a popular advent calendar of programming puzzles. I
have attempted to do it in the past with R but I always gave up half-way
through the month because it was taking too much of my time. Last year,
I had fun going through the challenges of the [Hanukkah of
Data](https://hanukkah.bluebird.sh/). This year, Bob Rudis, via his
excellent Daily Drop Newsletter, pointed to Advent of SQL. I am
attempting to solve the challenges using DuckDB and/or dplyr. I
appreciate that the challenges so far can be solved relatively quickly.

My answers to the challenges and some annotations are in this post. I’ll
update the post as the challenges get published and I find time to solve
them.

## Data import

The Advent of SQL provides data for each challenge as a SQL file. They
use Postgres for this challenge and while the compatibility between
Postgres and DuckDB is pretty high, some features are not available in
DuckDB, and the data files need to be modified to be able to import data
with DuckDB.

To create a DuckDB database from a SQL file:

    duckdb <database_file_name.duckdb> < <sql_file.sql>

Once the database file is created, you can work with it from R.

## Day 1

Create the DuckDB database:

    duckdb advent_day_01.duckdb < advent_of_sql_day_1.sql

``` r
library(duckdb)
library(dplyr)

## Connect to the Database
con_day01 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_01.duckdb")

## Check the content
dbListTables(con_day01)
dbGetQuery(con_day01, "SELECT wishes FROM wish_lists limit 10;")

## Install and load the json extension to work with the JSON data
dbExecute(con_day01, "INSTALL json; LOAD json;")

## Create tidy version of the data
dbExecute(
  con_day01,
  "
  CREATE OR REPLACE VIEW tidy_wishlist AS
  SELECT
    list_id,
    child_id,
    trim(wishes.first_choice::VARCHAR, '\"') AS primary_wish,
    trim(wishes.second_choice::VARCHAR, '\"') as backup_wish,
    trim(wishes.colors[0]::VARCHAR, '\"') AS favorite_color,
    json_array_length(wishes.colors) AS color_count
  FROM wish_lists;
  "
)

## Inspect newly created VIEW
tbl(con_day01, "tidy_wishlist")

## Build answer
left_join(
  tbl(con_day01, "children") |>
    select(child_id, name),
  tbl(con_day01, "tidy_wishlist"),
  by = "child_id"
) |>
  left_join(
    tbl(con_day01, "toy_catalogue"),
    by = c("primary_wish" = "toy_name")
  ) |>
  mutate(
    gift_complexity = case_when(
      difficulty_to_make == 1 ~ "Simple Gift",
      difficulty_to_make == 2 ~ "Moderate Gift",
      difficulty_to_make >= 3 ~ "Complex Gift",
      TRUE ~ NA_character_
    ),
    workshop_assignment = case_when(
      category == "outdoor" ~ "Outside workshop",
      category == "educational" ~ "Learning workshop",
      TRUE ~ "General workshop"
    )
  ) |>
  select(name, primary_wish, backup_wish, favorite_color, color_count, gift_complexity, workshop_assignment) |>
  arrange(name) |>
  collect(n = 5) |>
  rowwise() |>
  mutate(
    answer = glue::glue_collapse(c(name, primary_wish, backup_wish, favorite_color, color_count, gift_complexity, workshop_assignment), sep = ",")
  ) |>
  pull(answer)

dbDisconnect(con_day01, shutdown = TRUE)
```

The JSON extension in DuckDB allowed me to extract the required data to
create a tidy version to solve the problem.

## Day 2

### Data Preparation

The SQL dump for this challenge used the `SERIAL` data type which is
[not supported](https://github.com/duckdb/duckdb/issues/1768) by DuckDB.
`SERIAL` is a convenience to create unique, auto-incrementing, ids. The
way around it in DuckDB, is to create a sequence and using in the table
definition. I edited the SQL dump so the beginning of the file now looks
like:

``` sql
DROP TABLE IF EXISTS letters_a CASCADE;
DROP TABLE IF EXISTS letters_b CASCADE;

CREATE SEQUENCE laid START 1;
CREATE TABLE letters_a (
  id INTEGER PRIMARY KEY DEFAULT nextval('laid'),
  value INTEGER
);

CREATE SEQUENCE lbid START 1;
CREATE TABLE letters_b (
  id INTEGER PRIMARY KEY DEFAULT nextval('lbid'),
  value INTEGER
);
```

``` r
con_day02 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_02.duckdb")

## Create a single table combining `letters_a` and `letters_b` with `UNION`
## Use function `chr()` to convert ASCII codes into letters
dbExecute(
  con_day02,
  "CREATE OR REPLACE VIEW letters_decoded AS
   SELECT  *, chr(value) AS character FROM letters_a
   UNION
   SELECT  *, chr(value) AS character FROM letters_b
  "
)

## Define list of valid characters
valid_characters <- "[A-Za-z !\"\'(),-.:;?]"

## Filter data to only keep valid_characters
## and collapse results to extract message
tbl(con_day02, "letters_decoded") |>
  filter(grepl(valid_characters, character)) |>
  arrange(id) |>
  pull(character) |>
  paste(x =_, collapse = "")

dbDisconnect(con_day02, shutdown = TRUE)
```

## Day 3

Again, the SQL dump used `SERIAL`. Additionally, DuckDB does not support
the `XML` data type, so I switched to use `VARCHAR` and used R to work
with the XML data. I edited the beginning of the dump file to look like:

``` sql
DROP TABLE IF EXISTS christmas_menus CASCADE;

CREATE SEQUENCE cmid START 1;
CREATE TABLE christmas_menus (
  id INTEGER PRIMARY KEY DEFAULT nextval('cmid'),
  menu_data VARCHAR
);
```

I really didn’t use DuckDB’s engine for this challenge. I only relied on
R to work with the XML data:

``` r
con_day03 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_03.duckdb")

menus <- tbl(con_day03, "christmas_menus") %>%
  collect()


## Figure out how many XML schemas are being used in the data
get_menu_version <- function(menu_data) {
  xml2::read_xml(menu_data) |>
    xml2::xml_find_all(".//@version") |>
    xml2::xml_text()
}

menu_versions <- purrr::map_chr(menus$menu_data, get_menu_version)

unique(menu_versions)

## only 3 different versions

## Extract the number of guests based on the XML schema
get_guest_number <- function(menu_data, xml_version) {
  element <- switch(
    xml_version,
    "3.0" = ".//headcount/total_present",
    "2.0" = ".//total_guests",
    "1.0" = ".//total_count"
  )

  xml2::read_xml(menu_data) |>
    xml2::xml_find_all(element) |>
    xml2::xml_text() |>
    as.numeric()
}

n_guests <- purrr::map2_dbl(menus$menu_data, menu_versions, get_guest_number)

## Extract the food ids (only for events with the right number of guests)
food_ids <- purrr::map2(menus$menu_data, n_guests, function(.x, .g) {
  if (.g < 78) return(NULL)
  xml2::read_xml(.x) |>
    xml2::xml_find_all(".//food_item_id") |>
    xml2::xml_text()
})

## And count them to find the most common one
unlist(food_ids) |>
  as_tibble() |>
  count(value, sort = TRUE)

dbDisconnect(con_day03, shutdown = TRUE)
```

## Day 4

Again, the original dump used `SERIAL`, for this challenge, I simply
replaced it with `INTEGER`, so the beginning of the file looks like:

``` sql
DROP TABLE IF EXISTS toy_production CASCADE;

CREATE TABLE toy_production (
  toy_id INTEGER PRIMARY KEY,
  toy_name VARCHAR(100),
  previous_tags TEXT[],
  new_tags TEXT[]
  );
```

``` r
con <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_04.duckdb")

dbGetQuery(
  con,
  "SELECT
     toy_id,
     list_where(new_tags, list_transform(new_tags, x -> NOT list_contains(previous_tags, x))) AS added_tags,
     list_intersect(previous_tags, new_tags) AS unchanged_tags,
     list_where(previous_tags, list_transform(previous_tags, x -> NOT list_contains(new_tags, x))) AS removed_tags,
     len(added_tags) AS added_tags_length,
     len(unchanged_tags) AS unchanged_tags_length,
     len(removed_tags) AS removed_tags_length
   FROM toy_production;"
) |>
  slice_max(added_tags_length) |>
  select(toy_id, ends_with("length"))

dbDisconnect(con_day04, shutdown = TRUE)
```

This challenge required to dive into DuckDB’s functions to work with
lists. While there is a `list_intersect()` function, there does not seem
to be a `list_setdiff()` so instead I combined `list_where()` with
`list_transform()` and `list_contains()` to get there. I would be happy
to hear alternative approaches!

## Day 5

The data could be imported directly from the SQL dump.

``` r
con_day05 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_05.duckdb")

tbl(con_day05, "toy_production") |>
  mutate(previous_day_production = lead(toys_produced)) |>
  mutate(production_change = toys_produced - previous_day_production) |>
  mutate(production_change_percentage = production_change / toys_produced * 100) |>
  slice_max(production_change_percentage)

dbDisconnect(con_day05, shutdown = TRUE)
```

Solving this challenge required using the `lead()` function from the
tidyverse to calculate the change in production and its percentage. I
then used `slice_max()` to extract the row with the largest percentage
change.

## Day 6

The original dump used `SERIAL`, so I updated the beginning of the file
to look like:

``` sql
DROP TABLE IF EXISTS children CASCADE;
DROP TABLE IF EXISTS gifts CASCADE;

CREATE TABLE children (
    child_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    age INTEGER,
    city VARCHAR(100)
);

CREATE TABLE gifts (
    gift_id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    price DECIMAL(10,2),
    child_id INTEGER REFERENCES children(child_id)
);
```

``` r
con_day06 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_06.duckdb")

## First calculate average  price
avg_price <- tbl(con_day06, "gifts") |>
  summarize(mean_price = mean(price, na.rm = TRUE)) |>
  pull(mean_price)

## Join the gifts and children table and filter out results based on average
## price. Finally arrange by price.
tbl(con, "children") |>
  left_join(tbl(con, "gifts"), by = join_by(child_id)) |>
  filter(price >= avg_price) |>
  arrange(price)

dbDisconnect(con_day06, shutdown = TRUE)
```

## Day 7

The original dump used `SERIAL` again, but provided all the `elf_id` so I
updated the beginning of the file to use `INTEGER` instead and it looks like:

```
DROP TABLE IF EXISTS workshop_elves CASCADE;
CREATE TABLE workshop_elves (
    elf_id INTEGER PRIMARY KEY,
    elf_name VARCHAR(100) NOT NULL,
    primary_skill VARCHAR(50) NOT NULL,
    years_experience INTEGER NOT NULL
);
```

```r
con_day07 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_07.duckdb")

tbl(con_day07, "workshop_elves") |>
  filter(
    years_experience == min(years_experience) | years_experience == max(years_experience),
    .by = primary_skill
  ) |>
  filter(
    elf_id == min(elf_id),
    .by = c(primary_skill, years_experience)
  ) |>
  arrange(
    primary_skill,
    desc(years_experience)
  ) |>
  collect() |> 
  summarize(
    result = paste(elf_id, collapse = ","),
    .by = primary_skill
  ) |>
  summarize(
    result = paste(result, primary_skill, sep = ","),
    .by = primary_skill
  )

dbDisconnect(con_day07, shutdown = TRUE)
```

## Day 8

Again, the original data dump used `SERIAL` which I substituted for `INTEGER` to be able to import the data into DuckDB, so the beginning of the file looks like:


```sql
DROP TABLE IF EXISTS staff CASCADE;
CREATE TABLE staff (
    staff_id INTEGER PRIMARY KEY,
    staff_name VARCHAR(100) NOT NULL,
    manager_id INTEGER
);
```

I was in the rush that day, and the solution I came up with is quite hacky and
slow. All the computation takes place in R using a recursive function. This
challenge is a good opportunity to learn recursive CTEs but I'll need to come
back to it.

```r
con_day08 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_08.duckdb")

## make sure there is a single NA
tbl(con_day08, "staff") |>
  pull(manager_id) |>
  is.na() |>
  sum()

find_boss <- function(.data, idx) {
  if (is.na(idx)) {
    return(1)
  }
  
  res <- .data |>
    filter(staff_id == idx[1]) |>
    pull(manager_id)

  c(find_boss(.data, res), idx)
}

staff <- tbl(con_day08, "staff") |>
  collect()

staff |>
  rowwise() |> 
  mutate(path = list(find_boss(staff, .data$manager_id))) |>
  mutate(level = length(path)) |>
  ungroup() |>
  slice_max(level)

dbDisconnect(con_day08, shutdown = TRUE)
```

## Day 9

To replace `SERIAL`, I used `SEQUENCE` for both the `reindeer_id` and the
`session_id` so the beginning of the dump file looks like:

```sql
DROP TABLE IF EXISTS training_sessions CASCADE;
DROP TABLE IF EXISTS reindeers CASCADE;

CREATE SEQUENCE r_id START 1;
CREATE TABLE reindeers (
    reindeer_id INTEGER PRIMARY KEY DEFAULT nextval('r_id'),
    reindeer_name VARCHAR(50) NOT NULL,
    years_of_service INTEGER NOT NULL,
    speciality VARCHAR(100)
);

CREATE SEQUENCE s_id START 1;
CREATE TABLE training_sessions (
    session_id INTEGER PRIMARY KEY DEFAULT nextval('s_id'),
    reindeer_id INTEGER,
    exercise_name VARCHAR(100) NOT NULL,
    speed_record DECIMAL(5,2) NOT NULL,
    session_date DATE NOT NULL,
    weather_conditions VARCHAR(50),
    FOREIGN KEY (reindeer_id) REFERENCES reindeers(reindeer_id)
);
```

```r
con_day09 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_09.duckdb")

tbl(con, "training_sessions") |>
  left_join(
    tbl(con, "reindeers"),
    by = join_by(reindeer_id)
  ) |>
  filter(reindeer_name != "Rudolf") |>
  summarize(
    avg_speed = mean(speed_record, na.rm = TRUE),
    .by = c(reindeer_name, exercise_name)
  ) |>
  slice_max(avg_speed, by = reindeer_name) |>
  slice_max(avg_speed, n = 3) |>
  collect() |>
  glue::glue_data("{reindeer_name},{round(avg_speed, 2)}")

dbDisconnect(con_day09, shutdown = TRUE)
```

## Day 10

I again replaced `SERIAL` with using a `SEQUENCE` in the data dump. The
beginning of the file looks like:


```sql
DROP TABLE IF EXISTS Drinks CASCADE;

CREATE SEQUENCE d_id START 1;
CREATE TABLE Drinks (
    drink_id INTEGER PRIMARY KEY DEFAULT nextval('d_id'),
    drink_name VARCHAR(50) NOT NULL,
    date DATE NOT NULL,
    quantity INTEGER NOT NULL
    );
```

```r
con_day10 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_10.duckdb")

tbl(con_day10, "Drinks") |>
  summarize(
    quantity = sum(quantity, na.rm = TRUE), .by = c(date, drink_name)
  ) |>
  tidyr::pivot_wider(names_from = drink_name, values_from = quantity) |>
  filter(
    `Hot Cocoa` == 38,
    `Peppermint Schnapps` == 298,
    `Eggnog` == 198
  )

dbDisconnect(con_day10, shutdown = TRUE)
```

The magic here is that it all happens within DuckDB even with the call to
`pivot_wider()`.


## Day 11

The data could be imported directly from DuckDB.

I first wrote the solution to this challenge using the `{slider}` package to get
the moving average. But the data has to be pulled in R's memory to make this
work. I then tried to solve it using just DuckDB to practice window functions,
but the query returns a second result. I have not investigated why this is the
case yet.

```r
con_day11 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_11.duckdb")

## R solution
tbl(con_day11, "TreeHarvests") |>
  collect() |>
  mutate(
    avg_yield = slider::slide_dbl(trees_harvested, mean, .before = 2, .complete = FALSE),
    .by = c(field_name, harvest_year)
  ) |>
  slice_max(avg_yield)

## DuckDB SQL solution
dbGetQuery(con_day11,
  "
  WITH results AS (
  SELECT field_name, harvest_year, season,
     CASE WHEN season = 'Spring' THEN 1 WHEN season = 'Summer' THEN 2
          WHEN season = 'Fall' THEN 3 WHEN season = 'Winter' THEN 4 END AS season_order,
     trees_harvested,
     avg(trees_harvested) OVER
       (PARTITION BY 'field_name', 'harvest_year' ORDER BY 'season_order'
        ROWS 2 PRECEDING) AS avg_yield
  FROM TreeHarvests
  )
  SELECT * FROM results WHERE avg_yield = (SELECT max(avg_yield) FROM results)
  "
)

dbDisconnect(con_day11, shutdown = TRUE)
```

## Day 12

The data dump needed again to use `SEQUENCE` to replace `SERIAL`, so I edited
the beginning of the file to look like:

```sql
CREATE OR REPLACE SEQUENCE g_id START 1;
CREATE OR REPLACE SEQUENCE r_id START 1;

CREATE OR REPLACE TABLE gifts (
    gift_id INTEGER PRIMARY KEY DEFAULT nextval('g_id'),
    gift_name VARCHAR(100) NOT NULL,
    price DECIMAL(10,2)
);

CREATE OR REPLACE TABLE gift_requests (
    request_id INTEGER PRIMARY KEY DEFAULT nextval('r_id'),
    gift_id INT,
    request_date DATE,
    FOREIGN KEY (gift_id) REFERENCES gifts(gift_id)
);
```

This challenge could be solve using only {dplyr} functions. There were 10 gifts
that tied for first place, so I relied on the `n` argument from the `print`
function to display more data to be able to see the second most popular item.

```r
con_day12 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_12.duckdb")

tbl(con_day12, "gift_requests") |>
  count(gift_id) |>
  mutate(overall_rank = percent_rank(n)) |>
  left_join(tbl(con_day12, "gifts"), by = join_by(gift_id)) |>
  arrange(desc(overall_rank), gift_name) |>
  print(n = 20)

dbDisconnect(con_day12, shutdown = TRUE)
```

## Day 13

To be able to import the data into DuckDB, I once again edited the beginning of
the dump file to use `SEQUENCE` to replace `SERIAL`:

```sql
CREATE OR REPLACE SEQUENCE cl_id START 1;

DROP TABLE IF EXISTS contact_list;
CREATE TABLE contact_list (
    id INTEGER PRIMARY KEY DEFAULT nextval('cl_id'),
    name VARCHAR(100) NOT NULL,
    email_addresses TEXT[] NOT NULL
);
```

```r
con_day13 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_13.duckdb")

dbGetQuery(
  con_day13,
  "
  WITH all_domains AS
  (SELECT
   id, name, unnest(email_addresses) AS addresses,
   regexp_extract(addresses, '@(.+)$', 1) AS domains
  FROM contact_list
  )
  SELECT domains, COUNT(domains) AS n_users FROM all_domains
  GROUP BY domains
  ORDER BY n_users DESC
  "
)

dbDisconnect(con_day13, shutdown = TRUE)
```

## Day 14

I replaced `SERIAL` with `SEQUENCE` once again:

```sql
DROP TABLE IF EXISTS SantaRecords CASCADE;

CREATE OR REPLACE SEQUENCE rid START 1;

CREATE TABLE SantaRecords (
    record_id INTEGER PRIMARY KEY DEFAULT nextval('rid'),
    record_date DATE NOT NULL,
    cleaning_receipts JSON NOT NULL
);
```

```r
con_day14 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_14.duckdb")

dbExecute(con, "LOAD json;")

dbGetQuery(
  con_day14,
  "WITH extracted AS (
   SELECT
     record_date,
     cleaning_receipts->>'$..garment' AS garment,
     cleaning_receipts->>'$..color' AS color,
     cleaning_receipts->>'$..drop_off' AS drop_off,
     cleaning_receipts->>'$..receipt_id' AS receipt_id
   FROM SantaRecords
  ),
  tidy AS (
    SELECT
      record_date,
      unnest(garment) AS tidy_garment,
      unnest(color) AS tidy_color,
      unnest(drop_off) AS dropoff,
      unnest(receipt_id) AS tidy_receipt_id
    FROM extracted
  )
  SELECT * FROM tidy
  WHERE tidy_garment = 'suit' AND tidy_color = 'green'
  ORDER BY dropoff DESC;"
)

dbDisconnect(con_day14, shutdown = TRUE)
```

## Day 15

This was the first challenge of the series that dealt with spatial data. The
data required a little more preparation. I updated the dump file to:

- use `SEQUENCE` instead of `SERIAL`
- replace `GEOGRAPHY(POINT)` and `GEOGRAPHY(POLYGON)` with `GEOMETRY`
- for each spatial feature, I removed `ST_setSRID(..., 4326)` given that it's
  the default in DuckDB.
  
The beginning of the file looked like:

```sql
CREATE OR REPLACE SEQUENCE sid START 1;
CREATE TABLE sleigh_locations (
id INTEGER PRIMARY KEY DEFAULT nextval('sid'),
timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
coordinate GEOMETRY NOT NULL
);


CREATE OR REPLACE SEQUENCE aid START 1;
CREATE TABLE areas (
    id INTEGER PRIMARY KEY DEFAULT nextval('aid'),
    place_name VARCHAR(255) NOT NULL,
    polygon GEOMETRY NOT NULL
);
```

and the sleigh location table data looked like:

```sql
INSERT INTO sleigh_locations (timestamp, coordinate) VALUES
('2024-12-24 22:00:00+00', ST_Point(37.717634, 55.805825));
```

I edited the `areas` table the same way, for instance the first area looked
like:

```sql
('New_York', ST_GeomFromText('POLYGON((-74.25909 40.477399, -73.700272 40.477399, -73.700272 40.917577, -74.25909 40.917577, -74.25909 40.477399))')),
```

```r
con_day15 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_15.duckdb")

dbExecute(
  con_day15, "INSTALL spatial; LOAD spatial;"
)

dbGetQuery(
  con_day15,
  "
 SELECT areas.place_name
 FROM areas
 JOIN sleigh_locations on ST_Within(sleigh_locations.coordinate, areas.polygon)
  "
)
```

## Day 16

Day 16 required the same data preparation as for day 15.

```r
con_day16 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_16.duckdb")

dbExecute(
  con_day16, "INSTALL spatial; LOAD spatial;"
)

dbGetQuery(
  con_day16,
  "
  SELECT sleigh_locations.timestamp, areas.place_name
  FROM sleigh_locations
  JOIN areas on ST_Within(sleigh_locations.coordinate, areas.polygon)
  "
) |>
  summarize(time_spent = max(timestamp) - min(timestamp), .by = "place_name") |>
  slice_max(time_spent)

dbDisconnect(con_day16, shutdown = TRUE)
```

## Day 17

```r
con_day17 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_17.duckdb")
dbExecute(con_day17, "INSTALL icu; LOAD icu;")
dbGetQuery(
  con_day17,
  "SELECT
     *,
     ('2024-12-24' || ' ' || business_start_time || ' ' || timezone)::TIMESTAMPTZ AS start_time_utc,
     ('2024-12-24' || ' ' || business_end_time || ' ' || timezone)::TIMESTAMPTZ AS end_time_utc
   From Workshops,
 "
) |> filter(
  ## meeting cannot start before the earliest open workshop
  start_time_utc >= max(start_time_utc)
) |>
  pull(start_time_utc)
```

## Day 18

The data for this challenge is the same as for day 8, and requires the same
preparation: replacing `SERIAL` with `INTEGER`. Instead of using the same
(inefficient) recursive function in R as for day 8, here I learned how to use
them in DuckDB to compute the managerial paths. The wording of this challenge
seems confusing and it seemed uncessary to compute the number of peers with the
same manager to find the answer.

```r
con_day18 <- dbConnect(
  duckdb(),
  "2024-advent-of-sql-data/advent_day_18.duckdb"
)

dbGetQuery(
  con_day18,
  "
  WITH RECURSIVE path_tbl(staff_id, path) AS (
      SELECT staff_id, [manager_id] AS path
      FROM staff
      WHERE manager_id IS NULL
    UNION ALL
      SELECT staff.staff_id, list_prepend(staff.manager_id, path_tbl.path)
      FROM staff, path_tbl
      WHERE staff.manager_id = path_tbl.staff_id
  )
  SELECT path_tbl.staff_id, staff.manager_id, len(path) AS level
  FROM path_tbl
  JOIN staff ON staff.staff_id = path_tbl.staff_id
  ORDER BY path_tbl.staff_id, level DESC
  "
) |>
  mutate(
    total_peers_same_level = n(),
    .by = c(level)
  ) |> 
  arrange(desc(total_peers_same_level), level, staff_id)

dbDisconnect(con_day18, shutdown = TRUE)
```


## Day 19

Replace `SERIAL` with a `SEQUENCE`.


```sql
DROP TABLE IF EXISTS employees CASCADE;

CREATE OR REPLACE SEQUENCE eid START 1;
CREATE TABLE employees (
employee_id INTEGER PRIMARY KEY DEFAULT nextval('eid'),
name VARCHAR(100) NOT NULL,
salary DECIMAL(10, 2) NOT NULL,
year_end_performance_scores INTEGER[] NOT NULL
);
```

The main challenge here, is that the result is a large number, and by default R
does not print enough significant digits to get the correct answer. There are
several ways to get the correct number displayed but, in the end, used
`tibble::num()` which was new to me.

```r
con_day19 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_19.duckdb")

dbExecute(
  con_day19,
  "CREATE OR REPLACE VIEW average_score AS
    (SELECT
    *,
    year_end_performance_scores[len(year_end_performance_scores)] AS last_score
   FROM employees)
")

avg_score <- tbl(con_day19, "average_score") |>
  summarize(avg_score = mean(last_score)) |>
  pull(avg_score)

res <- tbl(con_day19, "average_score") |>
  mutate(gets_bonus = last_score >= avg_score) |>
  mutate(total_comp = case_when(
    gets_bonus ~ round(salary * 1.15, 2),
    TRUE ~ salary
  )) |> 
  summarize(total = sum(total_comp)) |>
  pull(total)

tibble::num(res, digits = 2)

dbDisconnect(con_day19, shutdown = TRUE)
```

## Day 20

Replace `SERIAL` with `SEQUENCE`, so the beginning of the file looks like:

```sql
DROP TABLE IF EXISTS web_requests CASCADE;
CREATE OR REPLACE SEQUENCE id START 1;
CREATE TABLE web_requests (
  request_id INTEGER PRIMARY KEY DEFAULT nextval('id'),
  url TEXT NOT NULL
  );
```

```r
con_day20 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_20.duckdb")

dbGetQuery(
  con_day20,
  "SELECT
    *,
    string_split(regexp_extract(url, '\\?(.+)', 1), '&') AS query
   FROM web_requests
   WHERE contains(url, 'utm_source=advent-of-sql')
   ORDER BY len(list_distinct(list_transform(query, p -> p.split('=')[1]))) DESC, url
   LIMIT 1
  ") |>
  pull(url)

dbDisconnect(con_day20, shutdown = TRUE)
```

## Day 21


Replace `SERIAL` with `SEQUENCE`, so the beginning of the file looks like:

```sql
DROP TABLE IF EXISTS sales CASCADE;

CREATE OR REPLACE SEQUENCE id START 1;
CREATE TABLE sales (
  id INTEGER PRIMARY KEY DEFAULT nextval('id'),
  sale_date DATE NOT NULL,
  amount DECIMAL(10, 2) NOT NULL
);
```

```r
con_day21 <- dbConnect(duckdb(), "2024-advent-of-sql-data/advent_day_21.duckdb")

dbGetQuery(
  con_day21,
  "
  SELECT *
  FROM (
    SELECT
      year(sale_date) AS year,
      quarter(sale_date) AS quarter,
      sum(amount) AS total_sale,
      lag(total_sale, 1) OVER (ORDER BY year, quarter) AS prev_sale,
      (total_sale-prev_sale)/prev_sale AS growth
    FROM sales
    GROUP BY year, quarter
    ORDER BY year, quarter
  )
  ORDER BY growth DESC
")

dbDisconnect(con_day21, shutdown = TRUE)
```

## Day 22

Once again, I used `SEQUENCE` to replace `SERIAL`:

```sql
DROP TABLE IF EXISTS elves CASCADE;
CREATE OR REPLACE SEQUENCE id START 1;
CREATE TABLE elves (
  id INTEGER PRIMARY KEY DEFAULT nextval('id'),
  elf_name VARCHAR(255) NOT NULL,
  skills TEXT NOT NULL
);
```

```r
con_day22 <- dbConnect(
  duckdb(),
  "2024-advent-of-sql-data/advent_day_22.duckdb"
)

dbGetQuery(
  con_day22,
  "
  SELECT
     count(id)
   FROM elves
   WHERE str_split(skills, ',').list_contains('SQL')
   "
)

dbDisconnect(con_day22, shutdown = TRUE)
```

## Day 23

The data could be imported as provided, and chose to solve the challenge with
dplyr.

```r
con_day23 <- dbConnect(
  duckdb(),
  "2024-advent-of-sql-data/advent_day_23.duckdb"
)

seq_id <- tbl(con_day23, "sequence_table") |>
  collect()

full_seq <- tibble(id = seq(min(seq_id$id), max(seq_id$id)))

## join complete and provided sequence and keep both
left_join(full_seq, seq_id, keep = TRUE) |>
  ## the NAs are the gaps
  filter(is.na(id.y)) |>
  ## identify groups
  mutate(next_id = id.x - lag(id.x, default = 1)) |>
  mutate(next_id = cumsum(next_id != 1)) |>
  ## format as expected
  nest_by(next_id) |>
  mutate(res = paste(data$id.x, collapse = ","))

dbDisconnect(con_day23, shutdown = TRUE)
```

## Day 24

The data could be imported directly into DuckDB. For this challenge,
manipulating the data with the dplyr verbs felt like the most efficient way of
solving it.

```r
con_day24 <- dbConnect(
  duckdb(),
  "2024-advent-of-sql-data/advent_day_24.duckdb"
)

tbl(con_day24, "user_plays") |>
  left_join(tbl(con_day24, "songs")) |>
  mutate(has_skip = as.integer(duration < song_duration)) |>
  summarize(
    n_plays = n(),
    n_skips = sum(has_skip),
    .by = song_title
  ) |>
  arrange(desc(n_plays), n_skips)

dbDisconnect(con_day24, shutdown = TRUE)
```