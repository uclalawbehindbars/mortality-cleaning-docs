# prisonmortality

This repository contains the code and configuration used to produce UCLA
Law Behind Bars’ annual release of in-custody mortality data for each
state prison system in the United States, as well as the federal Bureau
of Prisons.

**NOTE FOR FUTURE: MOST OF THE BELOW SHOULD PROBABLY GO IN A
`CONTRIBUTING.MD` FILE.**

## Repository Design Principles

This repository combines two discrete codebases:

1.  A reusable internal R package that holds code that functions
    independently of any specific state or source record.
2.  A pipeline directory that includes YAML configuration and
    source-specific R code for processing records from each state.

We do not include underlying source records in the repository itself for
a few reasons:

1.  Git is poorly designed for storing binary files that do not change
    much over time.
2.  Including the source files would add unnecessary bloat to the
    repository size, making it more difficult for others to clone and
    use our code.

Instead, we make source files available in a public archive (TO
IMPLEMENT). Running the full pipeline for a state will download files as
needed.

## Development Setup

### Tools

For the best experience working on this repository, it’s helpful to
install a few tools first.

#### A Code Editor or IDE

Most R developers like [RStudio](https://posit.co/downloads), but you
can use whatever tool you prefer, such as
[VSCode](https://code.visualstudio.com/download?_exp_download=fb315fc982)
or [NeoVim](https://neovim.io/). This project will include basic
configuration for each of these tools.

#### Docker and Docker Compose

To ensure reproducible code across platforms, we encourage you to use
[Docker](https://docs.docker.com/desktop/) and [Docker
Compose](https://docs.docker.com/compose/) to create a development
container that holds all of the code required to run this project. Note
that source files are not stored in the development container; instead
we use a bind-mount to access them from each host system.

If you don’t want to use Docker, you can run the code locally instead,
but your results may not be reproducible on other machines.

#### `just`

[`just`](https://just.systems/man/en/) is a modern, Rust-based command
runner. It allows us to save and reuse project-specific commands as
“recipes” that we can use for common tasks. Instead of trying to
remember exactly how to run the test suite in the development container,
you just (no pun intended) have to remember to type `just test`!

### Setup

After installing these tools, you just need to run `just setup`, which
will build the development container with Docker and install all R
packages required to run the code. If you insist on using a local R
setup, you should run `just setup-local` instead.

## Source Data

As noted above, we do not include source data files directly in this
repository. Instead, these files will be accessible in a Dataverse
instance (details TK) and downloaded to a local cache on an as-needed
basis.

The source configuration files for each record contains the Dataverse
UNF (or DOI?) for the current version of the source file on Dataverse,
as well as the MD5 (or SHA256?) sum to confirm file integrity.
