# base-workflows

Reusable GitHub Actions workflows for the **luvvano** organization.

Currently this repository provides the company-wide **Go base-linter**: a single
shared `golangci-lint` configuration that every Go repository in the
organization runs against, so all code is linted and formatted to the same
standard.

## What's here

| Path | Purpose |
|------|---------|
| `.github/workflows/go_linter.yaml` | Reusable workflow that runs the base-linter. |
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
