# Fetch a source file from a Dataverse server.

Downloads a named file from a Dataverse dataset and stores it in a local
destination directory. By default, files are saved under
`{project_root}/source_files/{state}/`, in a subdirectory named with the
dataset vintage followed by compressed contiguous runs of source years.

## Usage

``` r
fetch_source_file(
  config_path,
  file_destination = "{project_root}/source_files/{state}/{vintage}_{years}"
)
```

## Arguments

- config_path:

  A path to a YAML source file. The source file must have `filename`,
  `state`, `years`, `server`, and `source_md5` elements, a
  `dataset$vintage` (legacy top-level `vintage` also works), plus either
  a file `doi` or a numeric `database_id`. `source_md5` validates the
  downloaded file. If a file DOI is not yet available, `database_id`
  identifies the file instead. `dataset$doi` identifies the dataset and
  is not used to fetch a file.

- file_destination:

  Destination directory for the downloaded file. The default value
  expands under `{project_root}/source_files/{state}/` using the values
  in `source_config`. Custom destinations may include `{project_root}`,
  `{state}`, `{vintage}`, or `{years}`.

## Value

The path to the downloaded file, invisibly.
