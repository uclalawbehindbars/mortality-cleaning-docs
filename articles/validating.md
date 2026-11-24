# Validating final source-table records

``` r

library(dicr)
```

Validation checks the records **after** cleaning and filtering. For
example, if cleaning fills in a death year and filtering removes a
header row, QA should inspect the filled-in value and ignore the removed
row. Run the stages in order:

``` r

fetch_source("path/to/source.yaml")
extract_source("path/to/source.yaml")
clean_source("path/to/source.yaml")
filter_source("path/to/source.yaml")
validate_source("path/to/source.yaml")
```

[`validate_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/validate_source.md)
never changes the filtered data. A passing table gets an RDS with the
same rows, columns, values, and order, plus a YAML sidecar under
`validated/{state}/{vintage}_{years}`. A failing table gets a separate
failure report instead of a new validated pair.

## Describe the records you expect

Add `qa` to a legacy single-table source-file config or to an inline
source table. Suppose the final table contains Jane Doe with a recorded
death year, but the source did not provide her death date:

``` yaml
qa:
  required_columns: [record_locator, full_name, death_year]
  expected_blanks: [death_date]
  examples:
    - record_locator: "'Deaths'!A2:D2"
      full_name: Jane Doe
      death_year: 2020
      death_date: null
  constraints:
    death_year:
      min: 2019
      max: 2024
    full_name:
      unique: true
```

The example checks only the fields shown for that surviving locator.
Here `death_date: null` checks the absence of a date, while
`expected_blanks` checks that *every* row lacks one. When
`expected_blanks` is present, any all-`NA` column not listed there also
fails validation. Without `expected_blanks`, all-`NA` columns are not
checked. `required_columns` rejects `NA`; it does not treat empty
strings or whitespace as missing. If Jane’s name differs from the
example, the report identifies her locator without changing the record.

A constraint can also list accepted values with `options: [2019, 2020]`,
or combine `options`, inclusive `min`/`max`, and `unique: true` on one
column. Missing values pass constraints unless the column is also
required. A repeated nonmissing value flags every record sharing it.
Text bounds and options compare exact final values without trimming or
case folding. Configured dates and numbers are converted using the same
schema types as filtering; pass `schema_path` if the schema is not in
the project root.

## Reuse expectations across tables

A format can declare a shared `qa` block. A formatted source table with
no `qa` inherits it; a table with its own block replaces the **whole**
block. For example, if a format requires `full_name` and `death_year`,
this table checks only `full_name`:

``` yaml
source_tables:
  - source_table_id: ca_first
    format: path/to/format.yaml
    qa:
      required_columns: [full_name]
```

`qa: {}` or `qa: null` deliberately publishes without checks. If neither
the format nor the table has a `qa` key, validation warns once per table
on each call, even when its output is already cached. Inline tables use
their own `qa`; a file-level `qa` is not inherited by multi-table
configs.

## Inspect the result

The returned list is keyed by effective source-table ID. Run one table
with `source_table_id`, or omit it to try every table independently:

``` r

result <- validate_source("path/to/source.yaml", source_table_id = "ca_first")
result$ca_first$status  # "written" or "skipped" when the pair is current
```

The successful YAML sidecar includes effective checks, a summary for
each check, input/output record counts, and config and input identity.
When a check fails, the console gives a brief error and
`<source_id>-failures.yaml` contains the full details. For instance, a
missing required year is attributed to its record:

``` yaml
failures:
  - check: required_columns
    column: death_year
    type: records
    failure_count: 1
    record_locators: ["'Deaths'!A2:D2"]
```

Independent checks continue so the report includes other affected
locators as well. A failure leaves any earlier successful validated
RDS/YAML pair intact; a later success or valid cache hit removes the
stale report. To recheck an unchanged input instead of skipping the
cached result, use
`validate_source("path/to/source.yaml", force = TRUE)`. A filtered input
that is not a data frame or lacks `record_locator` fails before QA and
does not create a failure report. See
[Filtering](https://uclalawbehindbars.github.io/mortality-cleaning-docs/articles/filtering.md)
for preparing the input records.
