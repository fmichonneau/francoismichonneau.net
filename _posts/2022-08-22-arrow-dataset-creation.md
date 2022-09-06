---
layout: single
title: "Creating an Arrow dataset"
date: 2022-08-22
type: post
published: true
header:
  overlay_image: /images/2022-08-22-lots-of-arrows.jpg
  overlay_filter: 0.5
  caption: "Photo by [Possessed Photograph](https://unsplash.com/@possessedphotography?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/arrows?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)"
categories: ["Arrow Exploration"]
tags: ["r", "arrow"]
excerpt: "An exploration of the file formats that Arrow can read and write."
---


## Background

While getting started with Apache Arrow, I was intrigued by the variety
of formats Arrow supports. Arrow tutorials tend to start with already
prepared datasets ready to be ingested by `open_dataset()`. I wanted to
explore what it takes to create your dataset aimed to be analyzed with
Arrow and understand the respective benefits of the different file
formats it supports.

Arrow can read in a variety of formats: `parquet`, `arrow` (also known
as `ipc` and `feather`)[^1], and text-based formats like `csv` (as well
as `tsv`). Additionally, Arrow provides tools to convert between these
formats.

Having the possibility to import datasets in a variety of formats is
helpful as you are less constrained by the type of data you can start
your analysis on. However, if you are building a dataset from scratch,
which one should you choose?

To try to answer this question, we will be using the `{arrow}` R package
to compare the amount of hard drive space these file formats use and the
performance of a query in a multi-file dataset using these different
formats. This is not a formal evaluation of the performance of Arrow or
how best to optimize the partitioning of a dataset, rather it is a brief
exploration of the tradeoffs that come with using the different datasets
supported by Arrow. I also don’t explain the differences in the data
structure of these different formats.

## The dataset

We will be using data from <http://cran-logs.rstudio.com/>. This site
gives you access to the log files for all hits to the CRAN[^2] mirror
hosted by RStudio. For each day since October 1st, 2012, there is a
compressed CSV file (file with the extension `.csv.gz`) that records the
downloaded packages. Each row contains the date, the time, the name of
the R package downloaded, the R version used, the architecture (32-bit
or 64-bit), the operating system, the country inferred from the IP
address, and a daily unique identifier assigned to each IP address. This
website has also similar data for the daily downloads of R itself but I
will not be using this data in this post.

For this exploration, we are going to limit ourselves to a couple of
months of data which will be providing enough data for our purpose. We
will download the data for the period from June 1st, 2022 to August
15th, 2022.

Arrow is designed to read data that is split across multiple files. So,
you can point `open_dataset()` to a directory that contains all the
files that make up your dataset. There is no need to loop over each file
to build your dataset in memory. Splitting your datasets across multiple
files can even make queries on your dataset faster, as only some of the
files might need to be accessed to get the results needed. Depending on
the type of queries you perform most often on your dataset, it can be
worth considering how best to partition your files to accelerate your
analyses (but this is beyond the scope of this post). Here, the files
are provided by date, and we will keep a time-based file organization.

