# Cleaning

``` r

library(dicr)
```

The cleaning stage in our records-processing pipeline takes an extracted
source and produces a clean row for each row in the extracted source
data. Note that one row may or may not be identical to one unique death;
the cleaning stage does not handle duplicate or split data; it outputs
the same number of rows in the cleaned dataframe as it receives.

Run
[`extract_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/extract_source.md)
first (see [Extracting source
tables](https://uclalawbehindbars.github.io/mortality-cleaning-docs/articles/extracting.md));
cleaning reads its RDS and matching YAML sidecar. The
[`clean_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/clean_source.md)
function never adds, drops, deduplicates, or combines source records.
Aggregate expansion, record resolution, and release validation are
separate stages.

## How It Works

As with the extraction stage, we use a YAML-based declarative approach
to clean data instead of relying on bespoke cleaning scripts for each
data source. The cleaning engine abstracts common cleaning tasks like
renaming columns, parsing numbers or dates, mapping values, and
concatenating data from multiple columns into a consistent set of
cleaning “rules” that we use to produce each column in the cleaned data
fram.

Declare one rule per cleaned schema field under `clean$rules`, keyed by
the output field name. Each rule declares a `source` column, which is
either a reference to a column in the extracted dataframe or a forward
reference to a column created by another cleaning rule. Use the `@`
prefix to create a forward reference (i.e., `@death_date` refers to the
`death_date` column in the cleaned data; `death_date` without the prefix
would refer to the `death_date` column in the extracted data).

For example:

``` yaml
clean:
  rules:
    death_year:
      type: date_year
      source: '@death_date'
    death_date:
      type: date
      source: death_date_text
      order: ymd
    first_name:
      type: regex
      source: name
      pattern: '^(?<given>[^ ]+) (?<family>[^ ]+)(?: Jr[.])?$'
      capture: given
    last_name:
      type: regex
      source: name
      pattern: '^(?<given>[^ ]+) (?<family>[^ ]+)(?: Jr[.])?$'
      capture: family
    full_name:
      type: copy
      source: name
```

