---
title: vroom 1.0.0
author: Jim Hester
date: '2019-05-07'
slug: vroom-1-0-0
description: Introducing the vroom package, extremely fast data import in R.
categories:
  - package
tags: [vroom, r-lib]
photo:
  url: https://www.pexels.com/photo/12801/
  author: Chris Peeters
---



<html>
<style>
h2 code {
    font-size: 1em;
}
</style>
</html>

I'm excited to announce that [vroom 1.0.0](http://vroom.r-lib.org) is now
available on CRAN!

vroom reads rectangular data, such as comma separated
(csv), tab separated (tsv) or fixed width files (fwf) into R. It performs
similar roles to functions like [`readr::read_csv()`](http://readr.r-lib.org),
[`data.table::fread()`](http://r-datatable.com) or `read.csv()`. But for many
datasets `vroom::vroom()` can read them much, much faster (hence the name).

The main reason vroom can be faster is because character data is read from the
file lazily; you only pay for the data you use. This lazy access is done
automatically, so no changes to your R data-manipulation code are needed.

vroom also provides efficient, multi-threaded writing that is multiple times
faster on most inputs than the `readr::write_*()` functions.

Install vroom with:


```r
install.packages("vroom")
```

The best way to get acquainted with the package is the [getting
started](http://vroom.r-lib.org/articles/vroom.html) vignette.

## vroom vs readr

What does the release of vroom mean for readr? For now we plan
to let the two packages evolve separately, but likely we will unite the
packages in the future. One disadvantage to vroom's lazy reading is certain
data problems can't be reported up front, so how best to unify them requires
some thought.

## Reading delimited files

Compared to readr, the first difference you may note is you use only one
function to read the files,
[`vroom()`](http://vroom.r-lib.org/reference/vroom.html). This is because
`vroom()` guesses the delimiter of the file automatically based on the first
few lines (this feature is inspired by a similar feature in
`data.table::fread()`). This works well most of the time, but may fail to guess
properly in some cases. The `delim` argument can be used to specify the
delimiter of the file explicitly.


```r
library(vroom)

data <- vroom("flights.tsv")
#> Observations: 336,776
#> Variables: 19
#> chr  [ 4]: carrier, tailnum, origin, dest
#> dbl  [14]: year, month, day, dep_time, sched_dep_time, dep_delay, arr_time, sched_arr...
#> dttm [ 1]: time_hour
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

The summary message after reading also differs from readr. We hope this output
gives a more informative indication as to whether the types of your columns are
being guessed properly. However you can still retrieve and print the full
column specification with [`spec()`](http://vroom.r-lib.org/reference/spec.html).


```r
spec(data)
#> cols(
#>   year = col_double(),
#>   month = col_double(),
#>   day = col_double(),
#>   dep_time = col_double(),
#>   sched_dep_time = col_double(),
#>   dep_delay = col_double(),
#>   arr_time = col_double(),
#>   sched_arr_time = col_double(),
#>   arr_delay = col_double(),
#>   carrier = col_character(),
#>   flight = col_double(),
#>   tailnum = col_character(),
#>   origin = col_character(),
#>   dest = col_character(),
#>   air_time = col_double(),
#>   distance = col_double(),
#>   hour = col_double(),
#>   minute = col_double(),
#>   time_hour = col_datetime(format = "")
#> )
```

The message will be disabled if you supply a column specification to `col_types` when reading.


```r
s <- spec(data)

data <- vroom("flights.tsv", col_types = s)
```

## Reading multiple files

One feature new to vroom is built-in support for reading sets of files with the
same columns into one table. Just pass the filenames to be read directly to
`vroom()`. Imagine we have a directory of files containing the flights data, where
each file corresponds to a single airline.



Then, we can efficiently read all of the files into one tibble by passing a
vector of the filenames directly to `vroom()`.


```r
files <- fs::dir_ls(glob = "flights_*tsv")
files
#> flights_9E.tsv flights_AA.tsv flights_AS.tsv flights_B6.tsv flights_DL.tsv 
#> flights_EV.tsv flights_F9.tsv flights_FL.tsv flights_HA.tsv flights_MQ.tsv 
#> flights_OO.tsv flights_UA.tsv flights_US.tsv flights_VX.tsv flights_WN.tsv 
#> flights_YV.tsv
data <- vroom(files)
#> Observations: 336,776
#> Variables: 19
#> chr  [ 4]: carrier, tailnum, origin, dest
#> dbl  [14]: year, month, day, dep_time, sched_dep_time, dep_delay, arr_time, sched_arr...
#> dttm [ 1]: time_hour
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

## Reading and writing compressed files

Just like readr, vroom automatically reads and writes zip, gzip, bz2 and xz compressed
files with the standard file extensions.


```r
vroom_write(flights, "flights.tsv.gz")

# Check file sizes to show file is compressed
fs::file_size(c("flights.tsv", "flights.tsv.gz"))
#> 29.62M  7.87M

# Read the file back in
data <- vroom("flights.tsv.gz")
#> Observations: 336,776
#> Variables: 19
#> chr  [ 4]: carrier, tailnum, origin, dest
#> dbl  [14]: year, month, day, dep_time, sched_dep_time, dep_delay, arr_time, sched_arr...
#> dttm [ 1]: time_hour
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

# Reading remote files

vroom can also read files from the internet as well by passing the URL of the file to `vroom()`.


```r
file <- "https://raw.githubusercontent.com/r-lib/vroom/master/inst/extdata/mtcars.csv"
data <- vroom(file)
#> Observations: 32
#> Variables: 12
#> chr [ 1]: model
#> dbl [11]: mpg, cyl, disp, hp, drat, wt, qsec, vs, am, gear, carb
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

It can even read gzipped files from the internet (although currently not the other compressed formats).

## Reading and writing from pipe connections

vroom provides efficient input and output from `pipe()` connections.

This is useful for doing things like pre-filtering large inputs with command line tools like [grep](https://en.wikipedia.org/wiki/Grep).


```r
# Return only flights on United Airlines
data <- vroom(pipe("grep -w UA flights.tsv"), col_names = names(flights))
#> Observations: 58,665
#> Variables: 19
#> chr  [ 4]: carrier, tailnum, origin, dest
#> dbl  [14]: year, month, day, dep_time, sched_dep_time, dep_delay, arr_time, sched_arr...
#> dttm [ 1]: time_hour
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

Or using multi-threaded compression programs like
[pigz](https://zlib.net/pigz/), which can greatly reduce the time to write compressed
files.


```r
bench::workout({
  vroom_write(flights, "flights.tsv.gz")
  vroom_write(flights, pipe("pigz > flights.tsv.gz"))
})
#> # A tibble: 2 x 3
#>   exprs                                                process     real
#>   <bch:expr>                                          <bch:tm> <bch:tm>
#> 1 vroom_write(flights, "flights.tsv.gz")                  3.5s    2.69s
#> 2 vroom_write(flights, pipe("pigz > flights.tsv.gz"))    1.54s 975.09ms
```

## Column selection

`vroom` introduces a new argument, `col_select`, which makes selecting columns to
keep (or omit) more straightforward.

 `col_select` uses the same interface as `dplyr::select()`, so you can do flexible selection operations.

- Select with the column names

```r
data <- vroom("flights.tsv", col_select = c(year, flight, tailnum))
#> Observations: 336,776
#> Variables: 3
#> chr [1]: tailnum
#> dbl [2]: year, flight
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

- Drop columns by name

```r
data <- vroom("flights.tsv", col_select = c(-dep_time, -air_time:-time_hour))
#> Observations: 336,776
#> Variables: 13
#> chr [4]: carrier, tailnum, origin, dest
#> dbl [9]: year, month, day, sched_dep_time, dep_delay, arr_time, sched_arr_time, arr...
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```

- Use the selection helpers

```r
data <- vroom("flights.tsv", col_select = ends_with("time"))
#> Observations: 336,776
#> Variables: 5
#> dbl [5]: dep_time, sched_dep_time, arr_time, sched_arr_time, air_time
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
```
- Or rename columns

```r
data <- vroom("flights.tsv", col_select = list(plane = tailnum, everything()))
#> Observations: 336,776
#> Variables: 19
#> chr  [ 4]: carrier, tailnum, origin, dest
#> dbl  [14]: year, month, day, dep_time, sched_dep_time, dep_delay, arr_time, sched_arr...
#> dttm [ 1]: time_hour
#> 
#> Call `spec()` for a copy-pastable column specification
#> Specify the column types with `col_types` to quiet this message
data
#> # A tibble: 336,776 x 19
#>    plane  year month   day dep_time sched_dep_time dep_delay arr_time
#>    <chr> <dbl> <dbl> <dbl>    <dbl>          <dbl>     <dbl>    <dbl>
#>  1 N142…  2013     1     1      517            515         2      830
#>  2 N242…  2013     1     1      533            529         4      850
#>  3 N619…  2013     1     1      542            540         2      923
#>  4 N804…  2013     1     1      544            545        -1     1004
#>  5 N668…  2013     1     1      554            600        -6      812
#>  6 N394…  2013     1     1      554            558        -4      740
#>  7 N516…  2013     1     1      555            600        -5      913
#>  8 N829…  2013     1     1      557            600        -3      709
#>  9 N593…  2013     1     1      557            600        -3      838
#> 10 N3AL…  2013     1     1      558            600        -2      753
#> # … with 336,766 more rows, and 11 more variables: sched_arr_time <dbl>,
#> #   arr_delay <dbl>, carrier <chr>, flight <dbl>, origin <chr>,
#> #   dest <chr>, air_time <dbl>, distance <dbl>, hour <dbl>, minute <dbl>,
#> #   time_hour <dttm>
```

## Name repair

Often the names of columns in the original dataset are not ideal to work with.
`vroom()` uses the same [.name_repair](https://www.tidyverse.org/articles/2019/01/tibble-2.0.1/#name-repair)
argument as tibble, so you can use one of the default name repair strategies or
provide a custom function. A great approach is to use the
[janitor](http://sfirke.github.io/janitor/) `make_clean_names()` function as the input.


```r
vroom("flights.tsv", .name_repair = janitor::make_clean_names)
#> # A tibble: 336,776 x 19
#>     year month   day dep_time sched_dep_time dep_delay arr_time
#>    <dbl> <dbl> <dbl>    <dbl>          <dbl>     <dbl>    <dbl>
#>  1  2013     1     1      517            515         2      830
#>  2  2013     1     1      533            529         4      850
#>  3  2013     1     1      542            540         2      923
#>  4  2013     1     1      544            545        -1     1004
#>  5  2013     1     1      554            600        -6      812
#>  6  2013     1     1      554            558        -4      740
#>  7  2013     1     1      555            600        -5      913
#>  8  2013     1     1      557            600        -3      709
#>  9  2013     1     1      557            600        -3      838
#> 10  2013     1     1      558            600        -2      753
#> # … with 336,766 more rows, and 12 more variables: sched_arr_time <dbl>,
#> #   arr_delay <dbl>, carrier <chr>, flight <dbl>, tailnum <chr>,
#> #   origin <chr>, dest <chr>, air_time <dbl>, distance <dbl>, hour <dbl>,
#> #   minute <dbl>, time_hour <dttm>

vroom("flights.tsv", .name_repair = ~ janitor::make_clean_names(., case = "lower_camel"))
#> # A tibble: 336,776 x 19
#>     year month   day depTime schedDepTime depDelay arrTime schedArrTime
#>    <dbl> <dbl> <dbl>   <dbl>        <dbl>    <dbl>   <dbl>        <dbl>
#>  1  2013     1     1     517          515        2     830          819
#>  2  2013     1     1     533          529        4     850          830
#>  3  2013     1     1     542          540        2     923          850
#>  4  2013     1     1     544          545       -1    1004         1022
#>  5  2013     1     1     554          600       -6     812          837
#>  6  2013     1     1     554          558       -4     740          728
#>  7  2013     1     1     555          600       -5     913          854
#>  8  2013     1     1     557          600       -3     709          723
#>  9  2013     1     1     557          600       -3     838          846
#> 10  2013     1     1     558          600       -2     753          745
#> # … with 336,766 more rows, and 11 more variables: arrDelay <dbl>,
#> #   carrier <chr>, flight <dbl>, tailnum <chr>, origin <chr>, dest <chr>,
#> #   airTime <dbl>, distance <dbl>, hour <dbl>, minute <dbl>,
#> #   timeHour <dttm>
```


## Column types

Like readr, vroom guesses the data types of columns as they are read. readr
simply used the first `n` rows of data, vroom uses an improved heuristic of
looking at data throughout the file, which should improve guessing accuracy.
However if the guessing fails it can be necessary to change the type of one or
more columns.

The available specifications are: (with single letter abbreviations in quotes)

* `col_logical()` 'l', containing only `T`, `F`, `TRUE`, `FALSE`, `1` or `0`.
* `col_integer()` 'i', integer values.
* `col_double()` 'd', floating point values.
* `col_number()` [n], numbers containing the `grouping_mark`
* `col_date(format = "")` [D]: with the locale's `date_format`.
* `col_time(format = "")` [t]: with the locale's `time_format`.
* `col_datetime(format = "")` [T]: ISO8601 date times.
* `col_factor(levels, ordered)` 'f', a fixed set of values.
* `col_character()` 'c', everything else.
* `col_skip()` '_, -', don't import this column.
* `col_guess()` '?', parse using the "best" type based on the input.

You can tell vroom what columns to use with the `col_types()` argument in a number of ways.

If you only need to override a single column, the most concise way is to use a named vector.


```r
# read the 'year' column as an integer
data <- vroom("flights.tsv", col_types = c(year = "i"))

# also skip reading the 'time_hour' column
data <- vroom("flights.tsv", col_types = c(year = "i", time_hour = "_"))

# also read the carrier as a factor
data <- vroom("flights.tsv", col_types = c(year = "i", time_hour = "_", carrier = "f"))
```

However, you can also use the `col_*()` functions in a list.


```r
data <- vroom("flights.tsv",
  col_types = list(year = col_integer(), time_hour = col_skip(), carrier = col_factor())
)
```

This is most useful when a column type needs additional information, such as
for categorical data when you know all of the levels of a factor.


```r
data <- vroom("flights.tsv",
  col_types = list(dest = col_factor(levels = c("EWR", "JFK", "LGA")))
)
```

## Speed

vroom is fast, but how fast?
We benchmarked vroom using a real-world dataset of taxi-trip data, with
14.7 million rows, 11 columns. It contains a mix of numeric and text data, and has a
total file size of 1.55 GB.

    #> Observations: 14,776,615
    #> Variables: 11
    #> $ medallion       <chr> "89D227B655E5C82AECF13C3F540D4CF4", "0BD7C8F5B...
    #> $ hack_license    <chr> "BA96DE419E711691B9445D6A6307C170", "9FD8F69F0...
    #> $ vendor_id       <chr> "CMT", "CMT", "CMT", "CMT", "CMT", "CMT", "CMT...
    #> $ pickup_datetime <chr> "2013-01-01 15:11:48", "2013-01-06 00:18:35", ...
    #> $ payment_type    <chr> "CSH", "CSH", "CSH", "CSH", "CSH", "CSH", "CSH...
    #> $ fare_amount     <dbl> 6.5, 6.0, 5.5, 5.0, 9.5, 9.5, 6.0, 34.0, 5.5, ...
    #> $ surcharge       <dbl> 0.0, 0.5, 1.0, 0.5, 0.5, 0.0, 0.0, 0.0, 1.0, 0...
    #> $ mta_tax         <dbl> 0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0.5, 0...
    #> $ tip_amount      <int> 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0...
    #> $ tolls_amount    <dbl> 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 0.0, 4.8, 0.0, 0...
    #> $ total_amount    <dbl> 7.0, 7.0, 7.0, 6.0, 10.5, 10.0, 6.5, 39.3, 7.0...

We performed a series of simple manipulations with each approach.

  - Reading the data
  - `print()`
  - `head()`
  - `tail()`
  - Sampling 100 random rows
  - Filtering for "UNK" payment, this is 6434 rows (0.0435% of total).
  - Summarizing the mean fare amount per payment type.

<style>
td,th {
  padding: 0.4em;
}
thead {
border-top: 1px solid #aaa;
border-bottom: 1px solid #aaa;
}
table {
margin-left: auto;
margin-right: auto;
border-bottom: 1px solid #aaa;
}
</style>

<img src="/articles/2019-05-vroom-1-0-0_files/figure-html/benchmark_plot-1.png" width="960" />

|    package|     read| print| head| tail| sample| filter| summarise|    total|
|----------:|--------:|-----:|----:|----:|------:|------:|---------:|--------:|
| read.delim| 1m 21.5s|   6ms|  1ms|  1ms|    1ms|  315ms|     764ms| 1m 22.6s|
|      readr|    33.1s|  90ms|  1ms|  1ms|    2ms|  202ms|     825ms|    34.2s|
| data.table|    15.7s|  13ms|  1ms|  1ms|    1ms|  129ms|     394ms|    16.3s|
|      vroom|     3.6s|  86ms|  1ms|  1ms|    2ms|   1.4s|      1.9s|       7s|



<br/>

Some things to note in the results. The initial reading is much faster in vroom
than any other method, and most of the manipulations, such as `print()`,
`head()`, `tail()` and `sample()` are equally fast, so fast they can't be seen
in the plots. However because the character data is read lazily, operations such
as `filter` and `summarise`, which need character values, require additional
time. However, this cost will only occur once. After the values have been read,
they will be stored in memory, and subsequent accesses will be equivalent to
other packages.

For more details on how the benchmarks were performed and additional benchmarks
with other types of data see the [benchmark
vignette](http://vroom.r-lib.org/articles/benchmarks.html).

## Feedback welcome!

vroom is a new package and, like any newborn, may fall down a few times before
learning to run. If you do run into a bug or think of a new feature that
would work well in vroom please [open an
issue](https://github.com/r-lib/vroom/issues) so we can discuss it!

## Acknowledgements

Even though this is a new release, a number of people have been testing out
pre-release versions on their datasets and opening issues, which has been a
huge help in making the package more robust.

A big thanks to [&#x0040;alex-gable](https://github.com/alex-gable),
[&#x0040;andrie](https://github.com/andrie),
[&#x0040;dan-reznik](https://github.com/dan-reznik),
[&#x0040;Evgeniy-](https://github.com/Evgeniy-),
[&#x0040;ginolhac](https://github.com/ginolhac),
[&#x0040;ibarraespinosa](https://github.com/ibarraespinosa),
[&#x0040;KasperSkytte](https://github.com/KasperSkytte),
[&#x0040;ldecicco-USGS](https://github.com/ldecicco-USGS),
[&#x0040;LuisQ95](https://github.com/LuisQ95),
[&#x0040;matthieu-haudiquet](https://github.com/matthieu-haudiquet),
[&#x0040;md0u80c9](https://github.com/md0u80c9),
[&#x0040;mkiang](https://github.com/mkiang),
[&#x0040;R3myG](https://github.com/R3myG),
[&#x0040;randomgambit](https://github.com/randomgambit),
[&#x0040;slowkow](https://github.com/slowkow),
[&#x0040;telaroz](https://github.com/telaroz),
[&#x0040;thierrygosselin](https://github.com/thierrygosselin), and
[&#x0040;xiaodaigh](https://github.com/xiaodaigh)!

Also this package would not be possible without the following significant
contributions to the R ecosystem.

- [Gabe Becker](https://twitter.com/groundwalkergmb), [Luke
  Tierney](https://stat.uiowa.edu/~luke/) and [Tomas
  Kalibera](https://github.com/kalibera) for conceiving, implementing
  and maintaining the [Altrep
  framework](https://svn.r-project.org/R/branches/ALTREP/ALTREP.html) used extensively in vroom.
- [Romain François](https://twitter.com/romain_francois), whose
  [Altrepisode](https://purrple.cat/blog/2018/10/14/altrep-and-cpp/)
  package and [related
  blog-posts](https://purrple.cat/blog/2018/10/14/altrep-and-cpp/)
  were a great guide for creating new Altrep objects in C++.
- [Matt Dowle](https://twitter.com/mattdowle) and the rest of the
  [Rdatatable](https://github.com/Rdatatable) team,
  `data.table::fread()` is blazing fast and a great motivator to think about
  how to read delimited files fast\!
