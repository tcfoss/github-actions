# TcfOss GitHub Actions

Reusable GitHub Actions shared by TcfOss repositories.

## IssueTracker API

Six composite actions parse work item references and validate, fetch notes for, or resolve releases and individual work items through the IssueTracker API.

Use `issuetracker-validate` to check that a release's work items are resolved. The optional `status`, `work-item-status`, and `work-items-to-be-resolved` inputs validate the release and its work items against the supplied statuses and allow you to account for work items that will be resolved after validation but before completion:

```yaml
- name: Validate release
  id: validate
  continue-on-error: true
  uses: tcfoss/github-actions/.github/actions/issuetracker-validate@v1
  with:
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    status: Released
    work-item-status: Resolved
    work-items-to-be-resolved: '[123,456]'
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

The action outputs `is-valid`, `failed-work-items`, `release-not-active`, and `has-no-work-items`. `failed-work-items` is a JSON array containing unresolved work items; the final two outputs identify the other documented validation failure modes. Use `continue-on-error: true` when a later step needs to read outputs after validation fails.

Use `issuetracker-notes` to fetch release notes and use the generated Markdown file in a later step. `work-item-base-url` is used to construct links to individual work items. It, along with `release-date` are optional:

```yaml
- name: Fetch release notes
  id: notes
  uses: tcfoss/github-actions/.github/actions/issuetracker-notes@v1
  with:
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
    release-date: 2026-09-09

- name: Create GitHub release
  uses: softprops/action-gh-release@v3
  with:
    tag_name: v${{ steps.parse-release.outputs.version }}
    body_path: ${{ steps.notes.outputs.notes-file }}
```

Use `issuetracker-resolve` to resolve a release after publishing its artifacts. The optional `status`, `work-item-status`, `release-url`, and `work-item-base-url` inputs are sent to the IssueTracker API as query parameters:

```yaml
- name: Resolve release
  uses: tcfoss/github-actions/.github/actions/issuetracker-resolve@v1
  with:
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
    status: Released
    work-item-status: Resolved
    release-url: https://github.com/${{ github.repository }}/releases/tag/v${{ steps.parse-release.outputs.version }}
```

Use `issuetracker-validate-work-items` to check whether a set of work items can be resolved. `work-item-ids` must be a non-empty JSON array of integers. The action outputs `is-valid`, `work-items`, `target-status-error`, and `summary-file`. It exits with an error when validation fails, after writing all outputs and a Markdown summary:

```yaml
- name: Validate work items
  id: validate-work-items
  continue-on-error: true
  uses: tcfoss/github-actions/.github/actions/issuetracker-validate-work-items@v1
  with:
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    work-item-ids: '[123,456]'
    target-status: Resolved
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}

- name: Add validation to PR description
  uses: tcfoss/github-actions/.github/actions/sticky-pr-comment@v1
  with:
    marker: '<!-- work-item-validation -->'
    content-file: ${{ steps.validate-work-items.outputs.summary-file }}
```

Use `issuetracker-resolve-work-items` to resolve the same set of work items. It returns the API result through `is-valid`, `work-items`, and `target-status-error`, and fails when the API reports that any item could not be resolved:

```yaml
- name: Resolve work items
  uses: tcfoss/github-actions/.github/actions/issuetracker-resolve-work-items@v1
  with:
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    work-item-ids: '[123,456]'
    target-status: Resolved
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

Use `issuetracker-work-items-from-pr-description` to collect unique work item IDs from a pull request description containing directives such as `Resolves IT #123` or `Closes IT456`. Matching is case-insensitive, and IDs are returned in first-seen order. The action outputs `work-item-ids` as a JSON array and `has-work-items` as a boolean string:

```yaml
- name: Find referenced work items
  id: work-item-references
  uses: tcfoss/github-actions/.github/actions/issuetracker-work-items-from-pr-description@v1
  with:
    description: ${{ github.event.pull_request.body }}
```

The parser ignores content inside the `<!-- work-item-validation -->` markers used for generated validation results.

## Parsing a release version

`parse-release-version` parses a version out of a `release/vX.Y.Z[-suffix]` branch name, or passes through an explicit `version-override` (e.g. for `workflow_dispatch` inputs) without needing a branch name at all:

```yaml
- name: Parse release version
  id: parse-release
  uses: tcfoss/github-actions/.github/actions/parse-release-version@v1
  with:
    branch: ${{ github.head_ref }}
    version-override: ${{ github.event_name == 'workflow_dispatch' && inputs.version || '' }}
```

## Sticky PR comments

`sticky-pr-comment` adds or updates a marker-delimited block in the current pull request's description, replacing the block on subsequent runs instead of duplicating it:

```yaml
- name: Add coverage summary to PR
  uses: tcfoss/github-actions/.github/actions/sticky-pr-comment@v1
  with:
    marker: '<!-- coverage-summary -->'
    content-file: coverage/SummaryGithub.md
```

## Reusable workflows

`validate-pr-work-items.yml` extracts IssueTracker references from a pull request description, validates them, and adds the resulting Markdown to the description. Validation failures are written to the description before the workflow reports failure. Add this caller workflow to a consuming repository:

```yaml
name: Validate PR Work Items

on:
  pull_request:
    types: [opened, reopened, edited, synchronize, ready_for_review]

jobs:
  validate-work-items:
    if: github.event.sender.type != 'Bot'
    uses: tcfoss/github-actions/.github/workflows/validate-pr-work-items.yml@v1
    permissions:
      contents: read
      pull-requests: write
    with:
      pull-request-description: ${{ github.event.pull_request.body }}
      api-url: ${{ vars.ISSUETRACKER_API_URL }}
      target-status: Resolved
    secrets:
      issuetracker-api-token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

The bot guard prevents description updates made by automation from starting another validation run. GitHub also suppresses most events caused by the repository's `GITHUB_TOKEN`, but the explicit guard keeps the workflow safe when a different token is later used.

`resolve-pr-work-items.yml` extracts IssueTracker references from a pull request description and submits a request to the IssueTracker API to mark them as resolved. Add this caller workflow to a consuming repository:

```yaml
name: Resolve PR Work Items

on:
  pull_request:
    types: [closed]

jobs:
  resolve-work-items:
    if: github.event.pull_request.merged == true
    uses: tcfoss/github-actions/.github/workflows/resolve-pr-work-items.yml@v1
    permissions:
      contents: read
    with:
      pull-request-description: ${{ github.event.pull_request.body }}
      api-url: ${{ vars.ISSUETRACKER_API_URL }}
      target-status: Resolved
    secrets:
      issuetracker-api-token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

`validate-release.yml` parses the version from a release branch, validates it against the IssueTracker API, and posts the release notes to the PR description via `sticky-pr-comment`. It no-ops (skips) unless `branch` starts with `release/`. The optional `release-date` input is forwarded to the release notes fetch; `status`, `work-item-status`, and `work-items-to-be-resolved` are forwarded to release validation:

```yaml
today:
  runs-on: ubuntu-latest
  outputs:
    date: ${{ steps.today.outputs.date }}
  steps:
    - id: today
      run: echo "date=$(date -u +%F)" >> "$GITHUB_OUTPUT"

validate-release:
  needs: today
  if: startsWith(github.head_ref, 'release/')
  uses: tcfoss/github-actions/.github/workflows/validate-release.yml@v1
  permissions:
    contents: read
    pull-requests: write
  with:
    branch: ${{ github.head_ref }}
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    release-date: ${{ needs.today.outputs.date }}
    status: Released
    work-item-status: Resolved
    work-items-to-be-resolved: '[123,456]'
  secrets:
    issuetracker-api-token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

`coverage.yml` downloads `coverage-*` test-result artifacts uploaded by earlier jobs, generates an HTML/Markdown report with ReportGenerator, uploads it as a `coverage` artifact, and posts the summary to the PR via `sticky-pr-comment`:

```yaml
coverage:
  needs: unit-tests
  uses: tcfoss/github-actions/.github/workflows/coverage.yml@v1
  permissions:
    contents: read
    pull-requests: write
```


## Versioning

This repository uses three tag levels:

- `v1.0.0` is an immutable patch release tag.
- `v1.0` points to the latest `v1.0.x` patch release.
- `v1` points to the latest compatible `v1.x` release.

The `v1` tag is the recommended default for consumers that want automatic compatible bug fixes. Use `v1.0` when remaining within a specific minor release line. The release workflow refuses to move an existing `vX.Y.Z` tag and only advances the floating `vX` and `vX.Y` tags.

The repository should also use a GitHub ruleset for tags matching `v*.*.*` that restricts deletion and updates to administrators or the release workflow. The workflow provides a second guard, but repository rules are what protect immutable patch tags from direct pushes.

For production workflows that require reproducible action code, pin the action to a commit SHA and keep the version tag in a comment:

```yaml
# v1.0.0
uses: tcfoss/github-actions/.github/actions/issuetracker-validate@0123456789abcdef0123456789abcdef01234567
```

Releases are created manually from the GitHub Actions tab with the `Release` workflow. Enter the patch version without the `v` prefix, for example `1.0.1`, and select the commit to release.
