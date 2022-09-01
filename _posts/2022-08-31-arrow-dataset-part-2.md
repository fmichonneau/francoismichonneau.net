---
title: "Creating an Arrow dataset (part 2)"
excerpt: "How does partitioning impacts query performance?"
layout: "single"
date: "2022-08-31"
type: "post"
published: true
categories: ["Arrow exploration"]
tags: ["r", "arrow"]
keep-yaml: true
validate-yaml: false
header:
  overlay_image: "/images/2022-08-22-lots-of-arrows.jpg"
  overlay_filter: 0.5
  caption: "Photo by [Possessed Photograph](https://unsplash.com/@possessedphotography?utm_source=unsplash&utm_medium=ref
  erral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/arrows?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
format:
  gfm:
    reference-location: document
---

## Background

In this follow up post (see
[part 1]({% post_url 2022-08-22-arrow-dataset-creation %}) if you missed
it), we will explore what happens to the performance if we read the
files straight into Arrow instead of downloading them locally first?

## Reading remote CSV files

In the first part, we first downloaded the compressed CSV files locally
(using the `download.file()` function) and then we used the
`open_dataset()` function on this set of files to make it available to
Arrow.

However, it is possible to bypass the local download, and instead import
the files directly over an Internet connection using the
`read_csv_arrow()` function and providing the file URL as the first
argument. Once the file is loaded in memory, we can then write it to
disk in the parquet format (based on what we learned in part 1).

We can then modify the code from the `download_daily_package_logs_csv()`
function from part 1 to the following (line changed are highlighted).

``` r
library(tidyverse)
library(arrow)
```

``` r
## Download the data set for a given date from the RStudio CRAN log website.
## `date` is a single date for which we want the data
## `path` is where we want the data to live
download_daily_package_logs_parquet <- function(date,
                                                path = "~/datasets/cran-logs-parquet-by-day") {

  ## build the URL for the download
  date <- parse_date(date)
  url <- paste0(
    'http://cran-logs.rstudio.com/', date$year, '/', date$date_chr, '.csv.gz'
  )

  ## build the path for the destination of the download
  file <- file.path(
    path,
    paste0("year=", date$year),
    paste0("month=", date$month),
    paste0(date$date_chr, ".parquet")   # <--- change extension to .parquet
  )

  ## create the folder if it doesn't exist
  if (!dir.exists(dirname(file))) {
    dir.create(dirname(file), recursive = TRUE)
  }

  ## download the file
  message("Downloading data for ", date$date_chr, " ... ", appendLF = FALSE)
    arrow::read_csv_arrow(url) %>%      # <--- read directly from URL
      arrow::write_parquet(sink = file) # <--- convert to parquet on disk
  message("done.")

  ## quick check to make sure that the file was created
  if (!file.exists(file)) {
    stop("Download failed for ", date$date_chr, call. = FALSE)
  }

  ## return the path
  file
}

## This function is unchanged from part 1
## Check that the date is really a date,
## and extract the year and month from it
parse_date <- function(date) {
  stopifnot(
    "`date` must be a date" = inherits(date, "Date"),
    "provide only one date" = identical(length(date), 1L),
    "date must be in the past" = date < Sys.Date()
  )
  list(
    date_chr = as.character(date),
    year = as.POSIXlt(date)$year + 1900L, 
    month = as.POSIXlt(date)$mon + 1L
  )
}
```

Now that we are set up, we can create the file system the same way we
did, in part 1.

``` r
dates_to_get <- seq(
  as.Date("2022-06-01"),
  as.Date("2022-08-15"),
  by = "day"
)

purrr::walk(dates_to_get, download_daily_package_logs_parquet)
```

The result is similar to what we achieved in part 1. We have one file
for each day, placed in a folder corresponding to their month. Except
that this time, instead of having compressed CSV files, we have parquet
files:

    ~/datasets/cran-logs-parquet-by-day/
    └── year=2022
        ├── month=6
        │   ├── 2022-06-01.parquet
        │   ├── 2022-06-02.parquet
        │   ├── 2022-06-03.parquet
        │   ├── ...
        │   └── 2022-06-30.parquet
        ├── month=7
        │   ├── 2022-07-01.parquet
        │   ├── 2022-07-02.parquet
        │   ├── 2022-07-03.parquet
        │   ├── ...
        │   └── 2022-07-31.parquet
        └── month=8
            ├── 2022-08-01.parquet
            ├── 2022-08-02.parquet
            ├── 2022-08-03.parquet
            ├── ...
            └── 2022-08-15.parquet

