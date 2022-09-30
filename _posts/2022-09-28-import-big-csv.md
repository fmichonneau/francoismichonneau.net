---
title: "Working with large CSV files with Arrow"
excerpt: "A short practical guide to load a 15 GB dataset in R and Python."
layout: "single"
date: "2022-09-28"
type: "post"
published: true
categories: ["Arrow exploration"]
tags: ["r", "arrow"]
header:
  overlay_image: /images/2022-09-light-arrows.jpg
  overlay_filter: 0.5
  caption: "Photo by [Andryck Lopez](https://unsplash.com/@lopez1010?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/)"
toc: true
---


Lucky you! You just got hold of a largish CSV file (let's say 15 GB,
about 140 million rows). How do you handle this file to be able to
work with it using Apache Arrow?

Going through the documentation of Arrow, you might notice that
several ways are mentioned to import data. They fall into two
families:
- One is the **Dataset API** and;
- the other is the one that I will refer to as the **Single file** API.

The Dataset API is very flexible.  It can read multiple file formats,
you can point to a folder with multiple files and create a dataset
from them, it can read datasets from multiple sources (even combining
remote and local sources). This API can also be used to read single
files that are too large to fit in memory. This works because the
files are not loaded in memory. The functions scan the content so they
know where to look for the data and what is the schema (the data types
and names of each column). When you query the data, there is a little
overhead because the query engine needs to first read the data before
it can operate on it. (If you want to see some examples of what the
Dataset API can do, check out the previous posts).

The Single file API groups a series of functions that are specific to
each file format (CSV, JSON, Parquet, Feather/Arrow, ORC). They work
on one file at a time, and they load the data in memory. So depending
on the size of your file and the amount of memory you have available
on your system, it might not be possible to load the dataset this
way. If you can load this dataset, queries will however run faster
because the data is already in memory and will be quick to access for
the query engine.

In this post, we will explore how to convert a large CSV in the Apache
Parquet format using the Dataset and the Single file APIs with code
examples in R and Python.

## The Single file API in R

The functions in the Single file API in R start with `read_` or
`write_` followed by the name of the file format. For instance,
`read_csv_arrow()`, `read_parquet()`, `read_feather()`.

To read the data with our 15 GB CSV file, we would use:

```r
library(arrow)

data <- read_csv_arrow(
  "~/dataset/path_to_file.csv",
  as_data_frame = FALSE
)
```

Using `as_data_frame = FALSE` keeps the result as an Arrow table which
is a better representation for a file of this size. Attempting to
convert it into a data frame will most likely cause you to run out of
memory. This step takes about 15s on my system.

At this point, you have an Arrow formatted data to work with. You can
stop here and do your analysis. 

However, we saw in the previous post that Apache Parquet offered a
good balance between file size and reading speed. To convert this file
into Parquet using the Single file API, you would use:

```r
write_parquet(data, "~/dataset/data.parquet")
```

Creating this file takes about 85 seconds on my system. The resulting
file is about 9.5 GB. You can then load it again the next time you
need to work with it usiing:

```r
data <- read_parquet("~/dataset/data.parquet", as_data_frame = FALSE)
```

Reading this file takes about 25 seconds.

A query looking at the number of occurences for each of the 10
distinct values in a column takes half a second on the dataset loaded
in memory (it is about the same timing whether the data is loaded from
`read_csv_arrow()` or from `read_parquet()` because the in-memory
representation is the same).

Let's now use the Dataset API.

## The Dataset API in R

We will read the large CSV file with `open_dataset()`. This function can
be pointed to a folder with lots of files but it can also read a
single file.

```r
data <- open_dataset("~/dataset/path_to_file.csv")
```

With my 15GB file, it takes 0.05 seconds to "read" the file. It is
fast because the data does not get loaded in memory. `open_dataset()`
scans the content of the file, figures out the name of the columns and
their data types.

So running the same query as above, counting the number of unique
values in a column, takes 18 seconds. It is slow because to perform
this query, the query engine needs to actually read the data. It is
the same result that we had found in a previous post. Running queries
directly on a CSV file is slow. In that post, we had also found that
storing the data in the parquet format sped things up. Let's now
convert this dataset into the Parquet format using the Dataset API.

