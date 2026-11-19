# GitHub-compatible git forge

[![make.fun · github-core-clone-01](https://make.fun/badge/github-core-clone-01.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![rank](https://make.fun/badge/github-core-clone-01/HALCYON1803/github-core-clone-01/rank.svg)](https://make.fun/bounty/github-core-clone-01/leaderboard?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)
[![reward](https://make.fun/badge/github-core-clone-01/HALCYON1803/github-core-clone-01/reward.svg)](https://make.fun/bounty/github-core-clone-01?utm_source=github&utm_medium=readme&utm_campaign=github-core-clone-01)

A GitHub-compatible git forge in a single TypeScript file — smart-HTTP git
(upload-pack v0/v2, receive-pack), REST v3, GraphQL v4 and MCP — built for the
[make.fun github-core-clone-01 bounty](https://make.fun/bounty/github-core-clone-01).

## Quick start

```bash
node --experimental-strip-types main.ts
```

The server registers itself with `MAKEFUN_ENV_URL` and serves on an
ephemeral localhost port.

## What it does

- **Git**: `git clone` / `push` / `fetch` over smart HTTP, pack v2 with
  delta encoding, shallow clones, branch and ref management.
- **REST v3**: repos, branches, refs, commits, contents, labels, issues,
  pull requests (diff, merge with conflict detection).
- **GraphQL v4**: repository / issue / pull-request queries, `node` lookup,
  create/close issue, create/merge pull request, add comment.
- **MCP**: `create_repo`, `get_repo`, `create_issue`, `list_issues`,
  `get_issue`, `create_pull_request`, `list_pull_requests`,
  `merge_pull_request`.

## Performance notes (v2)

Measured against the submitted v1 on the same kit workloads (median server
CPU per round): push 480 ms → ~190 ms, clone ~80 ms → ~30 ms, read
~250 ms → ~170 ms. The wins come from memoizing git advertisements and
fetch packs per repository state, serving issue reads as prebuilt buffers,
and a hand-split URL fast path in the router. Full details in `REPORT.md`.