Let’s check how large this data is compared to the datasets we created
in Part 1:

``` r
dataset_size <- function(path) {
  fs::dir_info(path, recurse = TRUE) %>%
    filter(type == "file") %>%
    pull(size) %>%
    sum()
}

tribble(
  ~ Format, ~ size,
  "Compressed CSV", dataset_size("~/datasets/cran-logs-csv/"),
  "Arrow", dataset_size("~/datasets/cran-logs-arrow/"),
  "Parquet", dataset_size("~/datasets/cran-logs-parquet/"),
  "Parquet by day",  dataset_size("~/datasets/cran-logs-parquet-by-day/")
) 
```

    # A tibble: 4 × 2
      Format                size
      <chr>          <fs::bytes>
    1 Compressed CSV       5.01G
    2 Arrow               29.67G
    3 Parquet              5.06G
    4 Parquet by day       4.63G

The dataset with one parquet file per day, is slightly smaller than when
we let `write_dataset()` do its own partitioning which led to one file
per month.

We can now compare how quickly Arrow can read these datasets.

``` r
bench::mark(
  parquet = open_dataset("~/datasets/cran-logs-parquet", format = "parquet"),
  parquet_by_day = open_dataset("~/datasets/cran-logs-parquet-by-day", format = "parquet"),
  check = FALSE
)
```

    # A tibble: 2 × 6
      expression          min   median `itr/sec` mem_alloc `gc/sec`
      <bch:expr>     <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    1 parquet         148.2ms 181.07ms      5.56    7.91MB     0   
    2 parquet_by_day    3.8ms   4.35ms    224.      4.28KB     4.23

Even though there are more files to parse (76 vs. 3), loading the
dataset with a parquet file per day is a bit faster.

``` r
cran_logs_parquet <- open_dataset("~/datasets/cran-logs-parquet",  format = "parquet")
cran_logs_parquet_by_day <- open_dataset("~/datasets/cran-logs-parquet-by-day",  format = "parquet")
```

Let’s now explore the performance of a few queries on these datasets.

How long does it take to compute the number of rows in these datasets:

``` r
bench::mark(
  parquet = nrow(cran_logs_parquet),
  parquet_by_day = nrow(cran_logs_parquet)
)
```

    # A tibble: 2 × 6
      expression          min   median `itr/sec` mem_alloc `gc/sec`
      <bch:expr>     <bch:tm> <bch:tm>     <dbl> <bch:byt>    <dbl>
    1 parquet           744µs    782µs     1254.    4.74KB     8.54
    2 parquet_by_day    748µs    783µs     1251.    1.97KB     8.46

Not much of a difference.

Let’s compare the performance of the query we ran in part 1, where we
computed the 10 most downloaded packages in the time period covered by
our dataset.

``` r
top_10_packages <- function(data) {
  data %>%
    count(package, sort = TRUE) %>%
    head(10) %>%
    mutate(n_million_downloads = n/1e6) %>%
    select( - n) %>% 
    collect()
}

bench::mark(
  top_10_packages(cran_logs_parquet),
  top_10_packages(cran_logs_parquet_by_day)
)
```

    # A tibble: 2 × 6
      expression                                     min   median `itr/sec` mem_al…¹
      <bch:expr>                                <bch:tm> <bch:tm>     <dbl> <bch:by>
    1 top_10_packages(cran_logs_parquet)           3.46s    3.46s     0.289   7.19MB
    2 top_10_packages(cran_logs_parquet_by_day)    4.95s    4.95s     0.202 165.36KB
    # … with 1 more variable: `gc/sec` <dbl>, and abbreviated variable name
    #   ¹​mem_alloc
    # ℹ Use `colnames()` to see all variable names

This query runs 2 seconds faster on the dataset with one parquet file
per month compared to the dataset with one parquet file per day.

The way a dataset is partitioned has an impact on the performance of
queries. If you are filtering your dataset along a variable used in the
partitioning, some of the files can be skipped. Arrow can directly and
only read the file(s) with the relevant information for your query. For
instance, if you are performing a query that only touches the month of
July, Arrow does not need to look at the files for June or August,
leading to potential speed ups.