Partitioning a Parquet dataset could help with query performance. From
the example using the Single file API, we saw that we did not get
great performance with a single Parquet file. The particular dataset I
have on hands does not have any obvious variable we can use to
partition the data. Instead, we can use the `max_rows_per_file`
argument to limit how large each parquet file is. I found that (at
least with the dataset I'm currently working with), limiting the
number of rows to 10 million per file seemed like a good
compromise. Each file is about 720 MB which is close to the file sizes
in the NYC taxi dataset which is partitioned by year and month.

```r
write_dataset(
  data,
  format = "parquet",
  path = "~/datasets/my-data/",
  max_rows_per_file = 1e7
)
```

Writing these files on my system takes about 50 seconds.

Next time we want to analyze this data, we can load these files with:

```r
data <- open_dataset(
  "~/datasets/my-data"
  )
```

It takes about the same amount of time as scanning the CSV files (0.02
seconds).

Counting the number of occurences for each of the 10 unique values in
a column takes just 1 second. One second to summarize 140 million
rows. It is a little slower than performing this query when the entire
dataset is loaded in memory but loading the files is much faster.

One of the advantages of the Arrow ecosystem is that the approach that
works with R also works with Python. And because both language use the
same C++ backend, the code looks very similar.


## Single file API in Python

There are two functions in the Python Single API to read CSV files:
`read_csv()` and `open_csv()`. While `read_csv()` loads all the data
in memory, and does it fast by using multiple threads to read
different part of the files, `open_csv` reads the data in batches and
uses a single thread.

If the CSV file is small enough, you should use `read_csv`. The code
to read the CSV file and write it to a Parquet file would then look
like this:

```python
import pyarrow as pa
import pyarrow.csv
import pyarrow.parquet as pq

in_path = '~/datasets/data.csv'
out_path = '~/datasets/data.parquet'

data =  pa.csv.read_csv(in_path)

pq.write_table(data, out_path)
```

In our case, the file is too large to fit in memory. So instead of
using `read_csv()`, we need to use `open_csv()`. Because, the CSV file
is read in chunks, the code is a little more complex. We loop through
each chunk, read it, and write it to the Parquet file. This uses
little memory but can be a little slow as a single thread is used to
read the file. When using `open_csv()`, the data types need to be
consistent in your columns. The inferrence is made on the first chunk,
and if the type changes halfway through your dataset, you will run
into errors. You can avoid this by specifying the data types manually.

```python
# from https://stackoverflow.com/a/68563617/1113276
import pyarrow as pa
import pyarrow.parquet as pq
import pyarrow.csv

in_path = '~/datasets/data.csv'
out_path = '~/datasets/data.parquet'

writer = None
with pyarrow.csv.open_csv(in_path) as reader:
    for next_chunk in reader:
        if next_chunk is None:
            break
        if writer is None:
            writer = pq.ParquetWriter(out_path, next_chunk.schema)
        next_table = pa.Table.from_batches([next_chunk])
        writer.write_table(next_table)
writer.close()
```

On my system, the conversion from the CSV to Parquet takes about 190
seconds. Reading the parquet file can be done with:

```python
data = pq.ParquetDataset(out_path).read()
```

With this approach the dataset is in memory, and for this large
dataset, on my system, I need to be careful with what is running on my
system (I can't have my web browser open for instance) to be able to
load this without running out of memory.

## The Dataset API in Python

To load the CSV file with the Dataset API:

```python
import pyarrow.dataset as ds

in_path = "~/datasets/data.csv"
out_path = "~/datasets/my-data/"

data = ds.dataset(in_path)
```

Just like with R, reading this file takes about 0.02 seconds.

To convert it to parquet, you use the `write_dataset()` function:

```python
ds.write_dataset(data, out_path,
                 format = "parquet",
                 max_rows_per_file = 1e7)
```

And reading the collection of parquet files can also be done with the
`dataset()` function, just like when we used the function to read the
CSV file. The `format` argument is optional as the function detects
automatically the file type.


```python
data = ds.dataset(out_path, format = "parquet")
```

## What I didn't talk about

I focused on the reading of a CSV file and its conversion to
Parquet. I didn't talk about all the options that both the Single file
and the Dataset APIs have to customize the format of the files that
are being imported. For instance, both APIs can be used to specify a
different column-separator, and cell content that should be treated as
missing data.

## Conclusion

Unless you are working with a relatively small dataset stored as a
single file, using the Dataset API provides you with flexibility to
read and write datasets quickly and in a variety of formats. The
approach described here would work just as well with a much larger
dataset.

To summarize what we learned in this post, I put together this brief
decision guide to help you choose the correct API to import your
data. This should help you get started. However, you might need to
change which API you use depending on the specificities of your
dataset, your system, and the type of analyses you will perform on the
dataset.

{% include figure image_path="/images/2022-09-decision-map.webp" alt="Decision
tree to help you **choose** the most suitable API for your data. If
your dataset is large (more than a third of your available RAM) or if
it is split into multiple files use the Dataset API. Reserve the use
of the Single file API when the dataset is small." caption="Decision
tree to help you choose the appropriate Apache Arrow API for your
dataset." %}

## Acknowledgments

Thank you to [Danielle Navarro](https://djnavarro.net) for
reviewing this post.
