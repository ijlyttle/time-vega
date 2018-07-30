Timezones: Vega, Vega-Lite, Python and R
================

Until I can wrap my head around all the concepts, this is going to be me
“puking up” all of my knowledge and opinions on timezones. It is hoped
that this can, with enough consideration and revision, serve as a
foundation for efforts to improve the time-handling capabilities of Vega
and Vega-Lite.

In essence, I want to find a way to support timezone-aware objects in
Vega/Vega-Lite, as a user, I would like to build a spec using
timezone-aware (tz-aware) objects so that the rendering of the spec can
be independent of the timezone of the browser-locale.

## Motivation

Consider this quote from Jake Vanderplas:

> I wouldn’t want to make Pandas output UTC because then, for example,
> if I make a chart in Seattle and send it to a friend in NYC the
> rendering will be different.

I understand Jake’s motivation here and appreciate his solution. If I
can borrow Jake’s example, I would like that he be able to build that
chart using tz-aware objects (perhaps using the `"America/Los_Angeles"`
timezone), and that the chart would render the scales and aggregations
using the `"America/Los_Angeles"` timezone regardless of the system
settings of the browser in which it is rendered. In essence, another
solution for the Seattle/New York problem.

“But Jake *has* a solution already,” you might reasonably respond.
Having gone through this exercise to (try to) understand tz-naive and
tz-aware objects, I see that importing timestamps as local-time and
treating them as tz-naive solves 99% of the problem; I have come to
appreciate that.

However, there is a problem, twice a year with daylight-saving time. In
my experience, every so often I will get strange behaviors when working
with time-based data. If I get funny things happening in March and
November, I will retrace my steps and find that I have been bitten by
the local-timestamp bug.

I hate to sound absolutist about this, but the only way I have found to
avoid these bugs is to, as soon as possible after receiving data,
(figure out the right way to) cast it as UTC and store the Olson (IANA)
timezone. Whenever I serialize and unserialize the data, I use the
ISO-8601 format and find a way to “keep” the timezone handy.

Talk about industrialization, deploying in the wild, and decoupling the
data from the spec.

Accordingly I would like to see if it is possible to introduce a
tz-aware workflow into Vega/Vega-Lite, while keeping the existing
workflows intact (Jake’s existing chart would still work).

## Time

The first thing I want to sort out is how time is handled in R, Python,
and JavaScript. I think this is a useful prelimanary exercise because
chances are that a Vega/Vega-Lite spec is giong to be built using R or
Python.

I think it can also be useful so we can understand, a bit better, each
others’ languages. To make comparisons, I have written a function in
each language, called `describe()`, to help show the internal
representation of each language’s time-aware object.

### R

In R, we have a type `POSIXct`, which is timezone-aware. Internally, it
stores the number of seconds since the UNIX epoch. There is an
attribute, `tzone`, that may or may not be set (in my workflow, I always
set it).

``` r
library("magrittr")
library("readr")
library("glue")
```

First, I’m going to make a helper function to look a little at the
internals of a given `POSIXct`. This function (and the `describe()`
functions I create for the other languages) reports the milliseconds in
deference to JavaScript.

``` r
describe <- function(x) {
  
  tz <- attr(x, "tzone")
  
  if (identical(tz, "")) {
    as_utc <- "not defined"
  } else {
    as_utc <- x
    attr(as_utc, "tzone") <- "UTC"    
  }

  str_value <- formatC(as.numeric(x) * 1e3, digits = 13, width = 13)
  
  print(
    glue::glue("UTC time:       {as_utc}"),
    glue::glue("Formatted time: {x}"),
    glue::glue("Timezone:       {tz}"),
    glue::glue("Milliseconds:   {str_value}")
  )
  
  invisible(x)
}
```

-----

Let’s start with parsing a string into a POSIXct, and not specifying a
timezone - the naive case:

``` r
"2013-04-05 06:00:00" %>%
  as.POSIXct() %>%
  describe()
```

    ## UTC time:       not defined
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       
    ## Milliseconds:   1365159600000

