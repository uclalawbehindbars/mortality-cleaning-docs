# Changelog

## dicr 0.8.0

- Added crop option for PDFs.

## dicr 0.7.1

- Improved error message for disaggregation.

## dicr 0.7.0

- Added `html_table` extraction method.
- Added optional disaggregation step.

## dicr 0.6.0

- Added validation/QA stage to run after cleaning and filtering records.
- Enabled extraction from CSVs with embedded newlines.
- Changed CSV record locators to use *logical row number* rather than
  physical line number.

## dicr 0.5.0

- Enabled intermediate cleaning rules.

## dicr 0.4.0

- Added extraction-time filtering.

## dicr 0.3.2

- Updated cleaning docs.

## dicr 0.3.1

- Added `publish_source` function.

## dicr 0.3.0

- No substantive changes; testing version-bump system.

## dicr 0.2.0

- Now adds file DOI to source configs during fetch stage if missing.

## dicr 0.1.0

- Added explicit `type: none` option for extract stage.
- Added reusable format configs to handle files with shared schemas and
  formats
- Enabled multi-table inputs and outputs for pipeline stages
- Support for table-specific pipeline rules for multi-table files
- Consolidated source-table stage execution input dispatch

## dicr 0.0.2

- Support for partial DOB age calculations

## dicr 0.0.1

- Initial package scaffolding.
