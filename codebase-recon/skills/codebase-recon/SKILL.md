---
name: codebase-recon
description: >-
  Diagnose an unfamiliar or legacy codebase from its git history BEFORE reading
  source files. Runs a fixed set of git-log probes (change/churn hotspots,
  contributor bus factor, bug clustering, commit-velocity trend, firefighting
  frequency), cross-references them, and produces a prioritized file-reading list
  plus a short repo-health summary. Use this whenever picking up, onboarding to,
  auditing, or orienting in a repository you don't already know well — even when
  the user only says "help me understand this project", "where do I start",
  "what's risky here", "review this codebase", or "I just cloned X". Can be scoped to a
  single folder or file to stay fast and low-cost on large repos and monorepos. Language-
  and stack-agnostic; works on any git repository.
---

# Codebase Recon

The fastest way to understand an unfamiliar codebase is not to start reading files
top-to-bottom. The commit history already encodes where the pain is, who owns what,
and whether the project is healthy. Mine that first, then read the highest-signal
files with context instead of wandering.

This skill is a **reconnaissance pass**, not an audit. The deliverable is a **structured
report written in Korean**, organized under the five questions below, closing with a
prioritized reading list. Present each finding cleanly — short ranked lists or tables with
interpretation — rather than pasting raw terminal output at the user. Section headings and
all prose in the report are Korean; keep file paths, author names, and command output
verbatim.

## Why each signal matters (read this — it's the actual product)

The commands are trivial to copy. The value is in interpretation and in knowing when a
signal is **invalid**. A confident wrong conclusion is worse than no conclusion, so each
probe below lists what kills its signal.

## Step 0 — Prepare

Run these checks before the probes; skipping them produces garbage.

1. **Resolve the target scope — do this first, it controls cost.** Scanning a whole repo's
   history is slow and burns tokens for little gain. Pick the narrowest scope that answers
   the user's question and set `<TARGET>` accordingly:
   - **User named a folder** (`src/auth/`, `packages/billing/`) → `<TARGET>` = that path;
     scope every probe to it. This is the default for large repos and monorepos.
   - **User named a single file** → use **Single-file mode** (below) instead of the
     ranked-list report; the "most-changed file" framing is meaningless for one file.
   - **User named nothing** → detect source roots (step 3), and if the repo is large
     (`git rev-list --count HEAD` in the thousands, or many packages) confirm the scope
     with the user before scanning everything. Don't silently scan the whole project.

   Always state the chosen scope in the report header.