It looks like the system parses this as a local time (I am running this
from the `"America/Chicago"` timezone); its numerical value being UTC
time that corresponds to local time described by the string. Note that R
does not attach a timezone attribute to the value.

-----

Next, let’s specify the timezone to the constructor:

``` r
"2013-04-05 06:00:00" %>%
  as.POSIXct(tz = "UTC") %>%
  describe()
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       UTC
    ## Milliseconds:   1365141600000

This makes sense, and is repeatable; it does not depend on my system
settings.

-----

Let’s give the constructor an ISO-8601 string:

``` r
"2013-04-05T06:00:00Z" %>%
  as.POSIXct() %>%
  describe()
```

    ## UTC time:       not defined
    ## Formatted time: 2013-04-05
    ## Timezone:       
    ## Milliseconds:   1365138000000

This is not good. Not good at all. It is using the system time and has
choked on the `"T"` in the string. We can use the `format` argument in
this case, but we would like to to use a tool that recognizes ISO
strings natively.

-----

Luckily we have an alternative, the `readr::parse_datetime()` function.

``` r
"2013-04-05T06:00:00Z" %>%
  readr::parse_datetime() %>%
  describe()
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       UTC
    ## Milliseconds:   1365141600000

Here, we recognize the ISO-8601 string, and it sets the timezone to
`"UTC"`. Using Pandas’ vocabulary, we have localized to `"UTC"`.

-----

Let’s back up to see what happens in the naive case:

``` r
"2013-04-05 06:00:00" %>%
  readr::parse_datetime() %>%
  describe()
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       UTC
    ## Milliseconds:   1365141600000

Here, Hadley assumes that the timezone is UTC and sets it so as to
remove any downstream ambiguity (this does not seem a surprise).

-----

When parsing, we can set a timezone:

``` r
"2013-04-05 06:00:00" %>%
  readr::parse_datetime(locale = readr::locale(tz = "America/Chicago")) %>%
  describe()
```

    ## UTC time:       2013-04-05 11:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365159600000

If the format is ambiguous (no `"Z"` or `"+00:00"` at the end), the
parser localizes to the timezone provided.

-----

What if the string is already implies the localization to be UTC?

``` r
"2013-04-05T06:00:00Z" %>%
  readr::parse_datetime(locale = readr::locale(tz = "America/Chicago")) %>%
  describe()
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 01:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

In this case, the parser localized to `"UTC"`, then (again using Pandas’
vocabulary) converted to `"America/Chicago"`.

-----

I can manage the conversion myself by setting the `tzone` attribute:

``` r
timestamp <- readr::parse_datetime("2013-04-05T06:00:00Z") 
attr(timestamp, "tzone") <- "America/Chicago"

describe(timestamp)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 01:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

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

``` python
# Python
import pandas as pd
```

I’ll start with a Python version of my `describe()` function. I’m also
adding some information about how Pandas views the timestamp, given
Jake’s distinction between `datetime[ns]` and `datetime[ns, utc]`.

``` python
# Python
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

First, let’s try a naive format:

``` python
# Python
x = pd.to_datetime('2013-04-05 06:00:00')
describe(x)
```

    ## UTC time:        not defined
    ## Formatted time:  2013-04-05 06:00:00
    ## Timezone:        None
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns]
    ## dtype: object

Because we did not fix a timezone, it seems we are using a “local”
timezone where this is the number of seconds from an epoch defined in
that timezone - similar to what happens in R for the naive case.

-----

Let’s see what happens if we localize this to `'America/Chicago'`:

``` python
# Python
x = pd.to_datetime('2013-04-05 06:00:00').tz_localize('America/Chicago')
describe(x)
```

    ## UTC time:        2013-04-05 11:00:00+00:00
    ## Formatted time:  2013-04-05 06:00:00-05:00
    ## Timezone:        America/Chicago
    ## Milliseconds:    1365159600000.0
    ## 
    ## x    datetime64[ns, America/Chicago]
    ## dtype: object

This looks equivlaent to `readr::parse_datetime()`, localizing with
`"America/Chicago"`.

-----

Let’s see what happens if we provide in ISO string:

``` python
# Python
x = pd.to_datetime('2013-04-05T06:00:00Z')
describe(x)
```

    ## UTC time:        not defined
    ## Formatted time:  2013-04-05 06:00:00
    ## Timezone:        None
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns]
    ## dtype: object

Here, the object is not localized to UTC. As an aside, this seems like
an opportunity - but how strings are parsed into Python is outside of
this scope.

-----

We can get the “right” things to happen by localizing manually:

``` python
# Python
x = pd.to_datetime('2013-04-05T06:00:00Z').tz_localize('UTC')
describe(x)
```

    ## UTC time:        2013-04-05 06:00:00+00:00
    ## Formatted time:  2013-04-05 06:00:00+00:00
    ## Timezone:        UTC
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns, UTC]
    ## dtype: object

-----

If we want to consider this in the context of `'America/Chicago'`, we
can convert:

``` python
# Python
x = pd.to_datetime('2013-04-05T06:00:00Z').tz_localize('UTC').tz_convert('America/Chicago')
describe(x)
```

    ## UTC time:        2013-04-05 06:00:00+00:00
    ## Formatted time:  2013-04-05 01:00:00-05:00
    ## Timezone:        America/Chicago
    ## Milliseconds:    1365141600000.0
    ## 
    ## x    datetime64[ns, America/Chicago]
    ## dtype: object

-----

I think I have a reasonably good idea of how timestamps can be parsed,
localized, and converted in R and in Python, in isolation.

Given how the **altair** R package used **reticulate** to manage the
interface to the **Altair** Python package, it may be useful to see what
reticulate does and does not do, and to see if there is anything it
might do better.

### reticulate

``` r
library("reticulate")

pd <- import("pandas")
```

This is a bit of a detour - it is not essential to the Vega story;
rather it is interesting only to the R “altair” story.

Let’s create a data frame, using a localized time-stamp:

``` r
df_r_to_py <- 
  data.frame(
    x = readr::parse_datetime(
      "2013-04-05T06:00:00Z", 
      locale = readr::locale(tz = "America/Chicago")
    )
  )

describe(df_r_to_py$x)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 01:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

So far, no surprises.

-----

Let’s see what this looks like, having been converted to a Pandas Data
Frame:

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
the localization has not been brought over. I have filed an
[issue](https://github.com/rstudio/reticulate/issues/325) with
**reticulate** to see if this is the intended behavior (or if it could
be).

-----

Let’s copy the Pandas Data Frame, and provide the copy with a timezone:

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

-----

Let’s look at this new copy in R, to see if the newly-added timezone
survives the trip:

``` r
df_return <- py$df_py_to_r

describe(df_return$x)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 06:00:00
    ## Timezone:       UTC
    ## Milliseconds:   1365141600000

Here, there seems to be partially good-news. The numerical value has
been preserved, and the timzone is set.

The bad news is that the timezone is set to `"UTC"`.

``` r
attr(df_return$x, "tzone") <- "America/Chicago"

describe(df_return$x)
```

    ## UTC time:       2013-04-05 06:00:00
    ## Formatted time: 2013-04-05 01:00:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

Perhaps, if the Data Frame contains a column with a localized timestamp,
**reticulate** could set the timezone attribute of the `POSIXct` column
accordingly.

### JavaScript

In this section, I will repeat the same exercise in JavaScript, focusing
on how `Date.parse()` works and looking at how the **moment.js** library
(plus friends) works.

