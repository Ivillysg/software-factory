# Reviewer protocol

Shared by the factory reviewers. You review independently: read only the job files and the code, never the other reviewers' reports of the current round.

## Inputs

- `project.md`: conventions and commands (path given by the orchestrator).
- `<job dir>/input.md`, `spec.md`, `build.md`.
- The diff: `git diff <base>...<branch>` and `git log <base>..<branch>`. Read full files where the diff lacks context.
- Round 2 and later: your own previous report plus `feedback-<n-1>.md` (and any `rework-<k>.md`).

## Severity

- **blocker**: wrong behavior, data loss, exploitable flaw, broken AC, broken build.
- **major**: real defect or convention breach a maintainer would refuse to merge.
- **minor** / **nit**: improvement that does not block.

Verdict is `CHANGES` when at least one blocker or major exists, `PASS` otherwise. Your area has nothing to review (e.g. no UI in the change): `PASS` with a one-line `N/A:` reason.

## Report

Write only your output path. The first line is exactly `PASS` or `CHANGES`, nothing else on it.

```markdown
PASS
# <Area> review: <job-id>, round <n>
Range: <base>...<branch> @ <head sha>

## Findings
| # | Severity | Location | Issue | Required fix |
|---|---|---|---|---|
| 1 | major | src/a.ts:42 | ... | ... |

## Previous findings
Round 2+: each earlier finding of yours as Fixed / Not fixed, with evidence.

## Notes
Scope reviewed, what you ran, N/A reason.
```

Any server, watcher or background process you start, you stop before replying, process tree included; never kill one you did not start.

Every finding cites a concrete location and evidence you observed. The code and git history stay untouched: you read and run read-only commands, the builder fixes.

Reply to the orchestrator with one line: verdict plus count of blockers/majors.
