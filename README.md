# base-workflows

Reusable GitHub Actions workflows for the **luvvano** organization.

This repository provides:

- the company-wide **Go base-linter** — a single shared `golangci-lint`
  configuration that every Go repository in the organization runs against, so
  all code is linted and formatted to the same standard;
- reusable **Go unit-test and coverage** workflows that run the test suite,
  publish the coverage percentage as a check and a PR comment, and optionally
  fail a PR when coverage is below a required minimum.

## What's here

| Path | Purpose |
|------|---------|
| `.github/workflows/go_linter.yaml` | Reusable workflow that runs the base-linter. |
| `.github/workflows/go_unit.yaml` | Reusable workflow that runs unit tests and extracts the coverage percentage. |
| `.github/workflows/go_cover.yaml` | Reusable workflow that publishes coverage and enforces a minimum / no-regression. |
| `.golangci.yaml` | The shared golangci-lint config — the single source of truth for Go code style. |

## Using the base-linter in your repository

Add a workflow to your Go repository, e.g. `.github/workflows/lint.yml`:

```yaml
name: lint

on:
  pull_request:
  push:
    branches: [main]

jobs:
  base-linter:
    permissions:
      contents: read
      pull-requests: write
      checks: write
    uses: luvvano/base-workflows/.github/workflows/go_linter.yaml@main
    secrets:
      # Only needed if the repo imports private github.com/luvvano modules.
      private_modules_token: ${{ secrets.GH_PAT }}
```

That's it. On every pull request the base-linter pulls the shared
`.golangci.yaml` from this repository, runs `golangci-lint` against it, and
posts the result as a PR comment.

If your repository has no private `github.com/luvvano` module dependencies, drop
the `secrets:` block entirely.

### Inputs

All inputs are optional:

| Input | Default | Description |
|-------|---------|-------------|
| `go_mod_path` | `go.mod` | Path to the main `go.mod` file (used to pick the Go version). |
| `base_golangci_lint_version` | `v2.11.4` | golangci-lint version used to run the base config. |
| `base_workflows_ref` | `main` | Ref of this repository to load `.golangci.yaml` from. |
| `runs_on` | `ubuntu-latest` | Runner label for the job. |
| `gomaxprocs` | `4` | Value for `GOMAXPROCS`. |
| `base_linter_args` | `""` | Whitelisted extra args. Only `--build-tags=<comma-separated tags>` is allowed. |

### Secrets

| Secret | Required | Description |
|--------|----------|-------------|
| `private_modules_token` | no | A token (e.g. `GH_PAT`) with read access to private `github.com/luvvano` Go modules. Pass it directly — `private_modules_token: ${{ secrets.GH_PAT }}`. Omit if the repo has no private module dependencies. The token must be valid and have read access to **every** private `github.com/luvvano` repo the module graph pulls in (e.g. `common`, `lib`); a stale repo-level secret will shadow a valid org-level one. |

> Pass the token explicitly via the `private_modules_token` secret. Do **not**
> rely on `secrets: inherit` plus a secret *name* — dynamic secret-name lookup
> does not resolve reliably inside a reusable workflow.

Example with a private-module token and build tags:

```yaml
jobs:
  base-linter:
    permissions:
      contents: read
      pull-requests: write
      checks: write
    uses: luvvano/base-workflows/.github/workflows/go_linter.yaml@main
    secrets:
      private_modules_token: ${{ secrets.GH_PAT }}
    with:
      base_linter_args: --build-tags=integration,unit
```

## Running tests and coverage in your repository

Add a workflow that chains `go_unit.yaml` (runs the tests, extracts coverage)
and `go_cover.yaml` (publishes coverage and enforces the minimum). The example
below enforces a **minimum coverage of 60%**:

```yaml
name: tests

on:
  pull_request:
  push:
    branches: [main]

jobs:
  unit:
    permissions:
      contents: read
    uses: luvvano/base-workflows/.github/workflows/go_unit.yaml@main
    secrets:
      # Only needed if the repo imports private github.com/luvvano modules.
      private_modules_token: ${{ secrets.GH_PAT }}

  coverage:
    needs: unit
    permissions:
      contents: read
      checks: write
      pull-requests: write
    uses: luvvano/base-workflows/.github/workflows/go_cover.yaml@main
    with:
      unit_tests_coverage: ${{ needs.unit.outputs.coverage }}
      required_minimum_coverage: 60
```

On every pull request the tests run, the total coverage is published as a
`coverage` check and a PR comment, and the PR is blocked with a *changes
requested* review whenever coverage drops below `required_minimum_coverage`.
Once coverage is back at or above the threshold the review is dismissed
automatically.

### `go_unit.yaml` inputs

All inputs are optional:

| Input | Default | Description |
|-------|---------|-------------|
| `runs_on` | `ubuntu-latest` | Runner label for the job. |
| `go_mod_path` | `go.mod` | Path to the main `go.mod` file (used to pick the Go version). |
| `gomaxprocs` | `4` | Value for `GOMAXPROCS`. |
| `coverage_file` | `coverage.out` | Filepath to the coverage profile the test command produces. |
| `coverage_file_artifact_name` | `unit-coverage-file` | Artifact name for the uploaded coverage profile. |
| `go_run_tests_with_coverage_cmd` | `go test -v -race -coverpkg=./... -coverprofile coverage.out ./...` | Command that runs the tests and must create the coverage file. |

Outputs: `outcome` (test job outcome) and `coverage` (total coverage percentage).

### `go_cover.yaml` inputs

| Input | Default | Description |
|-------|---------|-------------|
| `runs_on` | `ubuntu-latest` | Runner label for the job. |
| `unit_tests_coverage` | `0.0` | Coverage percentage to publish — pass `${{ needs.unit.outputs.coverage }}`. |
| `required_minimum_coverage` | `0` | Minimum coverage. When `> 0`, the PR fails below this value. Set `60` to enforce 60%. |
| `compare_coverage_with_default_branch` | `false` | When `true`, the PR fails if coverage dropped versus the default branch. |

`go_unit.yaml` accepts the same `private_modules_token` secret as the
base-linter (see below) — omit it if the repo has no private
`github.com/luvvano` module dependencies. `go_cover.yaml` needs no secrets; it
only uses the built-in `GITHUB_TOKEN`.

## Running the linter locally

Install [`golangci-lint`](https://golangci-lint.run/welcome/install/) (v2),
then from your repository root:

```bash
curl -sSL https://raw.githubusercontent.com/luvvano/base-workflows/main/.golangci.yaml -o .golangci.yaml
golangci-lint run
```

To auto-fix everything that can be fixed automatically:

```bash
golangci-lint run --fix
```

> The base-linter always uses the version of `.golangci.yaml` from this
> repository — do not commit a copy into individual repositories.

## Changing the standard

The code style standard lives in `.golangci.yaml` in this repository. To change
it for the whole organization, open a pull request here — see
[CONTRIBUTING.md](./CONTRIBUTING.md).