Would the partitioning by day help us run our query faster if we were to
compute the 10 most downloaded package for a single day? After all, in
this case, we would only need to look at one of the files in our folder
of parquet files, and the file in question would be smaller than the one
that has all the data for the month. Let’s compare the performance of
this query for August 1st, 2022:

``` r
top_10_packages_by_day <- function(data) {
  data %>%
    filter(date == as.Date("2022-08-01")) %>%
    count(package, sort = TRUE) %>%
    head(10) %>%
    mutate(n_million_downloads = n/1e6) %>%
    select( - n) %>%
    collect()
}

bench::mark(
  top_10_packages_by_day(cran_logs_parquet),
  top_10_packages_by_day(cran_logs_parquet_by_day)
)
```

    # A tibble: 2 × 6
      expression                                          min median itr/s…¹ mem_a…²
      <bch:expr>                                       <bch:> <bch:>   <dbl> <bch:b>
    1 top_10_packages_by_day(cran_logs_parquet)         299ms  299ms    3.34   262KB
    2 top_10_packages_by_day(cran_logs_parquet_by_day)  380ms  393ms    2.54   219KB
    # … with 1 more variable: `gc/sec` <dbl>, and abbreviated variable names
    #   ¹​`itr/sec`, ²​mem_alloc
    # ℹ Use `colnames()` to see all variable names

Interestingly, running the query on the monthly parquet file is still
faster. It takes about 60% longer to run the queries on the one parquet
file per day. The overhead associated with having too many small files
in this situation does not compensate for having to look inside a single
file to perform this operation. For the benefits of partitioning to be
visible, we would need to have more data in each parquet file.

We don’t see a performance benefit of having many small files even when
we try to get the result on a single day. But how does this partitioning
impact the performance of a query that needs to access multiple random
rows? Let’s compare how a query that looks at the number of downloads
per day for a given package.

``` r
package_downloads_by_day <- function(data, pkg = "arrow") {
  data %>%
    filter(package == pkg) %>%
    count(date) %>%
    arrange(date) %>%
    collect()
}

bench::mark(
  package_downloads_by_day(cran_logs_parquet),
  package_downloads_by_day(cran_logs_parquet_by_day)
)
```

    Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    # A tibble: 2 × 6
      expression                                              min   median `itr/sec`
      <bch:expr>                                         <bch:tm> <bch:tm>     <dbl>
    1 package_downloads_by_day(cran_logs_parquet)           2.99s    2.99s     0.335
    2 package_downloads_by_day(cran_logs_parquet_by_day)    4.32s    4.32s     0.231
    # … with 2 more variables: mem_alloc <bch:byt>, `gc/sec` <dbl>
    # ℹ Use `colnames()` to see all variable names

In this case, it still takes about 60% longer to perform this query.

## Conclusion

This small example illustrates that it might be worth exploring how best
to partition your dataset to benefit the most from the speed that Arrow
brings to your queries. The variables you include in your queries have
also a role to play when deciding how to partition your dataset, and it
might be best to partition your dataset according to variables you use
most often in your queries.

