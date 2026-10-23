# Filter a cleaned source.

Filter a cleaned source.

## Usage

``` r
filter_source(
  config_path,
  cleaned_file = NULL,
  filtered_destination = "{project_root}/filtered/{state}/{vintage}_{years}",
  schema_path = file.path(project_root(), "datapackage.json"),
  force = FALSE
)
```

## Arguments

- config_path:

  Path to a YAML source configuration file.

- cleaned_file:

  Optional path to the cleaned source artifact. When `NULL`, the path is
  derived from the source configuration.

- filtered_destination:

  Destination directory for filtered artifacts. Custom destinations may
  include `{project_root}`, `{state}`, `{vintage}`, or `{years}`.

- schema_path:

  Path to the release Data Package schema. Defaults to

- force:

  When `TRUE`, rewrite filtered artifacts even when the existing
  artifact pair has current cache identity.

## Value

A list describing the filtered artifacts.

## Details

Matching rows are removed, in rule order. A filter may instead use a
repository-local script defining
`filter_records(cleaned_records, source_config)`, returning data,
filtered_records, and warnings.
