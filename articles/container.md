# Building and using the container

This package, and the data preparation repositories that depend on it,
utilized containers to ensure the user has all the required dependencies
to run the code. It is specifically required for the pdf table
extraction as that code relies on Python’s `pdfplumber` package.

These steps should be taken within the data preparation repository as
each repository may have additional dependencies outside of the dicr
package.

## First Set Up

One you have installed just and Docker, you are ready to build the
container in the terminal.

Many integrated development environments (IDE’s) include a terminal so
you can view code and access the terminal in the same window. For
example, in RStudio the terminal tab is next to Console; if it’s not
avialble you can access it by selecting Code \> Terminal from the header
menu.

Open the terminal at the data preparation repository, and build the
container:

    just setup

If you want to test code while you are extracting/cleaning data, access
R:

    just R

Now you will see what appears to be an R console within the terminal.
Run R code as you would normally, but since you are running it within
the container you can access all dependencies.

## Day-to-Day

Unless there have been changes to the container, you only need to setup
the container one time. Most days you can open the terminal and run
`just R` to access the R console.

## Update `dicr` version in Container

The most common change to the container will be additions or edits to
the dependencies, such as this package.

**Within the R console:**

1.  Install new or updated packages. For example if there is an update
    to the dicr package.

``` r

pak::pak("uclalawbehindbars/dicr") # pak::pak replaces devtools::github_install
```

2.  Update the r-environment:

``` r

renv::snapshot()
#>The following package(s) will be updated in the lockfile:
#>
#> # GitHub ---------------------------------------------------------------------
#>- dicr   [uclalawbehindbars/dicr@HEAD: OLDHASH -> NEWHASH]
#>
#>Do you want to proceed? [Y/n]: y
#>
#>- Lockfile written to "~/Documents/repos/data-preparation-repository/renv.lock".
```

**Within the terminal:**

Update the container with the changes

    just setup