We will use a [Hive-style](https://hive.apache.org/) partitioning by
year and month. We will have a directory for each year (there is only
one year in our example), and within it, a directory for each month. The
directory are named according to the convention
`<variable_name>=<value>`. So we will want to organize the files as
illustrated below:

    └── year=2022
        ├── month=6
        │   └── <data files>
        ├── month=7
        │   └── <data files>
        └── month=8
            └── <data files>

## Import the data as it is provided

``` r
library(arrow)
library(tidyverse)
library(fs)
library(bench)
```

The `open_dataset()` function in the `{arrow}` package can directly read
compressed CSV files[^3] (with the extension `.csv.gz`) as they are
provided on the RStudio CRAN logs website.

As a first step, we can download the files from the site and organize
them using the Hive-style directory structure as shown above.

``` r
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

## Download the data set for a given date from the RStudio CRAN log website.
## `date` is a single date for which we want the data
## `path` is where we want the data to live
download_daily_package_logs_csv <- function(date,
                                            path = "~/datasets/cran-logs-csv") {

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
    paste0(date$date_chr, ".csv.gz")
  )

  ## create the folder if it doesn't exist
  if (!dir.exists(dirname(file))) {
    dir.create(dirname(file), recursive = TRUE)
  }

  ## download the file
  message("Downloading data for ", date$date_chr, " ... ", appendLF = FALSE)
  download.file(
    url = url,
    destfile = file,
    method = "libcurl",
    quiet = TRUE,
    mode = "wb"
  )
  message("done.")

  ## quick check to make sure that the file was created
  if (!file.exists(file)) {
    stop("Download failed for ", date$date_chr, call. = FALSE)
  }

  ## return the path
  file
}
```

``` r
## build sequence of dates for which we want the data
dates_to_get <- seq(
  as.Date("2022-06-01"),
  as.Date("2022-08-15"),
  by = "day"
)

## download the data
walk(dates_to_get, download_daily_package_logs_csv)
```

Let’s check the content of the folder that holds the data we downloaded:

    ~/datasets/cran-logs-csv/
    └── year=2022
        ├── month=6
        │   ├── 2022-06-01.csv.gz
        │   ├── 2022-06-02.csv.gz
        │   ├── 2022-06-03.csv.gz
        │   ├── ...
        │   └── 2022-06-30.csv.gz
        ├── month=7
        │   ├── 2022-07-01.csv.gz
        │   ├── 2022-07-02.csv.gz
        │   ├── 2022-07-03.csv.gz
        │   ├── ...
        │   └── 2022-07-31.csv.gz
        └── month=8
            ├── 2022-08-01.csv.gz
            ├── 2022-08-02.csv.gz
            ├── 2022-08-03.csv.gz
            ├── ...
            └── 2022-08-15.csv.gz

We have one file for each day, placed in a folder corresponding to their
month. We can now read this data using `{arrow}`’s `open_dataset()`
function:

``` r
cran_logs_csv <- open_dataset(
  "~/datasets/cran-logs-csv/",
  format = "csv",
  partitioning = c("year", "month")
)
cran_logs_csv
```

    FileSystemDataset with 76 csv files
    date: date32[day]
    time: time32[s]
    size: int64
    r_version: string
    r_arch: string
    r_os: string
    package: string
    version: string
    country: string
    ip_id: int64
    year: int32
    month: int32

The partitioning has been taken into consideration as the output shows
that the dataset contains the variables `year` and `month` which are not
part of the data we downloaded. They are coming from the way we
organized the downloaded files.

## Convert to Arrow and Parquet files

Now that we have the compressed CSV files on disk, and that we opened
the dataset with `open_dataset()`, we can convert it to the other file
formats supported by Arrow using `{arrow}`’s `write_dataset()` function.
We are going to convert our collection of `.csv.gz` files into the Arrow
and Parquet formats.

``` r
## Convert the dataset into the Arrow format
write_dataset(
  cran_logs_csv,
  path = "~/datasets/cran-logs-arrow",
  format = "arrow",
  partitioning = c("year", "month")
)

## Convert the dataset into the Parquet format
write_dataset(
  cran_logs_csv,
  path = "~/datasets/cran-logs-parquet",
  format = "parquet",
  partitioning = c("year", "month")
)
```

Let’s inspect the content of the directories that contain these
datasets.

``` r
fs::dir_tree("~/datasets/cran-logs-arrow/")
```

    ~/datasets/cran-logs-arrow/
    └── year=2022
        ├── month=6
        │   └── part-0.arrow
        ├── month=7
        │   └── part-0.arrow
        └── month=8
            └── part-0.arrow

``` r
fs::dir_tree("~/datasets/cran-logs-parquet/")
```

    ~/datasets/cran-logs-parquet/
    └── year=2022
        ├── month=6
        │   └── part-0.parquet
        ├── month=7
        │   └── part-0.parquet
        └── month=8
            └── part-0.parquet

These two directories have the same layout organized by year and month
as with our CSV files given that we kept the same partitioning. The
files within the directories have an extension that matches their file
format. One difference is that there is a single file for each month. We
used the default values for `write_dataset()` and the number of rows for
each month is smaller than the threshold this function uses to split the
dataset into multiple files.

## Comparison of the different formats

Let’s compare how much space these different file formats take on disk:

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
  "Parquet", dataset_size("~/datasets/cran-logs-parquet/")
) 
```

    # A tibble: 3 × 2
      Format                size
      <chr>          <fs::bytes>
    1 Compressed CSV       5.01G
    2 Arrow               29.67G
    3 Parquet              5.06G

The Arrow format takes the most space with almost 30GB while both the
compressed CSV and the Parquet files use about 5GB of hard drive.

We are now set up to compare the performance of doing computation of
these different dataset formats.

Let’s open these datasets with the different formats:

``` r
cran_logs_csv <- open_dataset("~/datasets/cran-logs-csv/", format = "csv")
cran_logs_arrow <- open_dataset("~/datasets/cran-logs-arrow/", format = "arrow")
cran_logs_parquet <- open_dataset("~/datasets/cran-logs-parquet/", format = "parquet")
```

We will compare how long it takes for Arrow to compute the 10 most
downloaded packages in the time period our dataset covers using each
file format.

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
  top_10_packages(cran_logs_csv),
  top_10_packages(cran_logs_arrow),
  top_10_packages(cran_logs_parquet)
)
```

    Warning: Some expressions had a GC in every iteration; so filtering is disabled.

    # A tibble: 3 × 6
      expression                              min   median itr/se…¹ mem_al…² gc/se…³
      <bch:expr>                         <bch:tm> <bch:tm>    <dbl> <bch:by>   <dbl>
    1 top_10_packages(cran_logs_csv)       29.57s   29.57s   0.0338   8.19MB   0    
    2 top_10_packages(cran_logs_arrow)       2.1s     2.1s   0.475  165.39KB   0.475
    3 top_10_packages(cran_logs_parquet)    3.32s    3.32s   0.301  137.11KB   0    
    # … with abbreviated variable names ¹​`itr/sec`, ²​mem_alloc, ³​`gc/sec`

