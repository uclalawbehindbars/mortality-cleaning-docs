# filtering

``` r

library(dicr)
```

After the cleaning stage, we can pass records through an optional
filtering stage.

To filter records, you must add a `filter` section to the source
configuration YAML. The `filter` section uses the following schema:

``` yaml
filter:
  filtered_row_count: <integer>
  filters:
    - column: <column_name>
      predicate: eq|neq|gt|lt|gte|lte|in|contains|is_na
      value: <appropriately typed value, omitted for `is_na` predicate>
      negate: true|false <optional, default false>
      conjunction: <optional mapping to another filter rule>
        type: and|or|xor
        other:
          column: <other_column_name>
          predicate: <predicate, as above>
          negate: <boolean, as above>
          conjunction: <arbitrarily nested mapping>
```

The `filtered_row_count` value is a quality-assurance check to verify
that the filters worked as intended. You should compute this manually
before adding it to the config.

The `filters` identify filter rules that *exclude* rows that match the
filters. In other words, to exclude deaths that occurred after 2025, you
would use the following filter:

``` yaml
filter:
  ...
  filters:
    - column: death_year
      predicate: gt
      value: 2025
```

When multiple filter rules are provided, `filter_source` applies them
sequentially, filtering out rows that match *any* of the filters.

For example, the following configuration would filter out deaths based
on year and location:

``` yaml
filter:
  ...
  filters:
    - column: death_year
      predicate: gt
      value: 2025
    - column: death_location
      predicate: eq
      value: WORK RELEASE
```

## Filter predicates

Filter rules can use the following predicates:

- `eq`: Filters rows where `column == value`
- `neq`: Filters row where `column != value`
- `gt`: Filters rows where `column > value`
- `lt`: Filters rows where `column < value`
- `gte`: Filters rows where `column >= value`
- `lte`: Filters rows where `column <= value`
- `in`: Filters rows where `column %in% value`
- `contains`: Filters rows where `stringr::str_detect(column, value)`
- `is_na`: Filters rows where `is.na(column)`

## Filter negation

Sometimes, it’s easier to filter rows by checking whether a condition is
false rather than true. For example, if you wanted to filter deaths to
only include a specific list of facilities, you would use the following
rule:

``` yaml
filter:
  ...
  filters:
    - column: facility
      predicate: in
      negate: true
      value: ["Facility One", "Facility Two"]
```

This rule would filter any rows where the facility name *does not* match
one of the listed values.

## Conjunctions

In other cases, you may need to use more complex combinations of filters
to get the desired result. For instance, imagine that Facility One was
converted to a civil commitment center in July 2024, but it was used as
a regular prison in earlier years. To filter out deaths that occured in
Facility One after July 1, 2024, you can use the following rule:

``` yaml
filter:
  ...
  filters:
    - column: facility
      predicate: eq
      value: Facility One
      conjunction:
        type: and
        other:
          column: death_date
          predicate: gte
          value: 2024-07-01
```

There are three possible conjunction types:

- `and`: Filters rows where the top-level condition *and* `other` are
  both true
- `or`: Filters rows where the top-level condition *or* `other` are
  true. This is only useful for nested conjunctions, since top-level
  conditions are already applied additively. In other words, use it for
  checking that `A & (B | C)`, not for `A | (B & C)`.
- `xor`: Filters rows where *either* the top-level condition *or*
  `other` are true, but not both.

## Script-based filtering

Of course, as with extracting and cleaning data, it is sometimes easier
to write a bespoke script to perform more complex operations. The
filtering stage uses essentially the same contract for script-based
processing as the other stages in the pipeline.

To use a filter script, replace the `filters` attribute with a `script`
attribute that contains the path to the filter script (relative to the
project root).

``` yaml
filter:
  filtered_row_count: 25
  script: path/to/filter/script.R'
```

The filter script must define a top-level function called
`filter_records` that takes `cleaned_records` (the cleaned dataframe)
and `source_config` as inputs and returns a list containing `data` (the
filtered dataframe), `filtered_records` (a character vector with the
record locators for each row removed during the filtering process), and
`warnings` (a character vector of warning messages).
