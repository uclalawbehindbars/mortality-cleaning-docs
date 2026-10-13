# AGENTS.md

Guidance for future pi sessions and subagents working in this
repository.

## Project shape

- This is an R package plus data-preparation pipeline for prison
  mortality source extraction, cleaning, and validation.
- Prefer the project vocabulary in `docs/ubiquitous-language.md` and the
  ADRs under `docs/adr/` when naming concepts.
- User-facing descriptions of how the project works are found in:
  - `vignettes/extracting.Rmd`
  - `vignettes/cleaning.Rmd`
- Local planning/issues live under `.scratch/<feature>/issues/`. The
  issue tracker note is `docs/agents/issue-tracker.md`.

## Development commands

Use `just` recipes rather than ad-hoc R commands when possible:

- Setup: `just setup` for Docker, or `just setup-local` for local R.
- Run all tests: `just test`
- Run one test file: `just test-module tests/testthat/test-extract.R`
- Format: `just format`
- Lint: `just lint`
- Regenerate docs: `just docs`
- Full check: `just check`
- Run arbitrary R code: `just _r -q -e <CODE>`

The default runner is Docker. If local R is unavailable in the harness,
use the `just` recipes so Docker can run R.

## Style and conventions

- R indentation uses tabs, not spaces. This is enforced by `.lintr`.
- Keep public API docs in roxygen comments and regenerate `.Rd` files
  with `just docs` after API/documentation changes.
- `README.md` is generated from `README.Rmd`; edit `README.Rmd` and
  rebuild if README content needs changes.
- Prefer narrow, behavior-preserving prefactors before behavior changes
  when duplication makes the change risky.
- Keep extraction mechanical and source-shaped: do not add semantic
  cleaning, value interpretation, deduplication, or source-specific
  transforms to extraction.

## Testing expectations

- Add or update tests for every behavior change, especially source
  extraction edge cases.
- Preserve CSV behavior when changing shared extraction code.
- For Excel extraction, cover sidecar metadata, runtime warnings, record
  locators, cache/force behavior, parser failures, and config validation
  where relevant.
- If you cannot run the full suite, state exactly which command failed
  or was unavailable.
