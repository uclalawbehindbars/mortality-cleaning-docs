# Expand a file destination template string

Expand a file destination template string

## Usage

``` r
expand_file_destination(file_destination, source_config)
```

## Arguments

- file_destination:

  A file destination template string, which can include tokens such as
  `project_root`, `years`, `vintage`, or `state`.

- source_config:

  A source configuration list containing values to fill the
  `file_destination` template string.

## Value

A path string using the template and pre-filled values.
