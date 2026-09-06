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

### Why one change shows up twice

A consequence of that premise, and the first thing people take for a bug: the previewed changelog
lists the same change twice — once for the commit on the branch, once for the merge commit carrying
the PR title. Both are real. release-please reads both, and the published changelog has both
entries too; it just isn't obvious when the PR title matches the commit subject word for word, since
the two entries then read identically.

What is *not* real is that merge commit's SHA. It exists only inside the runner, so its link
resolves to nothing and GitHub will create a different commit on merge. The preview therefore names
that entry instead of citing it:

```markdown
* **ci:** extract apt-repo publish/sign into reusable composite actions (the merge commit GitHub would create for this PR)
* **ci:** extract apt-repo publish/sign into reusable composite actions ([380196e](…))
```

There is no good lever for removing the doubling: it follows from merging via merge commits at all,
which is the premise this action exists to model. Squash-merging avoids it and takes the prediction
with it. Giving the PR a title distinct from the commit subject at least makes the two entries say
different things, which is how this repository's own changelog reads.

Optionally (default on), it also posts/updates a PR comment previewing the release-please output,
and adds/removes a label on the PR to flag whether merging it would trigger a release.

That comment ends by naming the three inputs it was computed from — the head SHA, the base branch
and the commit it was at, and the PR title as the run was handed it. All three move, and the base
moves without anything re-running this action, since another PR merging is not an event on this one.
The footer is what lets a reader tell a current prediction from one that describes a state the
repository has since left.

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
consumes the output. The changelog release-please produced is still shown, folded, underneath that
explanation — the doubt is over the number, and a comment that showed nothing at all read as if the
action had failed to notice a release was coming.

## The PR that adopts release-please

The one PR whose base branch has no `.release-please-manifest.json` to read is the PR that adds it,
and that is a repository where a prediction is worth the most: nobody has seen this pipeline run
yet. So when the base branch has no manifest, the baseline is read from **this PR's own copy** of
it instead. That matches what release-please itself will do after the merge, since it classifies the
bump against the manifest as the merge leaves it.

Worth knowing what that means for a first release, because it surprises people: the version in the
manifest is one release-please treats as **already released**, not one it will publish — it holds
even with no tag and no GitHub release behind it. Commit `{".": "0.1.0"}` and the first
release-please release is 0.2.0, with 0.1.0 never published at all. Start the manifest at
`{".": "0.0.0"}` for 0.1.0 to be the first published release, since a `feat:` is a minor bump by
default and 0.0.0 plus a minor is 0.1.0.

## Usage

```yaml
on:
  pull_request:
    # `edited` matters here, and isn't in GitHub's defaults — see below.
    types: [opened, synchronize, reopened, edited]
    branches: [master]

# Also not optional — see "Why the concurrency group matters" below.
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true

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

### Which `pull_request` types to trigger on

Add **`edited`** to the defaults. GitHub runs a `pull_request` workflow on
[`opened`, `synchronize`, and `reopened` only](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows)
unless you say otherwise, and that set is not enough for this action:

- **`edited`** — fires when [a PR's title or body is edited](https://docs.github.com/en/webhooks/webhook-events-and-payloads?actionType=edited#pull_request).
  The title is not cosmetic here. This action bakes it into the merge commit body exactly as GitHub
  would, and release-please parses it as a conventional commit — so retitling a PR from
  `fix: ...` to `feat!: ...` changes the predicted bump. Without `edited`, that retitle produces no
  run: the preview comment, the release label and the `version` output all keep asserting the old
  prediction, with nothing to indicate they're stale. This is precisely the drift the action exists
  to catch, so leaving `edited` out undermines the point of running it.
- **`synchronize`** — covers new commits, including a rebase or force-push, on its own.
- **`opened`**, **`reopened`** — the obvious ones.

The cost of `edited` is some redundant runs, since it also fires on body edits, which never affect the
prediction. A run is a few seconds (and skips the dry run entirely on release-please's own release
PR), so this is normally a fair trade for never showing a stale number.

Retargeting a PR onto a different base branch is also reported to fire `edited`, which would matter
for a workflow using a `branches:` filter. GitHub's published description of `edited` mentions only
the title and body, so that isn't confirmed here — treat it as a bonus rather than something to rely
on.

### Why the concurrency group matters

Adding `edited` creates a second problem that the concurrency group solves, so the two belong
together. Force-push a branch and rename its PR at the same time and GitHub delivers two events a
second apart — `synchronize`, carrying the *old* title, then `edited`, carrying the new one. Two
runs start, against the same head SHA but different titles, and therefore different predictions.
Both then delete and repost this PR's comment and set its label, with nothing ordering them. Last to
finish wins, and which run that is has nothing to do with which one held the newer title.

This is not hypothetical. Seen in the wild on a PR renamed from `fix:` to `chore(ci):` while being
force-pushed: the `edited` run finished first, correctly reported no release and removed the label;
the `synchronize` run finished three seconds later, deleted that comment, and left a `0.4.1` patch
prediction and a `RELEASE` label on a PR that would release nothing.

```yaml
concurrency:
  group: ${{ github.workflow }}-${{ github.event.pull_request.number }}
  cancel-in-progress: true