The useR!2022 Arrow tutorial has a [convincing
demonstration](https://arrow-user2022.netlify.app/data-storage.html#multi-file-data-sets)
that taking advantage of partitioning for your queries makes them run
much faster.

<details>
<summary>
Expand for Session Info
</summary>

``` r
sessioninfo::session_info()
```

    ─ Session info ───────────────────────────────────────────────────────────────
     setting  value
     version  R version 4.2.1 (2022-06-23)
     os       Ubuntu 22.04.1 LTS
     system   x86_64, linux-gnu
     ui       X11
     language en_US
     collate  en_US.UTF-8
     ctype    en_US.UTF-8
     tz       Europe/Paris
     date     2022-09-01
     pandoc   NA (via rmarkdown)

    ─ Packages ───────────────────────────────────────────────────────────────────
     package       * version date (UTC) lib source
     arrow         * 9.0.0   2022-08-10 [1] CRAN (R 4.2.1)
     assertthat      0.2.1   2019-03-21 [1] RSPM
     backports       1.4.1   2021-12-13 [1] RSPM
     bench           1.1.2   2021-11-30 [1] RSPM
     bit             4.0.4   2020-08-04 [1] RSPM
     bit64           4.0.5   2020-08-30 [1] RSPM
     broom           1.0.0   2022-07-01 [1] RSPM
     cellranger      1.1.0   2016-07-27 [1] RSPM
     cli             3.3.0   2022-04-25 [1] RSPM (R 4.2.0)
     colorspace      2.0-3   2022-02-21 [1] RSPM
     crayon          1.5.1   2022-03-26 [1] RSPM
     DBI             1.1.3   2022-06-18 [1] RSPM
     dbplyr          2.2.1   2022-06-27 [1] RSPM
     digest          0.6.29  2021-12-01 [1] RSPM
     dplyr         * 1.0.9   2022-04-28 [1] RSPM
     ellipsis        0.3.2   2021-04-29 [1] RSPM
     evaluate        0.15    2022-02-18 [1] RSPM
     fansi           1.0.3   2022-03-24 [1] RSPM
     fastmap         1.1.0   2021-01-25 [1] RSPM
     forcats       * 0.5.1   2021-01-27 [1] RSPM
     fs              1.5.2   2021-12-08 [1] RSPM
     gargle          1.2.0   2021-07-02 [1] RSPM
     generics        0.1.3   2022-07-05 [1] RSPM
     ggplot2       * 3.3.6   2022-05-03 [1] RSPM
     glue            1.6.2   2022-02-24 [1] RSPM (R 4.2.0)
     googledrive     2.0.0   2021-07-08 [1] RSPM
     googlesheets4   1.0.0   2021-07-21 [1] RSPM
     gtable          0.3.0   2019-03-25 [1] RSPM
     haven           2.5.0   2022-04-15 [1] RSPM
     hms             1.1.1   2021-09-26 [1] RSPM
     htmltools       0.5.3   2022-07-18 [1] RSPM
     httr            1.4.3   2022-05-04 [1] RSPM
     jsonlite        1.8.0   2022-02-22 [1] RSPM
     knitr           1.39    2022-04-26 [1] RSPM
     lifecycle       1.0.1   2021-09-24 [1] RSPM
     lubridate       1.8.0   2021-10-07 [1] RSPM
     magrittr        2.0.3   2022-03-30 [1] RSPM
     modelr          0.1.8   2020-05-19 [1] RSPM
     munsell         0.5.0   2018-06-12 [1] RSPM
     pillar          1.8.0   2022-07-18 [1] RSPM
     pkgconfig       2.0.3   2019-09-22 [1] RSPM
     profmem         0.6.0   2020-12-13 [1] RSPM
     purrr         * 0.3.4   2020-04-17 [1] RSPM
     R6              2.5.1   2021-08-19 [1] RSPM
     readr         * 2.1.2   2022-01-30 [1] RSPM
     readxl          1.4.0   2022-03-28 [1] RSPM
     reprex          2.0.1   2021-08-05 [1] RSPM
     rlang           1.0.4   2022-07-12 [1] RSPM (R 4.2.0)
     rmarkdown       2.14    2022-04-25 [1] RSPM
     rvest           1.0.2   2021-10-16 [1] RSPM
     scales          1.2.0   2022-04-13 [1] RSPM
     sessioninfo     1.2.2   2021-12-06 [1] RSPM
     stringi         1.7.8   2022-07-11 [1] RSPM
     stringr       * 1.4.0   2019-02-10 [1] RSPM
     tibble        * 3.1.8   2022-07-22 [1] RSPM
     tidyr         * 1.2.0   2022-02-01 [1] RSPM
     tidyselect      1.1.2   2022-02-21 [1] RSPM
     tidyverse     * 1.3.2   2022-07-18 [1] RSPM
     tzdb            0.3.0   2022-03-28 [1] RSPM
     utf8            1.2.2   2021-07-24 [1] RSPM
     vctrs           0.4.1   2022-04-13 [1] RSPM
     withr           2.5.0   2022-03-03 [1] RSPM
     xfun            0.31    2022-05-10 [1] RSPM
     xml2            1.3.3   2021-11-30 [1] RSPM
     yaml            2.3.5   2022-02-21 [1] RSPM

     [1] /home/francois/.R-library
     [2] /usr/lib/R/library

    ──────────────────────────────────────────────────────────────────────────────

</details>
