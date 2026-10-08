# Extracting source tables

``` r

library(prisonmortality)
```

Extraction converts one fetched source artifact and one YAML source
config into a source-shaped artifact pair:

- `{source_id}.rds`: a tibble of selected source columns plus row-level
  provenance.
- `{source_id}.yaml`: a sidecar with source metadata, cache identity,
  column mappings, counts, runtime information, and structured warnings.

Version 1 extraction supports CSV, Excel, text-based PDF tables, and
locally saved HTML source artifacts. It applies only mechanical
normalization: UTF-8 text, Unicode NFC normalization, whitespace
trimming/collapsing, NBSP replacement, line-break replacement, and
character coercion. It does not interpret source-specific values such as
`N/A`, race labels, sex labels, dates, or duplicate records; those
decisions belong to cleaning.

## Source config shape

A v1 extraction config combines source identity, fetch metadata,
checksum metadata, and extraction instructions:

``` yaml
source_id: ca_deaths_2024
filename: deaths.csv
doi: doi:10.70122/FK2/ABC123/FILEABC
state: ca
agency: CDCR
years:
  - 2019
  - 2020
vintage: "2024-01-31"
server: https://dataverse.example.org
source_md5: 0123456789abcdef0123456789abcdef
extract:
  type: csv
  expected_record_count: 2
  columns:
    - source: Name
    - source: Death Year
    - position: 3
```

Required extraction fields are:

- `extract$type`: `csv`, `excel`, `pdf_table`, or `html`.
- `extract$expected_record_count`: the number of nonblank extracted rows
  expected after blank-row filtering.
- `extract$columns`: selected source columns, in output order.

Excel configs also require `extract$range`, an explicit full A1
rectangle such as `A1:F200` or `'Deaths 2020'!A1:F200`. Named ranges,
whole-column ranges, whole-row ranges, open-ended ranges, and implicit
used-range detection are intentionally unsupported so extraction can
compute reliable cell locators.

PDF configs also require `extract$tables`, a nonempty list of page/table
entries that defines the table rows to extract. HTML configs require
`extract$record_selector`, a CSS selector for candidate record elements.

Each selected CSV or PDF column uses exactly one of:

- `source`: a source header name. Matching first tries exact source
  text, then a cleaned snake-case form. This works for CSV files with
  headers and PDF table configs where every configured table entry has
  `header: true`.
- `position`: a 1-based physical column position in the parsed CSV or
  PDF table. Position selection works for headered and headerless
  CSV/PDF sources.

Excel configs use either `source` or `excel_column`; see the Excel
section below. Optional `extract$header: false` marks a headerless CSV
or Excel range. Headerless CSV configs must select columns by
`position`; headerless Excel configs must select columns by
`excel_column`. Headerless PDF table entries are marked within
`extract$tables` and require PDF columns to be selected by `position`.
Output names are based on source names or parser-generated names,
cleaned with the same deterministic output-name rules.

Configured output-name overrides are intentionally unsupported in v1.
Output names are derived from source names using snake-case cleaning.
HTML rejects cleaned-name collisions; tabular extractors repair
duplicates deterministically.

## CSV example

For this CSV:

``` csv
Name,Death Year,Notes
Jane Doe,2019,
John Roe,2020,
```

and this config:

``` yaml
source_id: ca_deaths_2024
filename: deaths.csv
doi: doi:10.70122/FK2/ABC123/FILEABC
state: ca
agency: CDCR
years:
  - 2019
  - 2020
vintage: "2024-01-31"
server: https://dataverse.example.org
source_md5: <md5 of deaths.csv>
extract:
  type: csv
  expected_record_count: 2
  columns:
    - source: Name
    - source: Death Year
```

run extraction after the file has been fetched into the deterministic
source path:

``` r

result <- extract_source("path/to/source.yaml")
result$rds
result$yaml
```

The RDS contains selected columns followed by provenance columns. All
columns are character:

``` r
name      death_year source_state source_agency source_doi                         record_locator
Jane Doe  2019       ca           CDCR          doi:10.70122/FK2/ABC123/FILEABC    line 2
John Roe  2020       ca           CDCR          doi:10.70122/FK2/ABC123/FILEABC    line 3
```

The sidecar records the selected column mapping:

``` yaml
columns:
- source: Name
  output: name
  position: 1
- source: Death Year
  output: death_year
  position: 2
record_count: 2
expected_record_count: 2
warnings: []
```

Blank rows are filtered after selected-column normalization. The
`record_locator` keeps the physical CSV line number, so locators may
have gaps when blank rows are skipped.

## Excel example

Use the generic Excel extractor for `.xlsx` or `.xls` workbooks where a
single explicit worksheet range contains the source table to extract.
For example, if worksheet `Deaths 2020` has headers in row 1 and data in
rows 2 through 200:

``` yaml
source_id: ca_deaths_excel_2024
filename: deaths.xlsx
doi: doi:10.70122/FK2/ABC123/FILEABC
state: ca
agency: CDCR
years:
  - 2019
  - 2020
vintage: "2024-01-31"
server: https://dataverse.example.org
source_md5: <md5 of deaths.xlsx>
extract:
  type: excel
  sheet: "Deaths 2020"
  range: A1:F200
  expected_record_count: 198
  columns:
    - source: Name
    - excel_column: D
```

`sheet` may be a worksheet name or a 1-based worksheet position. The
range may also name the sheet directly, for example
`'Deaths 2020'!A1:F200`; a sheet-qualified range takes precedence over
`sheet`. If both are supplied and disagree, extraction uses the sheet
named in the range, emits a runtime warning, and records an
`excel_sheet_mismatch` warning in the sidecar.

Headered Excel extraction selects columns with exactly one of:

- `source`: a header name, matched exactly or by cleaned-name
  equivalence.
- `excel_column`: a worksheet column letter such as `A`, `D`, or `AA`
  within the configured range.

Headerless Excel extraction uses `header: false` and supports
`excel_column` selection only:

``` yaml
extract:
  type: excel
  sheet: 2
  range: A1:C50
  header: false
  expected_record_count: 50
  columns:
    - excel_column: A
    - excel_column: C
```

The Excel sidecar records selected columns with their resolved
source/default names, cleaned output names, and worksheet column
letters:

``` yaml
extractor_type: excel
extract_options:
  sheet: Deaths 2020
  sheet_position: 1
  range: A1:F200
  header: true
  locator_style: excel_selected_cells
columns:
- source: Name
  output: name
  excel_column: A
- source: Death Date
  output: death_date
  excel_column: D
```

Excel `record_locator` values identify selected cells by worksheet
coordinates. Contiguous selected cells in the same row are compressed
into ranges, for example `'Deaths 2020'!A2:B2,'Deaths 2020'!D2`.

## HTML example: locally saved announcement page

