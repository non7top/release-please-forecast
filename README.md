# release-please-forecast

A composite GitHub Action that predicts the exact next [release-please](https://github.com/googleapis/release-please)
semver bump a pull request would trigger, *before* it's merged.

## Why

release-please decides the next version by reading conventional-commit messages back to the last
release tag. But if your repository merges pull requests via **"Create a merge commit"**, the
commit that actually lands on your base branch is the *merge commit* — subject
`Merge pull request #N from owner/branch`, body: the PR's title — not the PR branch's own commits.
GitHub bakes the PR title into that merge commit's body, and release-please reads it as its own
conventional commit. That means a plain dry run against just the PR branch's commits can
under-predict the real bump whenever the PR title has drifted from what those commits actually say.

This action closes that gap: it builds the *exact* merge commit GitHub would create for the PR,
dry-runs release-please's CLI against that synthetic history, and reports what would happen — with
no side effects on the real repository unless you opt into them.

Optionally (default on), it also posts/updates a PR comment previewing the release-please output,
and adds/removes a label on the PR to flag whether merging it would trigger a release.

## Usage

```yaml
jobs:
  preview:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      pull-requests: write
    steps:
      - uses: actions/checkout@v7
        with:
          # Full history and tags: this action needs to walk back to the
          # last release tag, and it builds its own merge commit on top of
          # this clone.
          fetch-depth: 0

      - uses: non7top/release-please-forecast@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
```

To only compute the prediction (e.g. to name build artifacts) without touching the PR at all:

```yaml
      - name: Predict next version
        id: predict
        if: github.event_name == 'pull_request'
        uses: non7top/release-please-forecast@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          post-comment: 'false'

      - run: echo "Would release ${{ steps.predict.outputs.version }}"
        if: steps.predict.outputs.version != ''
```

Note: this action leaves the working tree checked out on its own synthetic `preview-merge` branch
(or, if simulating the merge failed, on the PR's own unmerged head commit) so it can compute the
prediction. If a later step in your job needs the PR's real head commit checked out again (e.g. to
build it), restore it explicitly, since this action already fetched it locally:

```yaml
      - run: git checkout ${{ github.event.pull_request.head.sha }}
```

This action only runs anything meaningful on `pull_request` events — it reads
`github.event.pull_request.*` context that isn't populated on other event types.

## Requirements

- The repository must already be configured for release-please (a `release-please-config.json`
  and `.release-please-manifest.json` at minimum) — this action only *previews* what release-please
  would do, it doesn't replace configuring it.
- The checkout step before this action must use `fetch-depth: 0` (full history and tags).
- The repository's pull requests must be merged via "Create a merge commit" for the prediction to
  match reality; squash- or rebase-merge workflows don't produce the merge commit this action
  simulates, so the prediction may not reflect what actually lands.

## Inputs

| Name            | Required | Default   | Description                                                                                                                                                     |
|-----------------|----------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `github-token`  | yes      | —         | Token for release-please's own API calls and (if `post-comment` is `true`) for commenting/labeling the PR. Composite actions can't read the secrets context directly, so this must be passed in explicitly (e.g. `secrets.GITHUB_TOKEN`). |
| `post-comment`  | no       | `'true'`  | Whether to also post/update a preview PR comment and set/remove the release label as a side effect. |
| `release-label` | no       | `'RELEASE'` | Name of the label to create (if missing) and add/remove/recolor on the PR when `post-comment` is `true`, to flag whether merging it would trigger a release and (via color) what kind of bump it would be. |

## Outputs

| Name            | Description                                                                 |
|-----------------|------------------------------------------------------------------------------|
| `version`       | Predicted next version (e.g. `1.2.0`) if this PR were merged right now, or empty if there are no releasable changes. |
| `would-release` | Whether merging this PR would trigger a release (`"true"`/`"false"`).       |
| `bump-type`     | Predicted bump type (`"major"`/`"minor"`/`"patch"`) if this PR were merged right now, or empty if there are no releasable changes. |

When `post-comment` is `true`, the release label is also recolored by `bump-type`: red (`D93F0B`) for
major, yellow (`FBCA04`) for minor, green (`0E8A16`) for patch. If your workflow also colors this same
label elsewhere (e.g. from release-please's own generated release PR, once merged), keep both color
schemes in sync so the label means the same thing everywhere it shows up.

## Permissions

The calling job needs:

```yaml
permissions:
  contents: read
  pull-requests: write # only needed when post-comment is true
```

## Versioning

Tagged releases follow semver (`v1.0.0`, `v1.1.0`, ...), with a moving major tag (`v1`) kept up to
date with the latest compatible release, per the
[GitHub Actions versioning convention](https://docs.github.com/en/actions/creating-actions/about-custom-actions#using-release-management-for-actions).
Pin to `@v1` to track non-breaking updates, or to an exact tag for full reproducibility.

## License

MIT
