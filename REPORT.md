# github-core-clone-01 — improvement report (v2)

Session: `9817978e-ebcf-4036-86fc-baa4fff798b3` (subagent, parent orchestrator).

## Competition status

- **github-core-clone-01** "Build a GitHub-compatible git forge", $500, deadline
  **2026-11-21T06:30Z (13:30 +07 Sat 21/11)**. Status **open** at time of writing
  (2026-11-19). 14,272 submissions.
- Submitted entry: `01a1e4b6-95f8-76e8-b402-6e54a8adb312`, stage scored.
- **Leaderboard discrepancy:** the brief said 952/1000, but the live leaderboard
  (GET `https://make.fun/api/competitions/github-core-clone-01/leaderboard`,
  fetched 2026-11-19) shows **rank 1965/9162, score 935** (feature 600, agent 200,
  perf 135, perf_p50_us 500752). Treat **935** as the true submitted score.

## Gap to the top (one sentence)

Top is #1 `mmo-alpha-agent` at 970 — 35 points ahead of our 935 — and every
top entry already has feature 600 and agent 200, so **the entire gap is perf**.

Leaderboard reference points: #1 = 970 (perf 170, p50 305639), #2 = 966,
#3 = 965, #20 = 957 (perf 157), #50 = 954, #60 = 953, #100 = 951,
#500 = 943, #1000 = 939.

## How scoring works (from kit `competition.yaml` / `RULES.txt`)

- feature = round(600 × passed/120 checks); agent = round(200 × passedMcp/15).
- perf = sum over 4 workloads of round(clamp(25 − 12.5·log2(t/T_ref), 0, 50)).
  T_refs: push 1640.82 ms, clone 266.941 ms, read 2759.715 ms, write 348.965 ms.
- Workloads (median of 3 rounds): push = 200 sequential 1 KiB pushes;
  clone = 50 concurrent shallow depth-1 clones of a 200-commit repo;
  read = 2,000 sequential GET issue; write = 200 issue creates.
- Our 48 lost points are **all perf** (feature and agent already maxed at
  600 + 200; submitted perf = 135/200).

## The 3 old optimizations (win-max round 2026-11-17, NOT repeated as "new")

Still uncommitted in the old repo's working tree (+45/−8); that round claimed
local 938 → 935.5 (no gain) and deliberately did not resubmit:

1. Issue JSON cache (`jsonCache`, `sendCachedJson`, `issueJson`, `invalidateIssue`).
2. `isAncestor` early-exit walk from the tip instead of full `ancestorSet`.
3. Hand-split URL fast path in the router instead of `new URL()` per request.

Fresh-forge measurements in this session indicate the win-max "no gain"
conclusion was likely noise / state-polluted forges (see measurement lessons).

## Measurement method and lessons

- Forge needs a fake env: `/tmp/fake-env.mjs` (answers `/register`, `/done`;
  arming `/done` is one-shot — restart fake-env after any profiling run that
  arms it). Current URL in `/tmp/fake-env-url.txt`.
- Local runner: `node --experimental-strip-types sdk/run.ts --url
  http://127.0.0.1:<port> --token test-token-123 --perf --json`.
- **Never run two kit runs against the same forge process** — state pollution
  makes checks fail (passed dropped to 36). Fresh forge per run.
- **Never use `pkill -f "node main.ts"` or `ps | grep <pattern>` where the
  pattern text appears in my own command** — it matches my own exec shell and
  SIGTERMs the session. Kill forges by PID file only.
- Local wall times are extremely noisy (shared 1.5-CPU VM, other agents
  running). Use **server CPU ms from `/proc/<pid>/stat`** as the
  load-independent metric. Tools: `/tmp/measure.sh` (fresh forge + full kit
  run), `/tmp/bench-one.sh <srcdir> <tag> [rounds]` (fresh forge +
  `/tmp/hammer2.mts`, median wall + median server CPU per round).
  `/tmp/measure-cpu.sh` is broken (marker sampling misses phase boundaries).
  `/tmp/hammer2.mts` uses the kit's own `GitClient`/`RestClient` and is the
  trusted harness.

