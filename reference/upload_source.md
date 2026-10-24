# Create a Dataverse dataset and upload a configured source file

Creates a CC0 dataset under `parent`, then uploads the local source file
named by the YAML source config. This creates a draft; it does not
publish the dataset. An existing `dataset$doi` is reused; otherwise a
dataset is created and its DOI saved before the file upload. On
successful upload, the numeric file ID and local MD5 are saved as
`database_id` and `source_md5`. A file's `doi` is left untouched until
publication.

## Usage

``` r
upload_source(
  config_path,
  contact_name = Sys.getenv("DV_POC_NAME"),
  contact_email = Sys.getenv("DV_POC_EMAIL"),
  file_source = "{project_root}/source_files/{state}/{vintage}_{years}",
  parent = "mortalityrecords",
  token = Sys.getenv("DV_API_TOKEN")
)
```

## Arguments

- config_path:

  Path to a YAML source config containing `filename`, `state`, `years`,
  and `server`. An optional `dataset` mapping requires `title`,
  `agency`, and `vintage`; `doi`, `description`, and
  `public_records_law` are optional. Without `dataset`, its required
  fields are prompted for and written to the config before creation. New
  datasets require a YYYY-MM-DD `dataset$vintage`.

- contact_name, contact_email:

  Dataset contact details. Defaults to `DV_POC_NAME` and `DV_POC_EMAIL`
  environment variables.

- file_source:

  Directory containing the configured file; supports the same template
  tokens as
  [`fetch_source_file()`](https://uclalawbehindbars.github.io/mortality-cleaning-docs/reference/fetch_source_file.md)'s
  `file_destination`.

- parent:

  Alias of the parent Dataverse (`"root"` for the root).

- token:

  Dataverse API token; defaults to `DV_API_TOKEN`. Never put credentials
  in source configs.

## Value

Invisibly, the parsed response to the file upload.