There are a couple of points that are not quite clear in my head:

  - How [data
    format](https://vega.github.io/vega-lite/docs/data.html#format)
    works in Vega-Lite vs. Vega. For example, the documention implies
    that JavaScript’s `Date.parse()` is used in
    [Vega-Lite](https://vega.github.io/vega-lite/docs/data.html#format),
    but not in [Vega](https://vega.github.io/vega/docs/data/#format). Is
    this true or is this an omission the Vega documentation?

  - The JavaScript class is called `Date` - however seems to function as
    a “datetime”. Am I thinking about this correctly?

We start our investigation by creating a V8 session:

``` r
library("V8")
ct <- v8()
```

-----

Let’s start by parsing a naive string:

``` r
ct$eval("var i = Date.parse('2013-04-05 06:00:00').toString();")

ct$get("i")
```

    ## [1] "1365159600000"

We are working only with the number of milliseconds - here we localize
using the system.

`Date.parse()` seems to have recognized this as a non-ISO-formatted
string and parsed this to the number of milliseconds since the UNIX
epoch for the wall-clock time in my system’s timezone. This is
consistent with the naive case in R and Python.

-----

Next, let’s look at an ISO-formatted string:

``` r
ct$eval("var i = Date.parse('2013-04-05T06:00:00Z').toString();")

ct$get("i")
```

    ## [1] "1365141600000"

Here we localize to UTC.

`Date.parse()` seems to have recognized the ISO-formatted string and
parsed this to the number of milliseconds since the UNIX epoch. This in
consistent with the `readr::parse_datetime()`

-----

-----

Within the Vega context, perhaps all that `Date.parse()` does is to
parse the string into a “number of milliseconds”, then it becomes Vega’s
responsibility to interpret that number, either as a local time, UTC
time, or potentially as a *localized* time.

We should also note that `Date.Parse()` returns a number only - there is
no localization information.

#### moment.js

As a crazy idea, let’s work a little with **moment.js** and imagine how
it might be incorporated into the Vega framework.

``` r
# loading the moment libraries
ct$source("https://cdnjs.cloudflare.com/ajax/libs/moment.js/2.22.2/moment.min.js")
```

    ## [1] "true"

``` r
ct$source("https://cdnjs.cloudflare.com/ajax/libs/moment-timezone/0.5.21/moment-timezone-with-data.min.js")
```

    ## [1] "true"

Let’s design a describe function using momentjs.

``` r
ct$assign("describe", JS("
  function(x) {
    console.log('UTC time:       ', x.clone().tz('UTC').format());
    console.log('Formatted time: ', x.format());
    console.log('Timezone:       ', x.tz());
    console.log('Milliseconds:   ', x.valueOf());
  }
"))
```

``` r
ct$assign("a", JS("moment('2013-04-05 06:00:00')"))
ct$eval("describe(a)")
```

    ## UTC time:       2013-04-05T11:00:00Z
    ## Formatted time: 2013-04-05T06:00:00-05:00
    ## Timezone:       undefined
    ## Milliseconds:   1365159600000

Moment gives the same number of milliseconds as using `Date.parse()`.

-----

``` r
ct$assign("a", JS("moment('2013-04-05T06:00:00Z')"))
ct$eval("describe(a)")
```

    ## UTC time:       2013-04-05T06:00:00Z
    ## Formatted time: 2013-04-05T01:00:00-05:00
    ## Timezone:       undefined
    ## Milliseconds:   1365141600000

Moment gives the same number of milliseconds as using
`Date.parse()`.

-----

``` r
ct$assign("a", JS("moment.tz('2013-04-05 06:00:00', 'America/Chicago')"))
ct$eval("describe(a)")
```

    ## UTC time:       2013-04-05T11:00:00Z
    ## Formatted time: 2013-04-05T06:00:00-05:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365159600000

-----

``` r
ct$assign("a", JS("moment.tz('2013-04-05T06:00:00Z', 'America/Chicago')"))
ct$eval("describe(a)")
```

    ## UTC time:       2013-04-05T06:00:00Z
    ## Formatted time: 2013-04-05T01:00:00-05:00
    ## Timezone:       America/Chicago
    ## Milliseconds:   1365141600000

-----

## Intake

Jake VanderPlas:

> I wouldn’t want to make Pandas output UTC because then, for example,
> if I make a chart in Seattle and send it to a friend in NYC the
> rendering will be different.

For me, the takeaway is that we want a consistent rendering regardless
of where a chart is rendered, and to support and document how to do just
that.

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
Is a Vega `"date"` what I think of as a datetime?

In efforts to decouple the data from the visualization specification, it
becomes our responsibility to provide both the serialized dataframe
(csv, json, etc.) and the parsing specification (timezones).