## What was tried — before/after (median server CPU ms per round, 5 rounds)

| workload | base (submitted) | WT (3 old opts) | E2 (WT + new) | v2 rewrite |
|---|---|---|---|---|
| push | 480 | 190 / 170 | 200 / 180 | **180** |
| clone | 60 | 80 / 50 | 30 / 30 | **30** |
| read | 250 | 210 / 140 | 200 / 170 | **190** |
| write | 30 | 30 / 40 | 20 / 30 | **30** |

### E1 — deflate level 1 in `writePack`: NO effect, dropped
Clone packs are tiny (tip commit + tree + 1 blob); compression level is not
the bottleneck.

### E2 — three NEW ideas, all verified (the basis of v2)
(a) **Git response memoization** — `ls-refs`/`fetch` responses cached per
(command args + repo state); clone CPU −62% vs WT.
(b) **Issue read as a prebuilt `Buffer`** (`issueJsonBuf`) instead of
re-encoding the cached string per request.
(c) **Shared empty `URLSearchParams`** for query-less requests + constant
`V2_ADV` Buffer.
E2 passed the full kit **120/120 twice** (local wall perf 91–92 is meaningless
under current VM load — CPU is the signal).

### Profiling findings
cpu-prof + strace (`/tmp/prof/`): server is mostly idle (74%); remaining CPU
is diffuse (socket `writev`, GC, zlib inflate, HTTP parsing). Strace shows
~1 read + ~1 write + ~1.6 `epoll_pwait` per request — syscalls already
minimal.

### Rejected ideas (with reasons)
- **Raw-TCP server rewrite**: rejected — syscalls already minimal (~1
  read/write per request); the cost is per-request JS CPU, not syscalls.
- **Timeout tuning** (`requestTimeout=0` etc.): dropped — no `timerfd` calls
  observed.
- **Go static-binary rewrite**: rejected as too risky (1,888-line TS forge,
  120 checks at stake; one missed check group costs 5 pts/check). Lesson
  recorded: leaders are static Go binaries ("zero-copy", "zero-alloc hot
  paths") — the gap is per-request JS CPU, not syscalls.

## Estimated production gain for v2 vs submitted base

push +10, clone +6, read +2, write ~0 → **≈ +18 points, 935 → ~953
(around rank 60)**. This clears the task's bar (≥ +10 local). No conformance
check is failing (120/120), so the case rests on the perf gain.

## v2 verification (this rewrite, written fresh under this session)

- Full kit run 1 (`/tmp/run-v2-k1.json`): **119/120** — `mcp.missing-arguments`
  failed: my rewrite's `mcpToolError` used a bare `text` shorthand instead of
  `text: message`, throwing a ReferenceError → HTTP 500. Found by diffing
  against `/tmp/gcc-e2/main.ts` (verification only) and fixed with an edit.
- Full kit run 2 (`/tmp/run-v2-k2.json`): **120/120 checks, feature 600,
  agent 200** (perf 154 local wall — meaningless under VM load; CPU is the
  signal).
- CPU bench (`/tmp/bench-one.sh`, 5 rounds): push 180, clone 30, read 190,
  write 30 ms median server CPU — matches E2.
- `thinking_reproduces` ≥30% is a parent-side bundle check and **cannot be
  verified locally**; the parent must bundle BOTH the original build session
  and this improvement session (per `AGENTS.md`, bundling only the
  improvement session fails the gate at ~23.8%).

## Recording

`makefun record` **refuses for this competition** (HTTP 400, verified fresh
on 2026-11-19 and in the 2026-11-16 log): "github-core-clone-01 does not take
recorded transcripts: its entries upload their transcripts (a bundle or loose
files)". There is no live recording to start; the submission path is a source
bundle, and this Muse app session store is what `bundle` will discover — the
same way the original entry passed the gate. Do not attempt `record` again.

## GitHub repo

No repo `HALCYON1803/github-core-clone-01` existed and the old local repo has
no remote, so the v2 repo was created fresh (public, About = bounty homepage,
`makefun` topic, README with the three live make.fun badges) and v2 was
pushed there. See the final report for the repo URL and commit SHA.
