# dicr - A data pipeline for deaths in custody

This repository contains a data-cleaning pipeline for downloading and
processing in-custody death records obtained by UCLA Law’s Behind Bars
Data Project.

**NOTE FOR FUTURE: MOST OF THE BELOW SHOULD PROBABLY GO IN A
`CONTRIBUTING.MD` FILE.**

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
