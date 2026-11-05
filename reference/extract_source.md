# Extract a fetched source artifact.

Extract a fetched source artifact.

## Usage

``` r
extract_source(
  config_path,
  source_file = NULL,
  extracted_destination = "{project_root}/extracted/{state}/{vintage}_{years}",
  force = FALSE,
  source_table_id = NULL
)
```

## Arguments

- config_path:

  Path to a YAML source configuration file.

- source_file:

  Optional path to the fetched source artifact. When `NULL`, the path is
  derived from the source configuration.

- extracted_destination:

  Destination directory for extracted artifacts. The default value
  expands under `{project_root}/extracted/{state}/` using the values in
  the source configuration. Custom destinations may include
  `{project_root}`, `{state}`, `{vintage}`, or `{years}`.

- force:

  When `TRUE`, rewrite extracted artifacts even when the existing
  artifact pair has current cache identity.

- source_table_id:

  Optional effective source table ID. The complete source config is
  validated before selecting the table. By default all tables run;
  failures report table IDs and retain successful keyed results in the
  error's `results` field.

## Value

A list keyed by source ID, with each entry describing the extracted
artifacts, status, record count, and warnings.

## Details

Reads a source configuration, verifies the fetched source artifact, and
writes extracted RDS and YAML sidecar artifacts.

Version 1 extraction supports CSV, Excel, PDF table, locally saved HTML,
and explicit `none` (zero deaths reported) source artifacts. `none`
requires `expected_record_count: 0`, an empty `clean: {}` mapping, and a
verified source file; it does not parse the file.

CSV configs select columns by source header name or 1-based position;
Excel configs use an explicit full A1 rectangle range and select columns
by source header name or Excel column letter. Headered PDF table configs
select columns by source header name or 1-based position; headerless PDF
configs select columns by 1-based position. HTML configs select record
elements with `extract$record_selector` (CSS) and require
`extract$expected_record_count` and a nonempty `extract$columns` list.
Each HTML column has a `source` label, an optional scoped CSS `selector`
(default: the record element), and an optional `attribute` (default: DOM
text). HTML reads only the local artifact: it never fetches, follows
links, or paginates. Record locators refer to raw matches before blank
filtering; HTML sidecars record selectors, encoding selection (`auto` or
override) and a declared page charset when present, field mappings, and
structured missing/blank-record warnings. Source-specific prose
interpretation, including dates, names, IDs, and locations embedded in
announcement bodies, belongs to cleaning.

Optional `extract.filters` rules remove matching selected records from
CSV, Excel, PDF-table, HTML, or extraction-script output after
selection, text normalization, and blank-row removal, but before
`expected_record_count` checks and publication. Rules use selected
*output* column names (including columns selected only for exclusion),
not source selectors. They share the post-clean filter predicates,
conjunctions, negation, ordered removal, and no-rows/all-rows warnings,
but compare normalized text rather than cleaned numeric or date values.
`is_na` matches missing values, not empty text; `eq` with `""` matches
empty text. Surviving rows retain their original locators. `none` cannot
have filters.

A format's `extract.filters` list is inherited unless a table replaces
it wholesale; a table-level `filters: []` disables it. A null list is
invalid. For configured filters, the sidecar records
`extract_filtered_record_count` (zero for `[]`), counting only rows
removed by rules, not blank or unselected rows, and includes rule
warnings alongside extraction warnings. Invalid or absent selected
output names fail even on cache hits or zero-row results; rule changes
contribute to the existing config/format cache identity. Post-clean
filtering remains separate and retains typed comparisons.

PDF extraction requires Python plus the pinned `pdfplumber` dependency
in the supported environment. PDF passwords are used only to open
public-record PDFs and are not written to sidecars; sidecars record only
whether a password was provided. PDF v1 does not perform OCR,
cropping/region selection, or cross-page row continuation. Extracted
values receive mechanical text normalization only; source value
interpretation is left to later cleaning stages. Unselected parsed
columns are omitted from the RDS, but nonblank dropped columns produce
nonfatal runtime warnings and structured sidecar warnings. Unselected
HTML page content is not a dropped column and produces no dropped-column
warning. A top-level `encoding` may override archived HTML
character-encoding detection.

Extracted artifacts are written to destination-local temporary files
before final replacement. If a handled write or replacement error
occurs, temporary files are removed on a best-effort basis and any
existing final artifact pair is restored. This protects against ordinary
write failures, but does not claim multi-file atomicity across process
termination or operating-system failure.
