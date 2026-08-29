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

## release-please's own release PR

One PR must *not* be dry-run: the release PR release-please opens itself. Merging it lands a bump in
`.release-please-manifest.json` to a version with no tag or GitHub release behind it — that release
is exactly what the PR proposes. Dry-running the merge result leaves release-please unable to find
the commit of the "last release" its own manifest names, so it decides the repository still needs
bootstrapping and replays the *entire* history from the first commit. That re-lists every commit ever
in the changelog, and lets an ancient `Release-As:` footer (release-please's own bootstrap commit
carries one) override the computed bump — which is how a repository sitting at `4.0.2` gets told that
merging its `4.1.0` release PR would release `1.0.0`.

Nothing needs predicting for that PR anyway, so this action detects it — by the
`release-please--` branch prefix or the `autorelease: pending` label — and reports the version
straight from the PR's own manifest bump, skipping the dry run.

As a backstop for any other way the same bootstrapping replay can be triggered (a version in the
manifest whose tag or release went missing, say), a predicted version that isn't strictly newer than
the current one is treated as untrustworthy: `version` and `bump-type` come back empty and the
preview comment explains what looks wrong, rather than a bogus number being passed on to whatever
consumes the output.

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

## Preflight checks

Every requirement above is one you can get silently wrong: nothing fails, the prediction is just
quietly computed from a false premise and then reported with exactly the confidence of a correct one.
So the action checks its own preconditions first and annotates anything missing:

| check | warns when |
|---|---|
| full history | the checkout is shallow (`git rev-parse --is-shallow-repository`), so release-please may not see back to the last release |
| merge method | the repository has merge commits disabled, meaning no PR here can produce the merge commit this action simulates |
| `edited` trigger | the running workflow file never mentions `edited`, so retitling a PR won't re-run it |
| release-please config | `release-please-config.json` or `.release-please-manifest.json` is absent from the checkout |

These **only ever warn** — preflight never fails the run — and they **stay quiet whenever they can't
tell for certain**. A field the token can't read, a workflow file that isn't in the checkout, a
reusable workflow defined in another repository: all pass silently rather than being guessed at. The
asymmetry is deliberate. A warning that fires wrongly on a shared action teaches people to ignore all
of its warnings, so missing a real misconfiguration is the cheaper mistake.

The `edited` check reads only the workflow that is currently running, which needs no YAML parsing:
this action does nothing except on `pull_request` events, so that file necessarily has the trigger
already, and the only remaining question is whether `edited` appears in it. A mention anywhere in the
file — a comment included — counts as configured. Scanning just that one file also avoids having to
work out which workflows reference this action, which isn't reliably answerable once a repository has
been renamed or the action is pinned by SHA.

Set `preflight: 'false'` to switch all of it off.

## Inputs

| Name            | Required | Default   | Description                                                                                                                                                     |
|-----------------|----------|-----------|-------------------------------------------------------------------------------------------------------------------------------------------------------------|
| `github-token`  | yes      | —         | Token for release-please's own API calls and (if `post-comment` is `true`) for commenting/labeling the PR. Composite actions can't read the secrets context directly, so this must be passed in explicitly (e.g. `secrets.GITHUB_TOKEN`). |
| `post-comment`  | no       | `'true'`  | Whether to also post/update a preview PR comment and set/remove the release label as a side effect. |
| `release-label` | no       | `'RELEASE'` | Name of the label to create (if missing) and add/remove/recolor on the PR when `post-comment` is `true`, to flag whether merging it would trigger a release and (via color) what kind of bump it would be. |
| `preflight`     | no       | `'true'`  | Whether to check the calling workflow and repository for the setup this action depends on and annotate anything missing. Warnings only — never fails the run, and stays quiet when it can't tell. See [Preflight checks](#preflight-checks). |

## Outputs

| Name            | Description                                                                 |
|-----------------|------------------------------------------------------------------------------|
| `version`       | Predicted next version (e.g. `1.2.0`) if this PR were merged right now. Empty when there are no releasable changes, and also when a release would happen but the version couldn't be predicted with confidence — check `would-release` to tell the two apart. |
| `would-release` | Whether merging this PR would trigger a release (`"true"`/`"false"`).       |
| `bump-type`     | Predicted bump type (`"major"`/`"minor"`/`"patch"`) if this PR were merged right now. Empty in exactly the cases `version` is empty. |

Only plain `X.Y.Z` versions are recognized; a prerelease or otherwise differently shaped version
comes back empty rather than guessed at.

When `post-comment` is `true`, the release label is also recolored by `bump-type`: red (`D93F0B`) for
major, yellow (`FBCA04`) for minor, green (`0E8A16`) for patch, and grey (`6E7781`) when a release
would happen but the bump couldn't be determined (the preview comment says why). If your workflow
also colors this same label elsewhere (e.g. from release-please's own generated release PR, once
merged), keep both color schemes in sync so the label means the same thing everywhere it shows up.

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
