
Could `r_to_py()` and `py_to_r()` be adapted such that is the source
data has columns with timezone attributes, that the timezone is set in
the target data?

Could this be done without stepping on \#160? (tagging @DavisVaughn) On
this question, I wonder if a distinction can be made between tz-naive
`datetime64[ns]` and tz-aware `datetime64[ns, <tz>]`.

-----

My experience with Python is limited, so maybe the best thing I can do
is to demonstrate. In my Rmd, I am using both R and Python chunks, so I
will label those explicitly to appear in the Markdown.

``` r
# R
library("reticulate")
library("readr")
# imports "glue"

pd <- import("pandas")

# function to describe the attributes of a POSIXct:
describe <- function(x) {
  
  as_utc <- x
  attr(as_utc, "tzone") <- "UTC"
  
  str_value <- formatC(as.numeric(x) * 1000, digits = 13, width = 13)
  
  print(
    glue::glue("UTC time:       {as_utc}"),
    glue::glue("Formatted time: {x}"),
    glue::glue("Timezone:       {attr(x, 'tzone')}"),
    glue::glue("Milliseconds:   {str_value}")
  )
  
  invisible(x)
}
```

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
    print('Milliseconds:   ', x.value / 1.e6)
    print('')
    print(pd.DataFrame({'x' : [x]}).dtypes)
    return x
```

-----

For the first step, let’s create a data frame in R that we will send to
Python. It will have a single column with a single value.

``` r
# R
df_r_to_py <- 
  data.frame(
    x = parse_datetime(
      "2013-04-05T06:00:00Z", 
      locale = locale(tz = "America/Chicago")
    )
  )

describe(df_r_to_py$x)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 01:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

So far, so good.

-----

Let’s see what Python sees:

``` python
# Python
describe(r.df_r_to_py.x[0])
```

    ## UTC time:        not defined
    ## Formatted time:  2013-04-05 06:00:00
    ## Timezone:        None
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns]
    ## dtype: object

We see that the numerical representation has been “preserved”, but that
the localization has not been brought over.

-----

In Python, let’s create a copy of what was sent from R, then let’s do
the localization manually.

``` python
# Python
df_py_to_r = r.df_r_to_py.copy()
df_py_to_r.x = df_py_to_r.x.dt.tz_localize('UTC').dt.tz_convert('America/Chicago')
describe(df_py_to_r.x[0])
```

    ## UTC time:        2013-04-05 06:00:00+00:00
    ## Formatted time:  2013-04-05 01:00:00-05:00
    ## Timezone:        America/Chicago
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns, America/Chicago]
    ## dtype: object

This looks good now.

-----

Finally, let’s look at this in R:

``` r
# R
df_return <- py$df_py_to_r

describe(df_return$x)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       UTC
    ## Milliseconds:   1365141600000

Here, there seems to be partially good-news. The numerical value has
been preserved, and the timzone is set.

The “bad” news is that the timezone is set to `"UTC"`, but this is one
of the behaviors sought in \#160 (but perhaps only for the tz-naive
case).

## Summary

Just to restate from the top:

Could `r_to_py()` and `py_to_r()` be adapted such that is the source
data has columns with timezone attributes, that the timezone is set in
the target data?

Could this be done without stepping on \#160? On this question, I wonder
if a distinction can be made between tz-naive `datetime64[ns]` and
tz-aware `datetime64[ns, <tz>]`.
