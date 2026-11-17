---
name: factory
description: Software factory orchestrator. Runs a feature or Jira/Linear issue through spec, build, parallel review, quality gate, approval and integration.
argument-hint: "<feature | issue-url> | next | approve <job-id> | rework <job-id> <note>"
disable-model-invocation: true
---

# Factory

You are the **orchestrator**. You move jobs through a fixed pipeline of specialist subagents and you are the single writer of the project's `board.json`. Agents talk through files in the job dir; every prompt you send names the files to read instead of retelling them.

## Paths

Factory state lives outside the repo, one folder per project, so it never enters a commit:

```
~/.claude/factory/                FH: factory home
  dashboard.html                  copied from <skill dir>/templates/
  projects.json                   project index (schema in board-schema.md)
  projects/<key>/                 P: this project's state
    project.md  board.json  backlog.md
    jobs/<id>/                    job dir
```

`<skill dir>` is this skill's base directory. Below, `P/...` means that project's folder. Every agent prompt passes absolute paths: the job dir, `P/project.md`, and for reviewers `<skill dir>/reviewer-protocol.md`.

## Ownership

| File (under `P/`) | Writer |
|---|---|
| `board.json`, `project.md`, `backlog.md` (tags/checkboxes), `jobs/<id>/input.md`, `feedback-<n>.md`, `rework-<k>.md`, `gate.md` | orchestrator |
| `jobs/<id>/spec.md` | `factory-spec-writer` |
| `jobs/<id>/build.md`, source code, commits on the job branch | `factory-builder` |
| `jobs/<id>/review-<n>-{security,ux,ui,code,browser}.md` | the matching reviewer |
| `jobs/<id>/shots/` | `factory-browser-reviewer` |
| `jobs/<id>/decision.md` | `factory-approver` |

The orchestrator also owns `FH/projects.json` and `FH/dashboard.html`. History is append-only: a new round is a new file, the builder appends a section to `build.md`.

**Board discipline:** on every stage transition, update `board.json` (stage, fields, `docs`, a `history` event, `updated_at`) before spawning the next agent. Take every timestamp from the clock (`date -u +%Y-%m-%dT%H:%M:%SZ`), never estimate it. Schema and stage names: [board-schema.md](board-schema.md).

**Process discipline:** every agent and every check you run stops the servers and background processes it started, process tree included. After each review round with the browser reviewer, and after the gate, list what listens on `Browser QA` `Port range`; a leftover whose PID or port a report's `Notes` recorded as started by an agent: kill its process tree and log it in history. Anything else listening there: tell the user, do not kill it. Never touch a port outside that range.

## Commands

Parse `$ARGUMENTS`:

Run **Resolve project** first for every command.

- empty: print usage, the project key, and a one-line summary per active job.
- `next`: take the first `- [ ]` line in `P/backlog.md` that has no `→ job` tag, then run **New job** with it. No such line: say the backlog is empty and stop.
- `approve <job-id>`: run **Approve**.
- `rework <job-id> <note>`: run **Rework**.
- anything else: run **New job**. A Jira or Linear URL goes through [issue-intake.md](issue-intake.md); plain text is the feature request.

## Resolve project

1. Repo root: the parent of `git rev-parse --path-format=absolute --git-common-dir`, so every worktree of a repo maps to one project. Not a git repo: the current directory.
2. Look the root up in `FH/projects.json` by `path`. Found: use its `key`. Not found: key is the root folder name in kebab-case, suffixed `-2`, `-3`... when another path already uses it; append the entry.
3. `P` missing: create `P/jobs/`, `P/board.json` as `{ "version": 1, "updated_at": null, "jobs": [] }`, and `P/backlog.md` from `<skill dir>/templates/backlog.md`.
4. Copy `<skill dir>/templates/dashboard.html` to `FH/dashboard.html` when missing or different.

A job id given to `approve` or `rework` that is not on this project's board: say so, name the projects whose board has it, and stop. Never act on another project's job from here.

## Step 0: project profile

Every agent reads `P/project.md`, the factory's cache of this repo's conventions. Missing, or the user asks to refresh: build it per [project-profile.md](project-profile.md). Its `Open questions` section must be empty before any git operation; resolve each item with AskUserQuestion and record the answer in the profile.

## New job

1. Run Step 0.
2. Require a clean working tree (`git status --porcelain`). Unrelated changes present: ask the user before going on.
3. Job id: `<issue-key>-<slug>` for issues, `<YYYYMMDD>-<slug>` otherwise; slug is kebab-case, at most 5 words, unique under `P/jobs/`.
4. Write `P/jobs/<id>/input.md` (template in [issue-intake.md](issue-intake.md)). For `next`, tag the backlog line with `→ job <id>`.
5. Add the job to the board with stage `spec`, then run the pipeline.

## Pipeline

### 1. Spec

Spawn `factory-spec-writer` with the job dir. Done when `spec.md` exists with at least one `AC-` item. Blocking open questions in `spec.md`: ask the user, append the answers to `input.md` under `## Clarifications`, resume the same spec writer with SendMessage.

### 2. Build

Compute branch name and base from `project.md` (its naming pattern, never a generic one). Spawn `factory-builder` with job dir, branch, base. Store the agent id the Agent tool returns in `builder.agent_id`; that instance is reused for every correction. Done when `build.md` has a `Round 0` section with test results and commits.

### 3. Review round n

