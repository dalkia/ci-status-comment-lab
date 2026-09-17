# ci-status-comment-lab

Throwaway repo to validate the **unified CI status comment** from
[`decentraland/unity-explorer#9617`](https://github.com/decentraland/unity-explorer/pull/9617)
end-to-end, without needing Unity.

## What it tests

The real change collapses three separate `github-actions[bot]` comments (Build,
Lint, Tests) into **one** comment whose three sections are each updated
independently by their own workflow, via the `ci-status-comment` composite
action.

Because the real "Unity Cloud Build" and "Unity Test" workflows can't run
outside decentraland, this repo replaces them with **stubs** that emit the same
artifacts the comment workflows parse:

| Stub workflow      | Stands in for      | Emits                                                        |
| ------------------ | ------------------ | ------------------------------------------------------------ |
| `unity-cloud-build`| Unity Cloud Build  | a dummy `Decentraland_windows64` artifact                    |
| `unity-test`       | Unity Test         | `warning-result.json`, `failed-tests-{editmode,playmode}.json` |

The comment workflows (`pr-comment-artifact-url`, `pr-comment-warnings`,
`pr-comment-test-failures`) and the `ci-status-comment` action are copied from
the PR branch. The build one is trimmed (no perf dispatch / S3 / release lookup).

## How to run

Open a PR that edits `trigger.txt`. Both stubs start (`requested`) →
build + lint sections go **pending** (this also exercises the create-race dedup,
since two writers seed the comment at once). ~30s later both finish
(`completed`) → the sections fill in:

- **Build** → Success (stubbed commit + logs)
- **Lint** → Passed (warnings 10 → 8)
- **Tests** → Failed (EditMode all pass, PlayMode 2 failures, with a details list)

Expected end state: **one** comment with all three sections.

## InWorld section (added for #9713)

`inworld-suite.yml` is a fourth stub that validates
[`explorer-automation#86`](https://github.com/decentraland/explorer-automation/pull/86):
the InWorld suite no longer posts its own comment. Instead it runs
`upsert-ci-status.sh` **directly** (the external-caller contract: `NO_CREATE=1` +
`SECTION_BODY_FILE`, `SECTION=inworld`) to fold its result into an `inworld`
section of the unified comment, then retires any leftover standalone
`## InWorld suite` comment.

`inworld` is **on-demand** — it is *not* seeded into the skeleton (the real suite
only runs on release/hotfix PRs into main), so this stub also exercises the
"append a missing section fence to an existing comment" path, and the seed-path
fix that stops a non-skeleton section from wedging the survive check.

The stub seeds a fake leftover standalone comment first, so the run should end
with **one** unified comment (build/lint/tests/inworld) plus that leftover
edited down to a one-line pointer.

## Release-cut sections (added for #10139)

`release-cut.yml` is a fifth stub, standing in for unity-explorer's
`create-release-branch.yml`. It validates
[`unity-explorer#10139`](https://github.com/decentraland/unity-explorer/pull/10139):
a release PR is opened by `GITHUB_TOKEN`, so **no workflow ever runs on it** and
every section of its unified comment would otherwise sit on the skeleton's
"Waiting…" placeholder for the life of the release. The cut step instead fills
build, lint, tests and performance from the runs of the commit it cut from.

The step is copied verbatim from the PR apart from the two producer workflow
filenames and the S3 prefix; the header comment lists the substitutions.

Because the lab has no `dev` line, the workflow takes the commit to cut from as
an input rather than reading a branch tip. Use the head SHA of a PR that has
already run both stubs — the same situation the real workflow is in.

### How to run

1. Open a PR that edits `trigger.txt` and let both stubs finish. `Test (playmode)`
   fails on purpose, so this SHA has a green build, a green `Lint`, a red
   playmode suite and an editmode suite the matrix cancelled with it.
2. Run **Release Cut** from the Actions tab with that PR's head SHA and number.

Run it twice to cover both entry states:

- **delete the unified comment first** → the step is the only writer, so it seeds
  the skeleton and fills four of its six sections
- **leave the comment in place** → the step must overwrite the sections the
  normal workflows wrote, in place, without spawning a second comment. This is
  the property #10139 is actually about: the old code posted a standalone
  comment that nothing could ever update, so testers kept installing the cut
  commit's build after newer commits had landed.

Expected sections either way:

- **Build** → `Success!`, per-platform rows resolved from the build run's artifacts
- **Lint** → `Passed!`, off the `Lint` job's conclusion
- **Tests** → `Failed!`, naming editmode `cancelled` / playmode `failure`
- **Performance** → `Not dispatched`, explaining that the benchmark rides a build
  of the PR itself
- **Automation** → the untouched `On demand` placeholder from the skeleton