The cleaning engine applies rules in dependency order, meaning that
rules referred to with forward references are generated first. In the
example above, the cleaning engine will first produce the cleaned
`death_date` column by running
[`lubridate::ymd()`](https://lubridate.tidyverse.org/reference/ymd.html)
on the `death_date_text` column in the extracted dataframe. Then,
`death_year` is derived from the cleaned `death_date` column using
`as.integer(format(death_date, "%Y"))`.

Likewise, the cleaning engine produces `full_name` column by simply
copying the values from the `name` column in the extracted data; this is
effectively the same thing as using
[`dplyr::rename`](https://dplyr.tidyverse.org/reference/rename.html) to
update a column name.

The `first_name` and `last_name` cleaning rules use regex extractions to
get the first and last names, respectively from the full name text. A
regex cleaning rule must specify the extraction `pattern` and `capture`
group (using either a text-based or numeric index). Using text-based
capture groups can help improve readability if you have a particularly
complicated regex, but numeric capture groups work just fine if you have
a simple regex and want to save some extra typing.

## Cleaning a table that uses a format

A format defines shared `clean.rules`. A formatted table can replace an
existing rule by its exact cleaned-field name under `clean.override`, or
omit an existing rule under `clean.exclude`. An override supplies the
**complete** replacement rule; neither operation adds a new field.
Corrections belong to the table, not the format. For example, if the
format defines `full_name` and `death_year` as in [Extracting source
tables](https://uclalawbehindbars.github.io/mortality-cleaning-docs/articles/extracting.md),
one table entry may vary its cleaning:

``` yaml
- source_table_id: ca_deaths_2020_q2
  extract:
    sheet: Second
    range: A1:B3
    expected_record_count: 2
  clean:
    override:
      full_name:
        type: default
        source: name
        default_value: Unknown
    exclude: [death_year]
    corrections:
      - record_locator: "'Second'!A3:B3"
        field: full_name
        replacement: Verified Name
        reason: Checked against source
  filter:
    filtered_row_count: 2
```

Use this as a *variation* on that table, not as an extra duplicate
entry. Excluding a rule that another rule references with `@` is
invalid. Scripted format cleaning cannot use rule operations; table
corrections are still supported. Other tables using the same format
retain the format’s rules and do not inherit these corrections. For a
different core cleaning stage, choose a different format or opt out with
`format: null` and supply complete inline `extract` and `clean` stages
instead of replacing a whole stage.

## Per-table filtering

A format may define `filter.filters` (rules) or `filter.script` without
a count. For every table with an active inherited filter, set its own
`filter.filtered_row_count` to the **remaining** row count, as in the
preceding example; the inherited rules are applied without repeating
them. A table can instead replace all inherited rules or the script (not
merge individual rules) while supplying its count:

``` yaml
filter:
  filters:
    - column: full_name
      predicate: eq
      value: John Roe
  filtered_row_count: 1
```

This replacement illustrates a `Second` sheet with two records including
John Roe: matching records are removed. To disable an inherited filter,
write `filter: null` on that table (not an omitted `filter`, which is
invalid when the format has a filter). With no filter, the filtering
stage passes cleaned records through unchanged. Inline tables can
declare their own filter rules or script and count. Rule columns refer
to cleaned schema fields; a filtering script defines
`filter_records(cleaned_records, source_config)`, returning `data`,
`filtered_records`, and `warnings`.

## Aggregate data

Some states hate transparency, so they only provided us with aggregate
counts of deaths with varying degrees of specificity. For sources with
aggregate counts, the extraction stage produces a dataframe with one row
per count, with all values stored as characters; for instance,
`"2025","Male","25","Row 5"` might be the `year`, `sex`,
`aggregate_count`, and `record_locator` for one count.

The cleaning stage does not disagreggate these aggregate counts.
Instead, it merely converts them to the correct column names and data
types; cleaning the example row above would produce
`2025,"Male",25,"Source ID:Row 5"` with names `death_year`, `sex`,
`aggregate_count`, and `source_record`. Disaggregation will be applied
at a later stage in the pipeline.

To ensure a consistent API for the reshaping engine, the cleaning engine
produces `aggregate_count` values for *all* sources. If the cleaning
config defines an `aggregate_count` rule, this will be generated from
the extracted data; otherwise, it assumes the data is already
individualized and sets `aggregate_count` to `1` for each row.

## Handling Edge Cases

Data is messy, so it might not always be possible to apply a consistent
rule to generate a cleaned column. Maybe a correctional agency used
`<month_name> <day>, <year>` to format dates in 99% of columns, but for
one random column, they inexplicably used `YYYY-MM-DD` format. Instead
of trying to create rules that handle every imaginable edge case, the
default behavior is for the cleaning engine to produce a typed `NA`
value for cases that don’t fit the configured rule. The validation
engine (not yet configured) will flag unexpected `NA` values, allowing
us to adjust the cleaning config to handle them appropriately.

For rules with a small number of failing edge cases, the simplest
approach is to add a `corrections` section to your cleaning config. The
corrections section identifies individual cells that require manual
fixes and documents their replacements values,along with the reason for
replacing them. For example:

``` yaml
clean:
  rules: ...
  corrections:
    - record_locator: Sheet 1!A5:H5
      field: death_date
      replacement: 2025-05-05
      reason: The DOC used a nonstandard format.
    - record_locator: Sheet 1!A26:H26
      field: last_name
      replacement: Smith
      reason: The DOC used unusual formatting for the generational suffix (SMITHIII).
```

As illustrated above, each correction should specify the
`record_locator` for the row needing correction, the `field` that needs
correcting, the `replacement` value, and the human-readable `reason` for
replacing it.

Corrections are applied only after all rules run. Resolved failures
appear in the successful YAML sidecar’s `rule_diagnostics`, marked
`resolved_by_correction: true` alongside the correction audit. Unknown
sources, invalid configurations, and unexpected errors remain fatal.

Correcting a field (such as a date) does not recompute its derived
fields (such as death year or death age) If the resulting downstream
field is inconsistent, give it its own correction or address it during
later validation. Corrections do not add rule dependencies.

## Cleaning scripts

In some cases, the source data is just too darn messy to play by the
rules. While you’re encouraged to try using the rules-based cleaning
methods first (they’ll usually work), sometimes we need a loose-cannon
data wrangler who gets *results*.

In such instances, you can skip the rules-based cleaning by adding a
`script` key to the cleaning config instead of `rules`. The `script` key
should include a repo-local path to a file that defines an R function
called `clean_records` that takes a `records` dataframe as input and
outputs a clean dataframe with the same number of rows as output.
Whatever you do inside the function is your business, as long as it gets
the job done, but please note that the function runs as trusted R code,
so avoid doing anything that will create unexpected side-effects, such
as creating or deleting files.

`library` calls may not work as expected (I’m not totally sure why – can
experiment with it more if this becomes a quality-of-life hazard), so
the best practice is to use functions from other packages with the
`<package_name>::<function_name>` format, such as
`df |> dplyr::mutate(foo = bar + 1)` instead of
`df |> mutate(foo = bar + 1)`.

Cleaning scripts are only responsible for producing fields that would
otherwise appear in a rules-based config. In other words, if there’s no
`first_name` available in a source record, you don’t need to write a
`clean_records` function that produces an empty `first_name` column; the
cleaning engine will do that for you.

Cleaning scripts may also accept `source_config` as a second argument.
This is an effective table config: its `source_id` is the table’s ID and
file metadata remains available. Existing one-argument scripts keep
working.

In the YAML:

``` yaml
clean:
  script: "pipeline/<state>/scripts/<name_of_script>.R"
```

In the clean script:

``` r
# can write smaller functions to use in cleaning
fnc_to_pull_last_name <- function(name_col){}

clean_records <- function(records, source_config){
  # records = output from extraction
  records |>
    dplyr::mutate(
      last_name = fnc_to_pull_last_name(name_col)
    )
)
}
```

## Running cleaning and inspecting artifacts

Add `clean:` to the same source YAML used for extraction. Raw `source`
and `sources` accept either original column names or cleaned extracted
names. The `years` in source configuration describe the source, not each
row’s death year. Once extraction has produced the default RDS and
sidecar:

``` r

result <- clean_source("path/to/source.yaml", schema_path = "path/to/datapackage.json")
result[["example_source_id"]]$status       # "written" or "skipped"
rows <- readRDS(result[["example_source_id"]]$rds)
sidecar <- yaml::read_yaml(result[["example_source_id"]]$yaml)
```

Stage results are keyed by effective table ID even for legacy one-output
configs; replace `example_source_id` with the ID in your configuration.
With `source_tables`,
[`clean_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/clean_source.md)
and
[`filter_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/filter_source.md)
process all tables by default, or accept `source_table_id` to process
only that table. Run extraction, then cleaning, then filtering for each
desired table:

``` r

config <- "pipeline/ca/deaths-2020.yaml"
extract_source(config, source_table_id = "ca_deaths_2020_q2")
cleaned <- clean_source(config, source_table_id = "ca_deaths_2020_q2")
filtered <- filter_source(config, source_table_id = "ca_deaths_2020_q2")
filtered[["ca_deaths_2020_q2"]]$status
```

For a whole-file run, omit `source_table_id` in each call. All sibling
configs are validated before selection or processing; a runtime failure
in one table still attempts the others, and the aggregate
`source_table_stage_error` exposes successful keyed entries in `results`
and failed entries in `failures`. The selected format path and MD5 are
recorded in each output sidecar alongside the source-file config MD5.
Changing either whole YAML file invalidates affected cache identities.
Stage results do not support legacy direct `result$rds` access; index
with the effective table ID. The default extracted and cleaned locations
are respectively `extracted/{state}/{vintage}_{years}/{source_id}.rds`
and `cleaned/{state}/{vintage}_{years}/{source_id}.rds`, each with an
adjacent `.yaml` sidecar. Pass `extracted_file` and
`cleaned_destination` for custom locations; the latter is a directory,
not an output filename. Pass the release Data Package as `schema_path`;
when omitted, cleaning looks for `datapackage.json` in the project root,
not next to the source config or inside the package. The cleaned RDS has
the fields in the supplied schema except `death_id`, plus
`aggregate_count` and row provenance. Unpopulated fields remain typed
missing; raw source columns remain only in the extracted artifact.
`state` and the default individual `aggregate_count = 1` are engine
supplied. Each row’s `source_record` is `<source_id>:<record_locator>`.
Source IDs cannot contain colons, so the first colon separates the ID
from the entire locator, including any embedded colons.

The sidecar records counts, cache identity, checksum, applied correction
audit, and resolved rule diagnostics. A valid pair is skipped unless
`force = TRUE`. Config, input RDS and sidecar, schema, package version,
and script content (if used) contribute to cache identity. Missing
inputs, invalid configurations, unresolved rule errors, malformed script
output, or provenance/count failures stop the run without replacing the
previous successful artifact pair. Cleaning does not run general
release-schema or cross-field coherence validation.

## Rules and source values

A rule key must be a cleaned schema field or `aggregate_count`, not an
engine-owned field such as `state` or `record_locator`. A rule has
`type` and its inputs (see below). Duplicate YAML keys and dependency
cycles fail before processing rows. Missing or empty extracted strings
become missing input. `blank_values` adds exact aliases per rule after
extraction’s text normalization. Explicit maps must list every nonblank
source value, including identity mappings. Common types are:

| Type | Inputs and behavior |
|:---|:---|
| `copy` | `source`: transfer a value, coerced to the target type. |
| `constant` | `value`: supply the same value on each row. |
| `default` | `source`, `default_value`: copy and coerce nonblank input; fill missing input with `default_value`. |
| `map` | `source`, named `map`: explicitly map each nonblank value. |
| `concat` | `sources`, `separator`: join nonblank inputs in order; all missing yields missing. |
| `number` | `source`: parse a finite number; integer/year targets require whole numbers. |
| `date` | `source`, required lubridate `order` (e.g., `mdy`, `ymd`, `dmy_hms`): parse text as a date, discarding any time. Invalid nonblank dates fail. |
| `date_from_excel` | `source`, optional `date_only`: parse an Excel serial using the workbook epoch. |
| `time` | `source`: extract an Excel serial’s fractional time as `HH:MM:SS`, truncating seconds. |
| `date_year`, `date_month` | `source: '@death_date'`: derive year or English month from a parsed date. |
| `age` | `sources: ['@birth_date', '@death_date']`, `[birth_year, death_year]`, `[birth_year, '@death_date']`, or `[birth_year, birth_month, '@death_date']`: completed years at death; birth after death fails. Missing month/day in year-only values are interpolated as July 1; missing birth day with a known month is interpolated as the 15th. Year and month sources may be raw columns or `@` cleaned-field references. |
| `regex` | `source`, `pattern`, `capture`, optional `allow_empty: true`: extract one named or 1-based numbered group; allow an empty capture as `""`. |
| `race` | `source`, literal `separator`, named `map`: split, trim, map, deduplicate, and order schema race categories. |

For a reported death and birth date, define separate `date` rules for
`death_date` and `birth_date`, followed by
`death_age: {type: age, sources: ['@birth_date', '@death_date']}`.
`race` is a list field (which we might change): a missing input remains
missing, while an explicitly mapped empty list represents a reported
empty set. A complex name requiring reordering or interpretation may
instead need a script or correction.

### Excel dates and times

Extraction retains Excel serials as character values. `date_from_excel`
and `time` use
[`get_excel_epoch_origin()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/get_excel_epoch_origin.md)
to read the original workbook’s 1900 or 1904 epoch. The workbook must
still exist at the config-derived fetched path or be supplied as
`source_file` to
[`clean_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/clean_source.md).
Its MD5 is checked against `source_md5`, including on cache hits. For a
fractional serial, `date_from_excel` requires `date_only: true` to
discard time explicitly; a separate `time` rule can preserve the
fraction:

``` yaml
clean:
  rules:
    death_year:
      type: date_year
      source: '@death_date'
    death_date:
      type: date_from_excel
      source: serial
      date_only: true
    death_time:
      type: time
      source: serial
```

Use `date` with a declared `format` for text; no date-format or
workbook-epoch guessing is performed. Unsupported workbook types such as
`.xls` cannot supply an epoch through `openxlsx`, so we may need to add
a manual `date_epoch` override feature.
