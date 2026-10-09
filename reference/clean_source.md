# Clean one extracted source without changing its record cardinality.

`clean$rules` is a mapping keyed by cleaned schema field. Each rule has
a `type` and rule-specific arguments, but no repeated `field`
declaration. Raw `source`/`sources` inputs use source column names,
cleaned the same way as extracted columns; `@field` refers to another
rule, including one declared later. Dependencies are resolved before
records are evaluated; each field runs once per record. Types include
copy, regex, constant, default, map, concat, number, date,
date_from_excel, time, date_year, date_month, age, and race. Date rules
use a lubridate order such as `mdy` or `dmy_hms`. Excel serial rules
obtain their epoch from the source workbook, never from rule
configuration. A `regex` rule extracts a capture (named group or 1-based
group number) from a raw source or cleaned reference.
`allow_empty: true` permits an empty capture as `""`. Blank input yields
missing; a nonmatch still fails. Rule failures are recorded by locator
and field; corrections run after all rules and resolve only failures for
that same locator and field. Dependent rules are not re-evaluated after
corrections. Alternatively `clean$script` names a trusted
repository-local R script defining `clean_records(records, skip_mask)`.
Scripts should not download data or modify other files; they are
reviewable code, not sandboxed. `clean$corrections` contains
record_locator, field, replacement, and reason mappings.

## Usage

``` r
clean_source(
  config_path,
  extracted_file = NULL,
  cleaned_destination = "{project_root}/cleaned/{state}/{vintage}_{years}",
  force = FALSE,
  source_file = NULL
)
```

## Arguments

- config_path:

  Source YAML path.

- extracted_file:

  Optional extracted RDS path.

- cleaned_destination:

  Directory template for cleaned artifacts.

- force:

  Bypass a valid cache entry, but not safety checks.

- source_file:

  Optional original Excel workbook path for serial rules. Defaults to
  the config-derived fetched source path.

## Value

Artifact paths, status, and record count.

## Details

`source_record` is a one-element list containing
`<source_id>:<record_locator>`. Source IDs cannot contain colons, so the
first colon separates the ID from the complete locator.