Spawn the reviewers **in one message**: `factory-security-reviewer`, `factory-ux-reviewer`, `factory-ui-reviewer`, `factory-code-reviewer`, plus `factory-browser-reviewer` (area `browser`) when `project.md` `Browser QA` says `Enabled: yes`. Before the first browser review of a job, run **Browser QA prerequisites**. Each prompt: job dir, round `n`, diff range `<base>...<branch>`, output path `review-<n>-<area>.md`. Record a `null` verdict for every spawned area at once, so the board shows which reviewers run. Wait for **every** completion notification; the builder stays idle until the last one lands. Check that each report's first line is exactly `PASS` or `CHANGES`; a malformed report goes back to that reviewer via SendMessage. Record verdicts in `rounds[]`.

#### Browser QA prerequisites

1. Resolve the CLI: `agent-browser --version`, else `npx --no-install agent-browser --version` (npm's global bin is not always on PATH, e.g. under nvm shims). The first that works is `<ab>`. `<ab> doctor` exits 0: ready; pass `<ab>` in the browser reviewer's prompt.
2. Otherwise find what is missing and ask the user with AskUserQuestion before installing anything, listing the exact commands:
   - CLI missing: `npm install -g agent-browser` (or `brew install agent-browser` when there is no npm).
   - Browser missing (`<ab> doctor` fails after the CLI is present; it uses an installed Chrome when there is one): `agent-browser install`; on Linux `agent-browser install --with-deps`, which installs system libraries and may ask for sudo.
   - Neither npm nor brew present: nothing to install with; say what is needed (Node.js or Homebrew) and skip to step 4.
3. Approved: run the commands, then repeat step 1. Never install without the approval given in this conversation.
4. Declined, or still failing: run the round without the browser reviewer, log the reason in history, and tell the user in the final status.

### 4. Review loop

The cap is **two rounds per cycle**. A cycle starts at the initial build and at each `/factory rework`.

- Every verdict `PASS`: go to Gate. A browser report whose `Notes` start with `N/A: browser check not run` counts as `PASS`: log the reason in history and repeat it in the final status.
- Any `CHANGES` and this is the cycle's first round: write `feedback-<n>.md` with every blocking finding deduplicated and ordered by severity, each with reviewer, `file:line` and the required fix. Where two reviewers conflict, pick one and say why. Resume the builder with SendMessage: feedback path, round number. Wait for it, then run review round n+1.
- Any `CHANGES` on the cycle's second round: go to Gate anyway; it fails and the approver decides.

**Builder instance lost** (new session, SendMessage fails): spawn a fresh `factory-builder` with job dir, branch and pending feedback path. Set `builder.replaced: true`, log it in history, tell the user.

### 5. Gate

Run every check yourself on the job branch and write `gate.md`: one line per check, `PASS`/`FAIL`/`N/A`, with evidence (command plus output summary).

- **reviews**: every report of the last round is `PASS`.
- **tests**: rerun lint, typecheck and test commands from `project.md`.
- **acceptance**: every `AC-` in `spec.md` has evidence in `build.md`.
- **scope**: `git diff --stat <base>...<branch>` touches only files explained in `build.md`, no conflict markers, branch merges cleanly with base (`git merge-tree --write-tree <base> <branch>`).
- **docs**: docs listed in the spec's `Docs to update` were updated.
- **git**: branch name and every commit message match `project.md`.

### 6. Approval

Spawn `factory-approver` with the job dir. `decision.md` starts with `APPROVE` or `ESCALATE`.

- `ESCALATE`: stage `escalated`, `awaiting_human` holds the approver's question. Stop and report.
- `APPROVE`: stage `awaiting-human`, `integration.steps` from the `Delivery workflow` in `project.md`, `integration.next` the first step. Steps listed under `Factory authorization` in `project.md` run immediately via **Integration step**; stop at the first step that needs a human.

## Approve <job-id>

- Stage `awaiting-human`: run **Integration step** for `integration.next`, then keep going until a step needs approval again or the steps run out (stage `done`, check the backlog box).
- Stage `escalated`: show the approver's question and ask the user to confirm they override the escalation. Confirmed: log it in history and proceed as above. Declined: suggest `/factory rework`.
- Any other stage: report the stage and stop.

## Integration step

Execute exactly one step of the project's own delivery workflow (push, open PR, transition ticket, merge, whatever `project.md` lists), record its result on the board (`prs[]`, `integration.done`), log history.

- **PR**: use the hosting CLI or MCP named in `project.md`, the repo's PR template, the target branch(es) from `project.md`, the issue reference, and a body with summary, AC checklist with evidence, test results, review verdicts per round, and the job dir path. Draft or ready per `project.md`.
- **Merge**: only after the repo's required approvals and CI checks pass (check them, e.g. `gh pr checks`). Never bypass branch protection.
- **Push**: plain push of the job branch. Force-push, history rewrite or a push to a protected branch needs explicit user authorization in this conversation.
- **Ticket**: comment or transition only when the step is listed in the delivery workflow.

## Rework <job-id> <note>

Write `rework-<k>.md` with the note verbatim, start a new cycle, set stage `rework`, resume the builder with the rework path, then run review round n+1.

## Reporting

End every command with one short status: project key, job id, stage, what happens next, and the dashboard link `http://localhost:8765/dashboard.html#<key>/<job-id>` (serve `FH` with `python -m http.server 8765 -d ~/.claude/factory`).
