# Extract a fetched source artifact.

Reads a source configuration, verifies the fetched source artifact, and
writes extracted RDS and YAML sidecar artifacts.

## Usage

``` r
extract_source(
  config_path,
  source_file = NULL,
  extracted_destination = "{project_root}/extracted/{state}/{vintage}_{years}",
  force = FALSE
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

## Value

A list describing the extracted artifacts.

## Details

Version 1 extraction supports CSV, Excel, PDF table, and locally saved
HTML source artifacts. CSV configs select columns by source header name
or 1-based position; Excel configs use an explicit full A1 rectangle
range and select columns by source header name or Excel column letter.
Headered PDF table configs select columns by source header name or
1-based position; headerless PDF configs select columns by 1-based
position. HTML configs select record elements with
`extract$record_selector` (CSS) and require
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
