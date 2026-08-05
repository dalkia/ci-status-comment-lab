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
