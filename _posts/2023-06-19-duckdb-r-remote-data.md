---
title: "How to work with remote Parquet files with the duckdb R package?"
excerpt: "Learn how to work with Parquet files over HTTPS using duckdb and dplyr."
layout: "single"
date: "2023-06-19"
type: "post"
published: true
categories: ["Hacking"]
tags: ["r", "arrow", "duckdb"]
keep-yaml: true
validate-yaml: false
header:
  overlay_image: "/images/2023-06-bw-ducks.jpg"
  overlay_filter: 0.5
  caption: "Photo by [Zetton Zhang](https://unsplash.com/@zettonzhang?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/photos/kVkW6tCwcfI?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
format:
  gfm:
    reference-location: document
---


For large datasets, it is sometimes convenient to explore them without
downloading them locally. With Arrow, you can work with these remotes files if
they are stored in AWS S3 or Google Cloud Storage. It is however not yet
possible for files stored over HTTPS (it is on the roadmap). On the other hand,
with the "httpfs" extension, DuckDB allows you to query over the wire these
Parquet files.

You can even set things up so you can use dplyr verbs to work with these remote
files. I will demonstrate this using a Parquet version of the [penguins
dataset](https://allisonhorst.github.io/palmerpenguins/) hosted on my site.


Let's start by loading the required packages:

```r
library(DBI)
library(duckdb)
library(dplyr)
```

We are creating a `con` object to hold our DuckDB connection:


```r
con <- duckdb::duckdb()
```

Let's install (only needed once) and load the `httpfs` extension:

```r
dbExecute(con, "INSTALL httpfs;")
dbExecute(con, "LOAD httpfs;")
```

At this point, we could use DuckDB's SQL syntax to work with our remote dataset:


```r
dbGetQuery(con,
  "SELECT species,
          AVG(bill_length_mm) AS avg_bill_length,
          AVG(bill_depth_mm) AS avg_bill_depth
   FROM PARQUET_SCAN('https://francoismichonneau.net/assets/data/penguins.parquet')
   GROUP BY species;")
```

```
# A tibble: 3 × 3
  species   avg_bill_length avg_bill_depth
  <chr>               <dbl>          <dbl>
1 Adelie               38.8           18.3
2 Gentoo               47.5           15.0
3 Chinstrap            48.8           18.4
```

However, you can create a view using this remote file, which in turn, will allow
you to use dplyr to query your file:

```r
dbExecute(con,
  "CREATE VIEW penguins AS
   SELECT * FROM PARQUET_SCAN('https://francoismichonneau.net/assets/data/penguins.parquet');
")
```

You can check it worked by running:


```r
dbListTables(con)
```

```
[1] "penguins"
```

Now you can work with this remote data with dplyr:


```r
tbl(con, "penguins") |>
  group_by(species) |>
  summarize(
    avg_bill_length = mean(bill_length_mm),
    avg_bill_depth = mean(bill_depth_mm)
  )
```


```
# Source:   SQL [3 x 3]
# Database: DuckDB 0.8.1 [francois@Linux 6.2.0-20-generic:R 4.3.0/:memory:]
  species   avg_bill_length avg_bill_depth
  <chr>               <dbl>          <dbl>
1 Adelie               38.8           18.3
2 Gentoo               47.5           15.0
3 Chinstrap            48.8           18.4
```