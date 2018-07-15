Inst files
================

This document is run **only** if you want to reassemble the vegawidget
htmlwidget. This would be done to update this package in whenever the
**Vega-Lite** version is upgraded.

In concrete terms, this document will construct the `inst/htmlwidgets`
directory; the **only** way to put anything in the `inst/htmlwidgets`
directory is to run this file.

We have two “sources of truth”: this file and the contents of the
directory `data-raw/templates`.

As long as the vega-embed framework remains the same, all we should need
to do to update the **Vega-Lite** libraries is to change the parameter
`vega_lite_version` in the yaml-header to this file.

If you want to update the workings of the htmlwidget, you will have to
do so in the files `data-raw/templates` directory. It may be useful to
note this in a “contributing” document for this package.

## Configure

These packages are not listed in the `Suggests` section of the
`DESCRIPTION` file. It’s on you to make sure they are all up-to-date.

``` r
library("fs")
library("glue")
library("httr")
library("here")
```

    ## here() starts at /Users/ijlyttle/Documents/git/github/vegawidget/vegawidget

``` r
library("purrr")
library("readr")
library("dplyr")
```

    ## 
    ## Attaching package: 'dplyr'

    ## The following object is masked from 'package:glue':
    ## 
    ##     collapse

    ## The following objects are masked from 'package:stats':
    ## 
    ##     filter, lag

    ## The following objects are masked from 'package:base':
    ## 
    ##     intersect, setdiff, setequal, union

``` r
library("tibble")
library("stringr")
library("conflicted")
library("vegawidget")
```

For this document, there is a source-directory, `dir_templates`, and a
target-directory, `dir_htmlwidgets`. We will read information from the
source-directory and write it to the target-directory.

``` r
dir_templates <- here("data-raw", "templates")
dir_htmlwidgets <- here("inst", "htmlwidgets")
dir_block <- here("inst", "block")
```

Finally, we need to know which versions of the libraries (vega,
vega-lite, and vega-embed) to download. We do this by inspecting the
manifest of a specific version of the vega-lite library. This package
has an internal function, `vega_version()` to help us do this:

``` r
vega_versions_long <- get_vega_versions(params$vega_lite_version)

vega_versions_long
```

    ## $vega_lite
    ## [1] "2.5.0"
    ## 
    ## $vega
    ## [1] "4.0.0-rc.2"
    ## 
    ## $vega_embed
    ## [1] "3.14.0"

``` r
# we want to remove the "-rc.2" from the end of "4.0.0-rc.2"
# "-\\w.*$"   hyphen, followed by a letter, followed by anything, then end 
vega_versions_short <- map(vega_versions_long, ~sub("-\\w.*$", "", .x))
```

## Clean and create

Our first task is to create a clean directory `inst/htmlwidgets`. If it
exists, we delete it and create it anew.

``` r
if (dir_exists(dir_htmlwidgets)) {
  dir_delete(dir_htmlwidgets)
}

dir_create(dir_htmlwidgets)
```

## Vegawidget files

Here, we have copy some files from our templates directory.

``` r
file_copy(
  path(dir_templates, "vegawidget.js"), 
  path(dir_htmlwidgets, "vegawidget.js")
)
```

The file `vegawidget.yaml` requires the versions the JavaScript
libraries; we interpolate these from `vega_versions_short`.

``` r
path(dir_templates, "vegawidget.yaml") %>%
  read_lines() %>%
  map_chr(~glue_data(vega_versions_short, .x)) %>%
  write_lines(path(dir_htmlwidgets, "vegawidget.yaml"))
```

## Lib directory

Here’s where we download the libraries themselves, along with the
licences; the versions are interpolated from `vega_versions_long`.

``` r
downloads <-
  tribble(
    ~path_local,                         ~path_remote,
    "vega-lite/vega-lite-min.js",        "https://cdn.jsdelivr.net/npm/vega-lite@{vega_lite}",
    "vega-lite/LICENSE",                 "https://raw.githubusercontent.com/vega/vega-lite/master/LICENSE",
    "vega/promise.min.js",               "https://vega.github.io/vega/assets/promise.min.js",
    "vega/symbol.min.js",                "https://vega.github.io/vega/assets/symbol.min.js",
    "vega/vega.js",                      "https://cdn.jsdelivr.net/npm/vega@{vega}/build/vega.js",
    "vega/LICENSE",                      "https://raw.githubusercontent.com/vega/vega/master/LICENSE",
    "vega-embed/vega-embed.js",          "https://cdn.jsdelivr.net/npm/vega-embed@{vega_embed}",
    "vega-embed/LICENSE",                "https://raw.githubusercontent.com/vega/vega-embed/master/LICENSE"
  ) %>%
  mutate(
    path_remote = map_chr(path_remote, ~glue_data(vega_versions_long, .x))
  ) 
```

