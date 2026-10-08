# Validate the source file MD5 matches the configured value.

Validate the source file MD5 matches the configured value.

## Usage

``` r
validate_source_file_md5(path, source_config)
```

## Arguments

- path:

  The path to the source file.

- source_config:

  A source configuration list including a `source_md5` key.

## Value

A boolean indicating whether or not the source MD5 matches the expected
value.
