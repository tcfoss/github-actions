# TcfOss GitHub Actions

Reusable GitHub Actions shared by TcfOss repositories.

## IssueTracker release API

The IssueTracker composite action validates, retrieves notes for, and resolves releases through the IssueTracker API.

Use it from another repository with the major floating tag:

```yaml
- name: Validate release
  uses: tcfoss/github-actions/.github/actions/issuetracker@v1
  with:
    command: validate
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
```

Fetch release notes and use the generated Markdown file in a later step:

```yaml
- name: Fetch release notes
  id: notes
  uses: tcfoss/github-actions/.github/actions/issuetracker@v1
  with:
    command: notes
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
    release-url: ${{ vars.ISSUETRACKER_BASE_URL }}

- name: Create GitHub release
  uses: softprops/action-gh-release@v2
  with:
    tag_name: v${{ steps.parse-release.outputs.version }}
    body_path: ${{ steps.notes.outputs.notes-file }}
```

Resolve a release after publishing its artifacts. The optional `status` input is sent to the IssueTracker API as the resolve status:

```yaml
- name: Resolve release
  uses: tcfoss/github-actions/.github/actions/issuetracker@v1
  with:
    command: resolve
    api-url: ${{ vars.ISSUETRACKER_API_URL }}
    version: ${{ steps.parse-release.outputs.version }}
    token: ${{ secrets.ISSUETRACKER_API_TOKEN }}
    status: Released
    release-url: https://github.com/${{ github.repository }}/releases/tag/v${{ steps.parse-release.outputs.version }}
```

This repository uses three tag levels:

- `v1.0.0` is an immutable patch release tag.
- `v1.0` points to the latest `v1.0.x` patch release.
- `v1` points to the latest compatible `v1.x` release.

The `v1` tag is the recommended default for consumers that want automatic compatible bug fixes. Use `v1.0` when remaining within a specific minor release line. The release workflow refuses to move an existing `vX.Y.Z` tag and only advances the floating `vX` and `vX.Y` tags.

The repository should also use a GitHub ruleset for tags matching `v*.*.*` that restricts deletion and updates to administrators or the release workflow. The workflow provides a second guard, but repository rules are what protect immutable patch tags from direct pushes.

For production workflows that require reproducible action code, pin the action to a commit SHA and keep the version tag in a comment:

```yaml
# v1.0.0
uses: tcfoss/github-actions/.github/actions/issuetracker@0123456789abcdef0123456789abcdef01234567
```

Releases are created manually from the GitHub Actions tab with the `Release` workflow. Enter the patch version without the `v` prefix, for example `1.0.1`, and select the commit to release.

Supported commands are `validate`, `notes`, and `resolve`. The `notes` and `resolve` commands accept `release-url`; `resolve` also accepts `status`.