While it takes about 4 seconds to perform this task on the Arrow or
Parquet files, it takes more than 30 seconds to do it on the CSV files.

## Conclusion

Having Arrow point directly to the folder of compressed CSV file might
be the most convenient but it comes with a high-performance cost. Arrow
and Parquet have similar performance but the Parquet files take less
space on disk and would be more suitable for long-term storage. This is
why large datasets like the NYC taxi data is distributed as a series of
Parquet files.

In the future, I might explore how using different variables for
partitioning or how the number of files in the partitions affects the
performance of the queries (EDIT: this [post is now available]({%
post_url 2022-08-31-arrow-dataset-part-2 %}). If you have other ideas
of topics that you would me to explore, do not hesitate to leave a
comment below.

## Going further

If you would like to learn more about the different formats, check out
the [Arrow workshop](https://arrow-user2022.netlify.app/) (especially
[Part 3: Data
Storage](https://arrow-user2022.netlify.app/data-storage.html)) that
Danielle Navarro, Jonathan Keane, and Stephanie Hazlitt taught at
useR!2022.

## Acknowledgments

Thank you to [Kae Suarez](https://twitter.com/kae_suarez/) and [Danielle
Navarro](https://https://djnavarro.net) for reviewing this post.

## Post Scriptum

I wrote a [follow-up post]({% post_url 2022-08-31-arrow-dataset-part-2
%}) that explores the impact of partitioning the dataset on
performance.

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
     date     2022-08-19
     pandoc   NA (via rmarkdown)

    ─ Packages ───────────────────────────────────────────────────────────────────
     package       * version date (UTC) lib source
     arrow         * 9.0.0   2022-08-10 [1] CRAN (R 4.2.1)
     assertthat      0.2.1   2019-03-21 [1] RSPM
     backports       1.4.1   2021-12-13 [1] RSPM
     bench         * 1.1.2   2021-11-30 [1] RSPM
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
     fs            * 1.5.2   2021-12-08 [1] RSPM
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

[^1]: Feather was the first iteration of the file format (v1), the Arrow
    Interprocess Communication (IPC) file format is the newer version
    (v2) and has many new features.

[^2]: Comprehensive R Archive Network, the repository for the R packages

[^3]: since Arrow 9.0.0
