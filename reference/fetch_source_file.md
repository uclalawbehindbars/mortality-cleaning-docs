# Fetch a source file from a Dataverse server.

Downloads a named file from a Dataverse dataset and stores it in a local
destination directory. By default, files are saved under
`{project_root}/source_files/{state}/`, in a subdirectory named with the
source vintage followed by each source year.

## Usage

``` r
fetch_source_file(
  source_config,
  file_destination = "{project_root}/source_files/{state}/{vintage}_{years}"
)
```

## Arguments

- source_config:

  A list describing the source file. It must contain `filename`, `doi`,
  `state`, `years`, `vintage`, `server`, and `source_md5` elements.
  `source_md5` is used to validate the downloaded file. `doi` should
  identify the Dataverse file by persistent identifier, and `server`
  should be the base URL of the Dataverse server.

- file_destination:

  Destination directory for the downloaded file. The default value
  expands under `{project_root}/source_files/{state}/` using the values
  in `source_config`. Custom destinations may include `{project_root}`,
  `{state}`, `{vintage}`, or `{years}`.

## Value

The path to the downloaded file, invisibly.
