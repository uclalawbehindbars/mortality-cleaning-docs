# Disaggregate count columns into inferred death records

Disaggregate count columns into inferred death records

## Usage

``` r
disaggregate_source(
  config_path,
  extracted_file = NULL,
  disaggregated_destination = paste0("{project_root}/disaggregated/{state}/",
    "{vintage}_{years}"),
  force = FALSE,
  source_table_id = NULL
)
```

## Arguments

- config_path:

  Source YAML path.

- extracted_file:

  Optional extracted RDS path for legacy configs, or a directory of
  table-ID RDS files for inline tables.

- disaggregated_destination:

  Directory template for inferred artifacts.

- force:

  Rewrite even if a valid cache entry exists.

- source_table_id:

  Optional table ID; by default process all tables.

## Value

Results keyed by table ID, with paths, status, and record count.

## Details

Run explicitly after
[`extract_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/extract_source.md)
and before
[`clean_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/clean_source.md)
for a table declaring top-level `disaggregate`. Simple counts use
`count_column` and a table-specific `expected_record_count`. A `pivot`
instead selects ordered `pivot_columns`, `values_column_name`
(category), and `count_column_name` (intermediate count); `count_column`
must equal that intermediate name. Consumed count columns are removed
and other source text is carried forward. Do not select overlapping
totals and subgroups: an expected count checks row count, not whether
categories are disjoint.

Alternatively, a trusted `script` defines
`disaggregate_records(records, source_config)` and returns source-shaped
character rows with `record_locator` referencing an original extracted
row. It may return zero or multiple rows per original. The engine groups
rows by extracted order, retains script order within each group, and
replaces those references with inferred locators. Formats may share a
definition without an expected count; each inheriting table supplies its
own count, replaces the entire definition, or opts out with
`disaggregate: null`.

The suffix `!Inferred N` records generation order per original source
row, not an agency-reported individual or durable person identity.
Review corrections targeting these locators after source or script
reordering. Zero counts yield no rows. An expected count mismatch or
invalid count leaves any prior artifact pair unchanged. The sidecar
records `source_record_count` and `record_count` (inferred rows).
Cleaning preserves the inferred row count; post-clean filtering checks
remaining rows.
