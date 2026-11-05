# Publish a source's Dataverse dataset

Publishes the dataset identified by `dataset$doi` in a YAML source
config. This does not change the source config or publish its file DOI.

## Usage

``` r
publish_source(config_path, type = "major", token = Sys.getenv("DV_API_TOKEN"))
```

## Arguments

- config_path:

  Path to a YAML source config with `server` and `dataset$doi`.

- type:

  Publication version type, either `"major"` or `"minor"`.

- token:

  Dataverse API token; defaults to `DV_API_TOKEN`.

## Value

Invisibly, the parsed Dataverse response.
