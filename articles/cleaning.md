# cleaning

``` r

library(prisonmortality)
```

[`clean_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/clean_source.md)
turns one extracted source into release-shaped records. Run
[`extract_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/extract_source.md)
first (see [Extracting source
tables](https://uclalawbehindbars.github.io/mortality-cleaning-docs/articles/extracting.md));
cleaning reads its RDS and matching YAML sidecar. It never adds, drops,
deduplicates, or combines source records. Aggregate expansion, record
resolution, and release validation are separate stages.

Declare one rule per cleaned schema field under `clean$rules`. The key
is the output field, so do not repeat `field` inside the rule. Raw
`source` names are cleaned like extraction column names; a name prefixed
with `@` refers to another cleaned rule. References may point forward:
cleaning resolves dependencies before processing records, then evaluates
each rule once per record in dependency order. Independent rules do not
depend on YAML ordering. These field-keyed rules supersede the
ordered-rule and corrected-rule-skipping behavior described in ADR 0005;
the retired multi-output `name` rule is not supported. For example, when
the raw `name` column contains `Dr. Jane Doe Jr.`, keep its recognized
title and suffix in `full_name` by copying the raw value:

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
      pattern: '^(?:Dr[.] )?(?<given>[^ ]+) (?<family>[^ ]+)(?: Jr[.])?$'
      capture: given
    last_name:
      type: regex
      source: name
      pattern: '^(?:Dr[.] )?(?<given>[^ ]+) (?<family>[^ ]+)(?: Jr[.])?$'
      capture: family
    full_name:
      type: copy
      source: name
```

A `regex` capture selects a named group or a 1-based group number. Use
one rule for each name component rather than a multi-output name rule.
Set `allow_empty: true` on a regex rule when an empty capture is valid
(for example, a missing middle name); it returns `""` instead of
failing. A missing or blank input remains missing even with
`allow_empty: true`. For field-keyed rules, malformed nonblank values,
unmapped categories, and regex nonmatches become typed missing values
during rule evaluation. Dependent rules see these missing values. Each
failure has a `source_id`, `record_locator`, `field`, and `message`
diagnostic. Corrections are applied only after all rules run. A
correction to the same locator and field resolves the failure; any
unresolved failure stops cleaning without replacing an existing artifact
pair. Resolved failures appear in the successful YAML sidecar’s
`rule_diagnostics`, marked `resolved_by_correction: true` alongside the
correction audit. Unknown sources, invalid configurations, and
unexpected errors remain fatal.

Correcting a date does not recompute its derived year or age, even when
the original date failed. If the resulting downstream field is
inconsistent, give it its own correction or address it during later
validation. Corrections do not add rule dependencies. Use `clean$script`
instead of `clean$rules` when the source requires a trusted
repository-local `clean_records(records, skip_mask)` function. Scripts
and manual corrections retain the same artifact contract.

## Running cleaning and inspecting artifacts

Add `clean:` to the same source YAML used for extraction. Raw `source`
and `sources` accept either original column names or cleaned extracted
names. The `years` in source configuration describe the source, not each
row’s death year. Once extraction has produced the default RDS and
sidecar:

``` r

result <- clean_source("path/to/source.yaml")
result$status       # "written" or "skipped"
rows <- readRDS(result$rds)
sidecar <- yaml::read_yaml(result$yaml)
```

The default extracted and cleaned locations are respectively
`extracted/{state}/{vintage}_{years}/{source_id}.rds` and
`cleaned/{state}/{vintage}_{years}/{source_id}.rds`, each with an
adjacent `.yaml` sidecar. Pass `extracted_file` and
`cleaned_destination` for custom locations; the latter is a directory,
not an output filename. The cleaned RDS has the fields in
`datapackage.json` except `death_id`, plus `aggregate_count` and row
provenance. Unpopulated fields remain typed missing; raw source columns
remain only in the extracted artifact. `state` and the default
individual `aggregate_count = 1` are engine supplied. Each row’s
`source_record` is `<source_id>:<record_locator>`. Source IDs cannot
contain colons, so the first colon separates the ID from the entire
locator, including any embedded colons.

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
`type` and its inputs, but no `field` or `fields`. Duplicate YAML keys
and dependency cycles fail before processing rows. Missing or empty
extracted strings become missing input. `blank_values` adds exact
aliases per rule after extraction’s text normalization. Explicit maps
must list every nonblank source value, including identity mappings.
Common types are:

| Type | Inputs and behavior |
|:---|:---|
| `copy` | `source`: transfer a value, coerced to the target type. |
| `constant` | `value`: supply the same value on each row. |
| `default` | `source`, `default_value`: fill missing input only; unexpected nonblank input is an error. |
| `map` | `source`, named `map`: explicitly map each nonblank value. |
| `concat` | `sources`, `separator`: join nonblank inputs in order; all missing yields missing. |
| `number` | `source`: parse a finite number; integer/year targets require whole numbers. |
| `date` | `source`, required lubridate `order` (e.g., `mdy`, `ymd`, `dmy_hms`): parse text as a date, discarding any time. Invalid nonblank dates fail. |
| `date_from_excel` | `source`, optional `date_only`: parse an Excel serial using the workbook epoch. |
| `time` | `source`: extract an Excel serial’s fractional time as `HH:MM:SS`, truncating seconds. |
| `date_year`, `date_month` | `source: '@death_date'`: derive year or English month from a parsed date. |
| `age` | `sources: ['@birth_date', '@death_date']`: completed years at death; birth after death fails. |
| `regex` | `source`, `pattern`, `capture`, optional `allow_empty: true`: extract one named or 1-based numbered group; allow an empty capture as `""`. |
| `race` | `source`, literal `separator`, named `map`: split, trim, map, deduplicate, and order schema race categories. |

For a reported death and birth date, define separate `date` rules for
`death_date` and `birth_date`, followed by
`death_age: {type: age, sources: ['@birth_date', '@death_date']}`.
Forward references work, but corrections cannot change dependent values.
`race` is a list field: a missing input remains missing, while an
explicitly mapped empty list represents a reported empty set. A complex
name requiring reordering or interpretation may instead need a script or
correction.

### Excel dates and times

Extraction retains Excel serials as character values. `date_from_excel`
and `time` use
[`get_excel_epoch_origin()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/get_excel_epoch_origin.md)
to read the original workbook’s 1900 or 1904 epoch; never configure
`date_system`. The workbook must still exist at the config-derived
fetched path or be supplied as `source_file` to
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
`.xls` cannot supply an epoch through `openxlsx`.

## Corrections and scripts

`clean$corrections` is available with either rules or scripts. Each
correction identifies exactly one `(source_id, record_locator, field)`
and supplies a typed `replacement` and human-readable `reason`. Use
actual locators from the extracted RDS. Stale or ambiguous locators,
repeated corrections to the same field/row, protected targets, and
invalid replacements fail. A corrected successful declarative rule is
still evaluated and its prior value is audited; an errored rule yields
typed missing and a resolved diagnostic. A correction cannot resolve
another field’s error, and it never recomputes downstream rules.

For scripts, set `clean$script` instead of `clean$rules` to a
repository-local path. The script defines
`clean_records(records, skip_mask)`, receives the all-character
extracted data frame, and returns one cleaned-schema data frame row per
input row. `skip_mask` is a logical matrix indexed by row and target
field with `TRUE` for cells the engine will correct. It contains no
replacement values, so scripts can avoid work for corrected fields
without applying corrections themselves. The engine checks row count,
output columns, and provenance before applying corrections. Scripts are
trusted reviewable code, not sandboxed; they should not download data or
modify other files.