Use `html` for **one saved local page per source config**, not a live
URL. Save/fetch the page separately, configure its file path and MD5,
then call
[`extract_source()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/extract_source.md)
with that local artifact. Extraction does not fetch a page, load assets,
follow links (even to other saved files), or paginate. The tracked
announcement-page fixture contains ten teasers, selected here as
source-shaped title and body fields:

``` yaml
source_id: az_announcements
filename: announcements.html
doi: doi:10.70122/FK2/ABC123/FILEABC
state: az
agency: ADCRR
years:
  - 2026
vintage: "2026-09-28"
source_md5: <md5 of announcements.html>
extract:
  type: html
  record_selector: ".view.view-taxonomy-term .view-content > .views-row"
  expected_record_count: 10
  columns:
    - source: title
      selector: ".field--name-node-title h2 > a"
    - source: body
      selector: ".field--name-body > p"
```

With `source_file = "path/to/announcements.html"`,
`extract_source("path/to/source.yaml", source_file = "path/to/announcements.html")`
verifies `source_md5` before reading the page, and writes
`az_announcements.rds` and `az_announcements.yaml`. Alternatively place
the saved file at the config-derived source path. The example’s title
retains `Inmate Death Notification – Jeffery Pennington`; its body
retains the entire teaser, including
`Inmate Pennington was admitted to ADCRR custody...`. The page’s
publication date is **not** a death date. Name, age, death date, DOC ID,
and location are embedded in prose, not selected as separate fields.
Deriving them requires source-specific semantic cleaning after
extraction; these selectors neither parse prose nor visit linked detail
pages.

`record_selector` selects matching record elements in document order.
Within **each** record, every `columns` entry requires a nonempty
`source` label; optional `selector` selects one descendant element with
CSS. If omitted, it selects the record element itself. Without
`attribute`, extraction reads DOM text (including hidden descendants),
decodes entities, preserves `<br>` as a text boundary, and mechanically
normalizes whitespace. With `attribute`, extraction instead reads that
attribute’s **literal** value; relative links are not resolved. For
example:

``` yaml
extract:
  type: html
  record_selector: "article.announcement"
  expected_record_count: 1
  columns:
    - source: heading
      selector: "h2"
    - source: detail_link
      selector: "a"
      attribute: href
    - source: record_id
      attribute: id
```

CSS is the only supported selector language: XPath and output-name
overrides are unsupported. Multiple field matches, invalid CSS,
overlapping record elements, parser/encoding errors, unknown config
fields, and expected-count mismatches stop extraction. No field match
(or a missing attribute) on an individual record yields `""`; if a field
is missing from **every** matched record, extraction emits an
`html_field_missing` runtime warning and structured sidecar warning.
Zero record matches emit `html_no_records` (even if the expected count
is zero), while records all discarded as blank emit
`html_all_records_blank`. Blank records are filtered after
normalization, but duplicate nonblank records are retained. Individual
absences do not warn. A count mismatch remains fatal after filtering.

HTML `record_locator` values are `Record 1`, `Record 2`, etc., from raw
matches before blank filtering, so gaps are possible. The sidecar
records `extractor_type: html`, `extract_options` with the CSS
`record_selector`, encoding context (`auto` unless a top-level
`encoding` override is specified), and
`locator_style: html_record_index`. Its `columns` mapping lists each
`source`, cleaned `output`, field `selector` (or `record_element`), and
`value: text` or `value: attribute` (with the attribute name). For the
announcement config, that context includes:

``` yaml
extractor_type: html
extract_options:
  record_selector: ".view.view-taxonomy-term .view-content > .views-row"
  encoding: auto
  locator_style: html_record_index
columns:
- source: title
  output: title
  selector: ".field--name-node-title h2 > a"
  value: text
- source: body
  output: body
  selector: ".field--name-body > p"
  value: text
record_count: 10
warnings: []
```

Runtime warnings are retained in sidecar `warnings`, including on later
cache skips (which need not re-emit them). Unselected page
content—navigation, layout, scripts, publication dates, and unrelated
links—is not a dropped column and produces no dropped-column warning. An
optional top-level `encoding: ISO-8859-1` can override incorrect or
absent archived-page encoding metadata; otherwise parsing detects the
encoding.

## PDF table example

Use `pdf_table` for a text-based PDF where the configured pages contain
tables that `pdfplumber` can detect. PDF extraction is generic: the
config identifies one or more page/table units, then the extractor
concatenates the configured table rows in config order and page order
before applying the same source-shaped output contract used by CSV and
Excel.

``` yaml
source_id: example_deaths_pdf_2024
filename: deaths.pdf
doi: doi:10.70122/FK2/ABC123/FILEABC
state: ca
agency: CDCR
years:
  - 2019
  - 2020
vintage: "2024-01-31"
server: https://dataverse.example.org
source_md5: <md5 of deaths.pdf>
extract:
  type: pdf_table
  expected_record_count: 4
  table_settings:
    vertical_strategy: lines
    horizontal_strategy: lines
  tables:
    - pages: [1]
      index: 1
      header: true
      skip_rows: []
    - pages: [2]
      index: 1
      header: true
      skip_rows: [1]
      table_settings:
        snap_tolerance: 4
  columns:
    - source: Name
    - source: Death Date
    - position: 4
```

Each `extract$tables` entry describes a detected table unit:

- `pages`: one or more 1-based PDF page numbers. Pages are processed in
  the order listed. A configured page that is not present is fatal.
- `index`: the 1-based table index returned by `pdfplumber` for each
  configured page. If `pdfplumber` detects multiple tables on a page,
  inspect the PDF locally and choose the detected table number that
  contains the source rows. A missing table index is fatal.
- `header`: `true` when the first non-skipped row in each configured
  table is a header row. Header rows are used for column names and are
  not emitted as data. When multiple configured headered tables are
  combined, their normalized headers must match exactly. Use `false` for
  headerless tables; every non-skipped row is then data.
- `skip_rows`: 1-based row numbers within each detected table to drop
  before header handling. Use this for repeated titles, footers, or
  other table rows that are not data. Out-of-range skipped rows are
  fatal.
- `table_settings`: optional per-entry `pdfplumber` table settings.
  These override top-level `extract$table_settings` only for that table
  entry.

Top-level `extract$table_settings` are passed to all configured table
entries. The supported setting names mirror the `pdfplumber` table
extraction settings used by this package, including `vertical_strategy`,
`horizontal_strategy`, explicit line coordinates, tolerance settings,
edge length, and minimum-word settings. Explicit vertical or horizontal
lines must be numeric coordinates.

PDF columns use the same `extract$columns` shape as CSV columns:

- `source` selects a header name and is allowed only when every
  configured table entry has `header: true`.
- `position` selects a 1-based table column and works for headered,
  headerless, and mixed headered/headerless configs.

If any configured PDF table is headerless, select all PDF columns by
`position`. Headerless PDF output names come from parser-generated names
such as `X1`, `X2`, then receive the standard output-name cleaning. PDF
`record_locator` values identify raw detected table rows, for example
`Page 2!Table 1!Row 3`; locators reflect the original detected row
numbers, so skipped rows and header rows can create gaps.

The PDF sidecar records the resolved extraction options without copying
source data:

``` yaml
extractor_type: pdf_table
extract_options:
  locator_style: pdf_page_table_row
  password_provided: false
  table_settings:
    vertical_strategy: lines
    horizontal_strategy: lines
  tables:
  - pages: [1]
    index: 1
    header: true
    skip_rows: []
    table_settings:
      vertical_strategy: lines
      horizontal_strategy: lines
```

### PDF dependency expectations and local setup

PDF extraction is the only extractor that needs Python at runtime. The
supported Docker image sets
`RETICULATE_PYTHON=/opt/pdfplumber-venv/bin/python` and installs the
pinned Python dependency `pdfplumber==0.11.7`. That container is the
reproducible environment for PDF extraction.

Local, non-Docker runs are allowed but are more fragile. `reticulate`
must be able to initialize a Python interpreter, and that interpreter
must have the same supported `pdfplumber` dependency installed. If
Python or `pdfplumber` is unavailable, extraction stops with a
`PDF parser dependency problem` before reading the PDF. Parser errors
from `pdfplumber` are reported as `PDF parser problem` failures.
Existing current cache hits skip extraction and therefore do not
re-check the Python/PDF dependency.

### PDF password policy

Some public-record PDFs are encrypted but distributed with a public
access password. For those files, provide the password in the source
config:

``` yaml
extract:
  type: pdf_table
  password: "<public access password>"
  expected_record_count: 4
  tables:
    - pages: [1]
      index: 1
      header: true
      skip_rows: []
  columns:
    - source: Name
```

The password is passed to `pdfplumber` only when opening the PDF.
Sidecars must not contain the password; they record only
`extract_options$password_provided: true` or `false` so reviewers can
tell whether a password was used. Do not put private, non-public, or
person-specific credentials in source configs. If a PDF requires a
password that cannot be treated as a public-record access detail, handle
that source outside v1 extraction until an approved credential policy
exists.

### PDF v1 limitations

The v1 PDF extractor is intentionally limited:

- No OCR: scanned or image-only PDFs are unsupported unless text tables
  are already embedded in the PDF.
- No cropping or region selection: extraction runs table detection on
  the configured page and table index, not on a manually cropped page
  area.
- No cross-page row continuation: each detected table row becomes one
  candidate extraction row. Rows split across pages are not stitched
  together automatically.

## Dropped columns and warnings

Tabular columns parsed from the CSV or inside the configured Excel range
but not selected in `extract$columns` are dropped from the RDS.
Unselected HTML DOM content is outside this dropped-column warning
contract. Extraction still inspects dropped columns after the same
mechanical text normalization used for selected values. If a dropped
column contains nonblank cells, extraction emits a runtime warning and
records structured warnings in the sidecar. For CSV, warnings include
the physical parsed position:

``` yaml
warnings:
- type: dropped_column_nonblank
  source: Notes
  position: 3
  nonblank_count: 1
```

For Excel, warnings include the worksheet column letter instead:

``` yaml
warnings:
- type: dropped_column_nonblank
  source: Notes
  excel_column: C
  nonblank_count: 1
```

For PDF, warnings use the parsed table source name and 1-based table
column position. Columns outside configured PDF table entries are
outside the extraction universe and are not inspected. Columns outside
an Excel configured range are likewise outside the extraction universe.
Dropped-column warnings are nonfatal. They do not add rows or columns to
the extracted RDS.

## Cache identity and force rewrites

Extraction skips rewriting an existing artifact pair when the sidecar
cache identity is current:

- `source_md5`
- `config_md5`
- package version

Use `force = TRUE` to rewrite current artifacts. Force rewrites still
validate the source MD5 and extraction config.

When extraction does rewrite artifacts, it writes each replacement to a
temporary file in the destination directory before replacing the final
RDS/YAML paths. If a handled write or replacement error occurs,
extraction removes temporary files on a best-effort basis and attempts
to restore the previous final artifact pair. This protects against
ordinary write failures, but it is not a multi-file atomic transaction
across process termination or operating-system failure.

``` r

extract_source("path/to/source.yaml", force = TRUE)
```

## Fatal extraction failures

Extraction stops for unsupported types, missing or unknown config
fields, MD5 mismatches, CSV parser or encoding problems, Excel parser
problems, PDF dependency or parser problems, embedded newlines in CSV
cells, invalid or unsupported Excel ranges, invalid PDF table entries or
settings, missing or ambiguous selected columns, duplicate selected
positions or Excel columns, invalid positions, out-of-range
`excel_column` selectors, invalid PDF page/table/row references,
headerless PDF name-based selection, PDF header mismatches, ragged PDF
table rows, invalid/ambiguous HTML selectors or overlapping records,
HTML parser/encoding failures, and expected record-count mismatches.
