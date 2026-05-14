# Contributing Guidelines

This repository holds reusable workflows and the shared linter config used by
every Go repository in the organization. Changes here affect all of them, so
test before merging.

## Changing a workflow

1. Create a branch with the updated workflow.
2. Point a real repository at your branch to test it, e.g.:

   ```yaml
   uses: luvvano/base-workflows/.github/workflows/go_linter.yaml@<TEST_BRANCH>
   ```

3. Open a pull request.

## Changing the linter standard (`.golangci.yaml`)

Because the base-linter always fetches `.golangci.yaml` from the `main` branch,
a change here applies organization-wide as soon as it is merged.

1. Edit `.golangci.yaml` on a branch.
2. Test against at least one or two active Go repositories by pointing their
   lint workflow at `base_workflows_ref: <TEST_BRANCH>`.
3. Note in the PR description which repositories you validated against and
   whether the change is expected to surface new findings.

## Submitting Pull Requests

1. Resolve any merge conflicts.
2. One pull request per fix or feature.
3. A pull request needs at least one approval.
4. It's your responsibility to merge an approved pull request.