```

Keyed per PR, so concurrent PRs still preview in parallel. `cancel-in-progress: true` also stops
paying for a run whose answer is already superseded; a group without it serialises rather than
cancels, which fixes the ordering too and is why preflight only checks that a `concurrency` group
exists at all.

Cancelling is the only fix here that covers the label as well as the comment. Everything else this
action does to keep a stale answer off a PR works by putting information *in* the comment, and a
label has no body to carry it.

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
- The calling workflow should declare a `concurrency` group keyed per PR, or two runs racing can
  leave the older one's prediction on the PR — see
  [Why the concurrency group matters](#why-the-concurrency-group-matters).

## Preflight checks

Every requirement above is one you can get silently wrong: nothing fails, the prediction is just
quietly computed from a false premise and then reported with exactly the confidence of a correct one.
So the action checks its own preconditions first and reports anything missing:

| check | warns when |
|---|---|
| full history | the checkout is shallow (`git rev-parse --is-shallow-repository`), so release-please may not see back to the last release |
| merge method | the repository has merge commits disabled, meaning no PR here can produce the merge commit this action simulates |
| `edited` trigger | the running workflow file never mentions `edited`, so retitling a PR won't re-run it |
| `concurrency` group | the running workflow file never mentions `concurrency`, so two runs racing can leave the older one's prediction on the PR |
| release-please config | `release-please-config.json` or `.release-please-manifest.json` is absent from the checkout |

Anything found is reported twice: as a job annotation, and as a warning block at the top of the
preview comment (when `post-comment` is `true`), above the version and changelog it qualifies. The
comment is the copy that matters — an annotation is only seen by someone who opens the run summary,
whereas these warnings all amount to "the version below may be wrong", which needs to reach whoever
is about to click merge.

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
| `preflight`     | no       | `'true'`  | Whether to check the calling workflow and repository for the setup this action depends on and report anything missing, as job annotations and at the top of the preview comment. Warnings only — never fails the run, and stays quiet when it can't tell. See [Preflight checks](#preflight-checks). |

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

Nothing about releasing is manual. This repository runs release-please on itself: land a conventional
commit on `master`, release-please opens a release PR, and merging that PR cuts the tag and the GitHub
release. [`.github/workflows/publish-tag.yml`](.github/workflows/publish-tag.yml) then force-moves the
matching major tag (`v1`) onto the same commit.

A missed major-tag move is silent — `@v1` consumers simply stay on older code, with nothing anywhere
to hint at it — which is exactly what happened before this was automated, and why it isn't left to
hand. `publish-tag.yml` is reachable two ways for the same reason: its `push: tags` trigger for a
hand-cut tag, and a direct `workflow_call` from the release workflow, because a tag pushed with
`GITHUB_TOKEN` can't be relied on to fire that trigger.

### It previews its own release PRs

[`.github/workflows/release-please-preview.yml`](.github/workflows/release-please-preview.yml) runs
this action on this repository's own pull requests, using `uses: ./` rather than a released tag — so a
PR runs the action *as that PR would change it*, and one that breaks the prediction fails its own
preview instead of shipping.

That matters more than ordinary dogfooding. The bug behind
[#2](https://github.com/non7top/release-please-forecast/issues/2) only ever surfaced on
release-please's *own* release PR — a shape no unit test reproduces, and one this repository now
generates for itself every release. A regression there shows up as a visibly wrong comment on the PR
that caused it.

## License

MIT
