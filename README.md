Timezones: Vega, Vega-Lite, Python and R
================

Until I can wrap my head around all the concepts, this is going to be me
“puking up” all of my knowledge and opinions on timezones. It is hoped
that this can, with enough consideration and revision, serve as a
foundation for efforts to improve the time-handling capabilities of Vega
and Vega-Lite.

We want a way to render a spec independent of the browser-locale.

``` r
library("magrittr")
library("readr")
library("glue")
```

## Time

The first thing I want to sort out is how time is handled in R and
Python. I think this is a useful prelimanary exercise because chances
are that a Vega/Vega-Lite spec is giong to be built using R or Python.

I think it can also be useful so we can understand, a bit better, each
other languages.

### R

In R, we have a type `POSIXct`, which is timezone-aware. Internally, it
stores the number of seconds since the UNIX epoch. There is an
attribute, `tzone`, that may or may not be set (in my opinion, it should
be set).

First, I’m going to make a helper function to look a little at the
internals of a given `POSIXct`.

``` r
describe <- function(x) {
  
  as_utc <- x
  attr(as_utc, "tzone") <- "UTC"
  
  print(
    glue::glue("UTC time:       {as_utc}"),
    glue::glue("Formatted time: {x}"),
    glue::glue("Timezone:       {attr(x, 'tzone')}"),
    glue::glue("Numeric value:  {as.numeric(x)}")
  )
  
  invisible(x)
}
```

-----

Let’s start with parsing a string into a POSIXct, and not specifying a
timezone:

``` r
"2013-04-05 06:07:08" %>%
  as.POSIXct() %>%
  describe()
```

    ## UTC time:       2013-04-05 11:07:08
    ## Formatted time: 2013-04-05 06:07:08
    ## Timezone:       
    ## Numeric value:  1365160028

It looks like the system parses this as a local time (I am running this
from the `"America/Chicago"` timezone), so it is stored as the UTC time
that corresponds to local time described by the string.

-----

Next, let’s specify the timezone to the constructor:

``` r
"2013-04-05 06:07:08" %>%
  as.POSIXct(tz = "UTC") %>%
  describe()
```

    ## UTC time:       2013-04-05 06:07:08
    ## Formatted time: 2013-04-05 06:07:08
    ## Timezone:       UTC
    ## Numeric value:  1365142028

This makes sense, and is repeatable, not depending on my system
settings.

-----

Let’s give the constructor an ISO-8601 string:

``` r
"2013-04-05T06:07:08Z" %>%
  as.POSIXct() %>%
  describe()
```

    ## UTC time:       2013-04-05 05:00:00
    ## Formatted time: 2013-04-05
    ## Timezone:       
    ## Numeric value:  1365138000

This is not good. Not good at all. It is using the system time and has
choked on the `"T"` in the string.

-----

Luckily we have an alternative, the `readr::parse_datetime()` function.

``` r
"2013-04-05T06:07:08Z" %>%
  readr::parse_datetime() %>%
  describe()
```

    ## UTC time:       2013-04-05 06:07:08
    ## Formatted time: 2013-04-05 06:07:08
    ## Timezone:       UTC
    ## Numeric value:  1365142028

Here, we recognize the ISO-8601 string, and it sets the timezone to
`"UTC"`. Using Pandas’ vocabulary, we have localized to `"UTC"`.

-----

When parsing, we can set a timezone:

``` r
"2013-04-05 06:07:08" %>%
  readr::parse_datetime(locale = readr::locale(tz = "America/Chicago")) %>%
  describe()
```

    ## UTC time:       2013-04-05 11:07:08
    ## Formatted time: 2013-04-05 06:07:08
    ## Timezone:       America/Chicago
    ## Numeric value:  1365160028

If the format is ambiguous (no `"Z"` or `"+00:00"` at the end), the
parser localizes to the timezone provided.

-----

What if the string is already implies the localization to be UTC?

``` r
"2013-04-05T06:07:08Z" %>%
  readr::parse_datetime(locale = readr::locale(tz = "America/Chicago")) %>%
  describe()
```

    ## UTC time:       2013-04-05 06:07:08
    ## Formatted time: 2013-04-05 01:07:08
    ## Timezone:       America/Chicago
    ## Numeric value:  1365142028

In this case, the parser localized to `"UTC"`, then (again using Pandas’
vocabulary) converts to `"America/Chicago"`.

-----

When writing a spec directly from R, it becomes the R users’
responsibility to make sure that POSIXct is serialized to an ISO format,
and to serialize the timezone somehow (whenever that may become
meaningful).

### Python

Here’s where I need to educate myself a little bit, so I’ll follow
Jake’s lead:

<https://github.com/vega/vega-lite/issues/4044#issuecomment-408565320>

> One thing that would be worth thinking about is whether we can support
> `datetime64[ns,utc]` as well as `datetime64[ns]` in a streamlined way.
> That way people could choose UTC or not within Python, and everything
> else should just flow from there.

