# Filter a cleaned source.

Filter a cleaned source.

## Usage

``` r
filter_source(
  config_path,
  cleaned_file = NULL,
  filtered_destination = "{project_root}/filtered/{state}/{vintage}_{years}",
  schema_path = file.path(project_root(), "datapackage.json"),
  force = FALSE,
  source_table_id = NULL
)
```

## Arguments

- config_path:

  Path to a YAML source configuration file.

- cleaned_file:

  Optional path to the cleaned source artifact for legacy configs. For
  inline source tables, an explicit path is a directory of table-ID RDS
  files. When `NULL`, paths are derived from the config.

- filtered_destination:

  Destination directory for filtered artifacts. Custom destinations may
  include `{project_root}`, `{state}`, `{vintage}`, or `{years}`.

- schema_path:

  Path to the release Data Package schema. Defaults to
  `datapackage.json` in the project root.

- force:

  When `TRUE`, rewrite filtered artifacts even when the existing
  artifact pair has current cache identity.

- source_table_id:

  Optional effective source table ID. All sibling configs are validated
  before selection; by default all tables run. A runtime error retains
  successful keyed results in its `results` field.

## Value

A list keyed by source ID, with each entry describing the filtered
artifacts, status, and record count.

## Details

Matching rows are removed, in rule order. A filter may instead use a
repository-local script defining
`filter_records(cleaned_records, source_config)`, returning data,
filtered_records, and warnings. When the source has no `filter`
configuration, cleaned rows are passed through unchanged into a filtered
artifact.