2. **Confirm it's a git repo and not shallow.** Full history is required for velocity and
   authorship.
   ```bash
   git rev-parse --is-inside-work-tree
   git rev-parse --is-shallow-repository   # if "true", history is truncated
   ```
   If shallow, run `git fetch --unshallow` (with the user's OK) or flag that
   velocity/bus-factor results are unreliable.

3. **Locate the source roots (only if no target was given).** Lockfiles, generated code,
   vendored deps, and docs otherwise dominate every "most changed" list. Inspect the tree:
   ```bash
   git ls-files | sed 's#/.*##' | sort | uniq -c | sort -rn | head
   ```
   Identify the real source dirs (`src/`, `app/`, `lib/`, `packages/*/src`, `cmd/`,
   `internal/`). In a monorepo, run per package, not across the whole tree.

4. **Set a noise filter.** Use git pathspec excludes so the approach stays
   language-agnostic instead of hardcoding one stack's layout:
   ```
   ':(exclude)*.lock' ':(exclude)*-lock.json' ':(exclude)*.lockb'
   ':(exclude)dist/*' ':(exclude)build/*' ':(exclude)vendor/*'
   ':(exclude)*.min.*' ':(exclude)*.generated.*' ':(exclude)**/__generated__/*'
   ```
   Add whatever the repo treats as generated/committed-output.

**Cost controls.** Beyond scoping `<TARGET>`: narrow `--since` (6 months for an active repo),
keep `head -20`, and never pipe raw un-aggregated `git log` output into context — the
`sort | uniq -c` aggregation is what keeps the output (and token cost) bounded regardless of
repo size.

## The five probes

Substitute `<TARGET>` with the scoped path(s) from Step 0 plus the excludes (e.g.
`-- src/auth/ <excludes>`). If no target was given, `<TARGET>` is the detected source roots.

### 1. Churn hotspots — what changes the most
```bash
git log --format=format: --name-only --since="1 year ago" <TARGET> \
  | sort | uniq -c | sort -rn | head -20
```
The most-edited files over the last year. **High churn alone is not "bad"** — it can just
be active development. The danger sign is high churn on a file *nobody wants to own*:
every change is a patch on a patch and the blast radius of a small edit is unpredictable.
Treat the top 5 as candidates, not conclusions. Confirm against probe 3.

### 2. Bus factor — who built this
```bash
git shortlog -sn --no-merges
git shortlog -sn --no-merges --since="6 months ago"
```
Contributors ranked by commits. If one person is ≥~60% of total, that's the bus factor.
Compare the all-time list to the 6-month list: **if the top historical contributor is
absent from the recent window, the people who built the system are not the people
maintaining it** — flag this prominently.

*Invalidated by:* squash-merge workflows. If every PR is squashed to one commit, this
reflects who *merged*, not who *wrote*. Check the merge strategy (look at merge-commit
ratio: `git log --oneline --merges | wc -l` vs total) before drawing authorship
conclusions.

### 3. Bug clustering — where defects concentrate
```bash
git log -i -E --grep="fix|bug|broken|hotfix|regression" --name-only --format='' <TARGET> \
  | sort | uniq -c | sort -rn | head -20
```
Same shape as probe 1, filtered to commits whose messages mention fixing things. Files
high on **both** this list and the churn list are the highest-risk code in the repo:
they keep breaking and keep getting patched but never get properly fixed.

*Invalidated by:* poor commit-message discipline. If commits say "update stuff", this
returns noise. If the team references an issue tracker instead (e.g. `JIRA-1234`), adapt
the grep to the ticket pattern. Zero results means *either* clean code *or* no
discipline — disambiguate before concluding.

### 4. Velocity trend — accelerating or dying
```bash
git log --format='%ad' --date=format:'%Y-%m' | sort | uniq -c
```
Commits per month across full history. Read the **shape**, not the numbers:
- Steady rhythm → healthy.
- A sudden drop by half in one month → usually someone left.
- A slow decline over 6–12 months → the team is losing momentum.
- Spikes then quiet → work is batched into releases, not shipped continuously.

This is *team* data, not code data. Don't over-read short-term dips (holidays, freezes).

### 5. Firefighting rate — how often the team is in crisis
```bash
git log --oneline --since="1 year ago" | grep -iE 'revert|hotfix|emergency|rollback'
```
A handful per year is normal. Reverts every couple of weeks mean the team doesn't trust
its deploy process — evidence of unreliable tests, missing staging, or painful rollbacks.
*Invalidated by:* same discipline caveat — zero results can mean stable *or* mean nobody
writes descriptive commit messages.

## The cross-reference (the key analytical move)

The single most useful output is the **intersection of probe 1 and probe 3**. Compute it
explicitly:
- `churn ∩ bugs` → highest-risk files. Read these first, and read them looking for *why*
  they keep breaking (missing tests, tangled responsibilities, leaky abstractions).
- High churn, low bugs → active area; read to understand current direction.
- Low churn, high bugs → fragile but stalled; read if it's on your change path.

## Single-file mode

When `<TARGET>` is one file, the ranked-list framing collapses (there's only one file), so
report **that file's own story** instead. Replace the five question-sections with:

- **변경 이력** — how often it churns and when:
  ```bash
  git log --format='%ad' --date=format:'%Y-%m' -- <file> | sort | uniq -c
  ```
- **작성자** — who owns it, and whether the main author is still active:
  ```bash
  git log --format='%an' --no-merges -- <file> | sort | uniq -c | sort -rn
  ```
- **버그 이력** — how much of its history is firefighting:
  ```bash
  git log -i -E --grep="fix|bug|broken" --oneline -- <file>
  ```
- **함께 바뀌는 파일 (coupling)** — the hidden blast radius: files most often committed
  alongside the target. High coupling into unrelated modules is a design smell. Bound it
  with `--since` so a long history doesn't blow up time/cost:
  ```bash
  for c in $(git log --format=%H --since="1 year ago" -- <file>); do \
    git show --name-only --format='' "$c"; done \
    | grep -v '^<file>$' | sort | uniq -c | sort -rn | head -20
  ```

Then give the same **먼저 읽을 파일** recommendation, led by the most-coupled files (they're
what you'll actually have to touch alongside the target).

## Output format — the report

Produce the report **in Korean**, with these exact section headings, in this order. The
first five mirror the five diagnostic questions; the last three synthesize them into
something actionable. Keep each of the five question-sections to a few lines — this is
orientation, not a full audit.

```
# 코드베이스 정찰 — <repo 이름>
_기간: <예: 최근 1년> · 범위: <소스 경로 / 패키지> · 분석 커밋 수: <n>_

## 가장 자주 바뀌는 파일
<churn 상위 파일을 짧은 순위 목록으로 (횟수 — 경로). 눈에 띄는 파일은 활발한 개발인지
소유자 없는 drag인지 구분해 적고, 가장 많이 바뀐 단일 파일을 명시한다.>

## 누가 만들었는가
<기여자 순위. bus factor를 명시한다 (예: "최상위 작성자 = 전체 커밋의 64%"). 전체 기간과
최근 6개월을 비교하고, 초기 개발자가 더 이상 보이지 않으면 강조한다.>

## 버그가 몰리는 곳
<fix/bug 커밋이 가장 자주 건드린 파일을 짧은 순위 목록으로. 그것이 시사하는 바.>

## 성장 중인가, 쇠퇴 중인가
<커밋 속도의 형태를 말로 — 꾸준함 / 하강 / 스파이크 — 눈에 띄게 꺾이는 달을 짚는다.
이것은 코드 데이터가 아니라 팀 데이터다.>

## 얼마나 자주 긴급 대응하는가
<기간 내 revert/hotfix 빈도 + 배포 신뢰도에 대해 시사하는 바.>

## 최고위험 파일 (churn ∩ 버그)
<"가장 자주 바뀌는 파일"과 "버그가 몰리는 곳"의 교집합 — 자주 바뀌면서 동시에 계속 버그
수정을 받는 파일. 이 보고서의 핵심 결론.>

## 먼저 읽을 파일
1. <경로> — <태그: churn ∩ 버그 / 높은 churn / 높은 버그> — <열었을 때 무엇을 볼지>
2. ...

## 주의사항 (신호 신뢰도)
<약하거나 무효인 신호: shallow clone, squash-merge로 인한 authorship 왜곡, 부실한 커밋
메시지, 모노레포 집계. 신뢰도가 깎인 신호를 사실처럼 제시하지 말 것.>
```

After producing the report, proceed to read the "먼저 읽을 파일" list (that is the point of
the recon) unless the user only asked for the diagnosis.

## Adapting beyond the defaults

- **Non-git VCS.** The methodology is VCS-agnostic; the commands aren't. For Mercurial use
  `hg churn` / `hg log`; for `jj` use `jj log`; for Subversion fall back to `svn log -v`
  plus scripting. Keep the same five questions and the cross-reference.
- **Windows / no Unix shell.** The pipes assume a POSIX shell. In PowerShell, replace
  `sort | uniq -c | sort -rn` with `Group-Object | Sort-Object Count -Descending`, or just
  run inside Git Bash / WSL.
- **Very young repos.** With weeks of history, the probes return little. That thinness is
  itself a signal (no legacy yet) — say so plainly rather than over-interpreting noise.
- **Tuning windows.** `--since="1 year ago"` and `head -20` are starting points. Widen for
  slow-moving repos, narrow for very active ones. State the window you used in the report.