I’ll start with a Python version of my `describe()` function.

``` python
# Python
import pandas as pd
def describe(x):
    if x.tzinfo is None:
      utc_time = 'not defined'
    else:
      utc_time = x.tz_convert('UTC')
    
    print('UTC time:       ', utc_time)
    print('Formatted time: ', x)
    print('Timezone:       ', x.tzinfo)
    print('Numeric value:  ', x.value)
    print('')
    print(pd.DataFrame({'x' : [x]}).dtypes)
    return x
```

``` python
x = pd.to_datetime('2013-04-05 06:07:08')
describe(x)
```

    ## UTC time:        not defined
    ## Formatted time:  2013-04-05 06:07:08
    ## Timezone:        None
    ## Numeric value:   1365142028000000000
    ## 
    ## x    datetime64[ns]
    ## dtype: object

``` python
x = pd.to_datetime('2013-04-05T06:07:08Z').tz_localize('UTC').tz_convert('America/Chicago')
describe(x)
```

    ## UTC time:        2013-04-05 06:07:08+00:00
    ## Formatted time:  2013-04-05 01:07:08-05:00
    ## Timezone:        America/Chicago
    ## Numeric value:   1365142028000000000
    ## 
    ## x    datetime64[ns, America/Chicago]
    ## dtype: object

``` python
x = pd.to_datetime('2013-04-05T06:07:08Z')
describe(x)
```

    ## UTC time:        not defined
    ## Formatted time:  2013-04-05 06:07:08
    ## Timezone:        None
    ## Numeric value:   1365142028000000000
    ## 
    ## x    datetime64[ns]
    ## dtype: object

### reticulate

``` r
library("reticulate")

pd <- import("pandas")
```

Let’s try sending data to Python, testing if the timezone gets sent
over:

``` r
df <- 
  data.frame(
    x = readr::parse_datetime(
      "2013-04-05T06:07:08Z", 
      locale = readr::locale(tz = "America/Chicago")
    )
  )

describe(df$x)
```

    ## UTC time:       2013-04-05 06:07:08
    ## Formatted time: 2013-04-05 01:07:08
    ## Timezone:       America/Chicago
    ## Numeric value:  1365142028

``` r
pd_df <- r_to_py(df)

cat("\n")
```

``` r
pd_df
```

    ##                     x
    ## 0 2013-04-05 06:07:08

``` r
pd_df$dtypes
```

    ## x    datetime64[ns]
    ## dtype: object

Let’s set the timezone there and bring it back

``` r
pd_df_new <- pd_df$copy()
pd_df_new$x <- pd_df_new$x$dt$tz_localize("UTC")
pd_df_new$x
```

    ## 0   2013-04-05 06:07:08+00:00
    ## Name: x, dtype: datetime64[ns, UTC]

``` r
pd_df_new$x <- pd_df_new$x$dt$tz_convert("America/New_York")
pd_df_new$x
```

    ## 0   2013-04-05 02:07:08-04:00
    ## Name: x, dtype: datetime64[ns, America/New_York]

``` r
df_new <- py_to_r(pd_df_new)
describe(df_new$x)
```

    ## UTC time:       2013-04-05 06:07:08
    ## Formatted time: 2013-04-05 06:07:08
    ## Timezone:       UTC
    ## Numeric value:  1365142028

## Intake

Jake VanderPlas:

> I wouldn’t want to make Pandas output UTC because then, for example,
> if I make a chart in Seattle and send it to a friend in NYC the
> rendering will be different.

For me, the takeaway is that we want a consistent rendering regardless
of where a chart is rendered, and to support and document how to do just
that.

Would like more detail on how the [data
format](https://vega.github.io/vega-lite/docs/data.html#format) works in
Vega/Vega-Lite. For example, the documention implies that JavaScript’s
`Date.parse()` is used in
[Vega-Lite](https://vega.github.io/vega-lite/docs/data.html#format), but
not in [Vega](https://vega.github.io/vega/docs/data/#format).

Look to <https://github.com/altair-viz/altair/pull/1053> for how Jake
uses timezones.

Also, look to
<https://github.com/altair-viz/altair/pull/1053#issuecomment-408610839>
for the problem of browser location. (Earlier Jake comment)

It can be useful to have a slightly different interpretation of date
vs. datetime.

The timezone should be an attribute of the data itself. It should be
available at the top level: see
<https://github.com/vega/vega-lite/issues/4004>

Vega can parse to `"boolean"`, `"date"`, `"number"` or `"string"`. So,
in Vega, is a datetime just a number that represents the number of
milliseconds from the UNIX epoch? If we are to associate a timezone with
a datetime in the parsing, it would seem to require its own type. Maybe
this one of reasons behind [this
comment](https://github.com/vega/vega-lite/issues/4044#issuecomment-406023278).

In efforts to decouple the data from the visualization specification, it
becomes our responsibility to provide both the serialized dataframe
(csv, json, etc.) and the parsing specification (timezones).