``` r
get_file <- function(path_local, path_remote, path_local_root) {
  
  path_local <- fs::path(path_local_root, path_local)
  
  # if directory does not yet exist, create it
  dir_local <- fs::path_dir(path_local)
  
  if (!fs::dir_exists(dir_local)) {
    dir_create(dir_local)
  }
  
  resp <- httr::GET(path_remote)
  
  text <- httr::content(resp, type = "text", encoding = "UTF-8")
  
  readr::write_file(text, path_local)
  
  invisible(NULL)
}
```

Here, we create the `lib` directory, then “walk” through each row of the
`downloads` data frame to get each of the files and put it into place.

``` r
dir_lib <- path(dir_htmlwidgets, "lib")
dir_create(dir_lib)

pwalk(downloads, get_file, path_local_root = dir_lib)
```

We need to add a command to include our copy of `vega-embed.css`:

``` r
fs::file_copy(
  fs::path(dir_templates, "vega-embed.css"), 
  fs::path(dir_lib, "vega-embed/vega-embed.css")
)
```

## Patch

Here, Alicia Schep noticed that there was a problem to render vega
charts within the RStudio IDE, and she figured out a workaround (as well
as a [PR]() for the RStudio IDE to fix the problem). Here’s her patch
for older versions of the IDE:

``` r
vega_embed_path <- path(dir_lib, "vega-embed/vega-embed.js")
vega_embed <- readr::read_file(vega_embed_path)

vega_mod <- stringr::str_replace_all(vega_embed, 'head>"','he"+"ad>"') 
vega_mod <- stringr::str_replace_all(vega_mod, '"<\\/head>','"</he"+"ad>') 

readr::write_file(vega_mod, path(dir_lib, "vega-embed/vega-embed-modified.js"))
fs::file_delete(vega_embed_path)
```

## Block

``` r
if (dir_exists(dir_block)) {
  dir_delete(dir_block)
}

dir_create(dir_block)
```

``` r
fs::file_copy(
  fs::path(dir_templates, "vega-embed.css"), 
  fs::path(dir_block, "vega-embed.css")
)

fs::file_copy(
  fs::path(dir_templates, "index.html"), 
  fs::path(dir_block, "index.html")
)
```

## Schema

One of the purposes of this package is to provide a means to validate a
spec.

Having thought about this (perhaps too much), the only reasonable way to
go for a given release to support only a single version of the
javascript libraries and the schema.

``` r
data_schema <- vega_versions_long[c("vega", "vega_lite")]

data_schema_major <- map(data_schema, ~str_extract(.x, "^\\d+"))
names(data_schema_major) <- 
  map(names(data_schema), ~paste(.x , "major", sep = "_"))


schema <- 
  tribble(
    ~path_local,                         ~path_remote,
    "vega/v{vega_major}.json",           "https://vega.github.io/schema/vega/v{vega}.json",
    "vega-lite/v{vega_lite_major}.json", "https://vega.github.io/schema/vega-lite/v{vega_lite}.json"
  ) %>%
  mutate(
    path_local = map_chr(path_local, ~glue_data(data_schema_major, .x)),
    path_remote = map_chr(path_remote, ~glue_data(data_schema, .x))
  )
```

``` r
dir_schema <- here("inst","schema")

if (dir_exists(dir_schema)) {
  dir_delete(dir_schema)
}
dir_create(dir_schema)

pwalk(schema, get_file, path_local_root = dir_schema)
```

## Internal data

``` r
.vega_versions <- vega_versions_long

devtools::use_data(.vega_versions, internal = TRUE, overwrite = TRUE)
```

    ## Saving .vega_versions as sysdata.rda to /Users/ijlyttle/Documents/git/github/vegawidget/vegawidget/R