# Publish filtered source records as validated records.

Publish filtered source records as validated records.

## Usage

``` r
validate_source(
  config_path,
  filtered_file = NULL,
  validated_destination = "{project_root}/validated/{state}/{vintage}_{years}",
  schema_path = file.path(project_root(), "datapackage.json"),
  force = FALSE,
  source_table_id = NULL
)
```

## Arguments

- config_path:

  Path to a YAML source-file config.

- filtered_file:

  Optional filtered RDS for a legacy single-table config, or a directory
  of table-ID RDS files for inline or formatted source tables. By
  default, read from the source table's filtered destination.

- validated_destination:

  Destination directory for validated artifacts; accepts
  `{project_root}`, `{state}`, `{vintage}`, and `{years}` tokens.

- schema_path:

  Path to the release Data Package schema, used when resolving source
  tables and converting configured examples and constraints.

- force:

  When `TRUE`, rewrite a current validated artifact pair.

- source_table_id:

  Optional effective source table ID. By default, run all tables. A
  sibling failure retains successful keyed results in the aggregate
  error's `results` field.

## Value

A list keyed by effective source table ID, with each entry describing
the validated artifact pair, status, and record count.

## Details

Run
[`filter_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/filter_source.md)
first, then call `validate_source(config_path)` to check the final
records and publish an unchanged validated RDS with a YAML sidecar. Use
`source_table_id` to validate one table or `force = TRUE` to rerun a
cached result. If checks fail, inspect the failure report beside the
validated artifacts; an earlier successful pair remains available. See
[`vignette("cleaning", package = "dicr")`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/articles/cleaning.md)
for QA examples.
