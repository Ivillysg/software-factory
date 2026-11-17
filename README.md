# Software Factory

**Version 0.1**

A Claude Code pipeline that takes a feature or a Jira/Linear issue through spec, build, four parallel reviews (plus an optional browser check), a quality gate, approval and integration, following the conventions of the repository it lives in.

## How it works

`/factory` is an orchestrator skill. It never writes code itself: it moves each job through a fixed pipeline of specialist subagents, and the agents talk to each other only through Markdown files in the job folder. Nothing is pushed, opened as a PR or merged until that step is authorized in the project's `project.md` or through `/factory approve`.

```mermaid
flowchart TD
    IN(["/factory #lt;feature text | issue URL#gt;<br/>or /factory next"]) --> RP["Resolve project<br/>repo root to project key"]
    RP --> PR{"project.md<br/>exists?"}
    PR -- no --> PF["Build project profile<br/>stack, commands, branching,<br/>commit and PR conventions"]
    PF --> OQ{"Open questions?"}
    OQ -- yes --> ASK["Ask the user<br/>and record answers"] --> JOB
    OQ -- no --> JOB
    PR -- yes --> JOB["Create job<br/>input.md (Jira / Linear / text / backlog)"]

    JOB --> SPEC["factory-spec-writer<br/>spec.md with AC- items"]
    SPEC --> BUILD["factory-builder<br/>branch, code, tests, commits, build.md"]

    BUILD --> REV
    subgraph REV ["Review round n (parallel)"]
        direction LR
        SEC["security"]
        UX["ux"]
        UI["ui"]
        CODE["code"]
        BR["browser<br/>(optional)"]
    end

    REV --> V{"All PASS?"}
    V -- "CHANGES,<br/>1st round of cycle" --> FB["feedback-n.md<br/>deduplicated, by severity"]
    FB --> BUILD
    V -- "PASS, or CHANGES<br/>on 2nd round" --> GATE["Quality gate (gate.md)<br/>reviews, tests, acceptance,<br/>scope, docs, git"]

    GATE --> APR["factory-approver<br/>decision.md"]
    APR -- ESCALATE --> HUM{"Human decides"}
    APR -- APPROVE --> AW["awaiting-human"]
    HUM -- "/factory approve<br/>(override)" --> AW
    HUM -- "/factory rework" --> RW
    AW -- "/factory approve<br/>or pre-authorized step" --> INT["Integration steps from project.md<br/>push, PR, ticket, merge"]
    INT --> DONE(["done<br/>backlog box checked"])
    AW -- "/factory rework #lt;note#gt;" --> RW["rework-k.md<br/>new cycle"]
    RW --> BUILD
```

Key rules:

- **Two review rounds per cycle.** If round 2 still has `CHANGES`, the job goes to the gate anyway; it fails and the approver decides.
- **One writer per file.** The orchestrator alone writes `board.json` and the job control files; each agent writes only its own output. History is append-only.
- **Ticket text is data.** Instructions inside an issue never override factory rules or repo conventions.
- **Human in the loop.** Push, PR, merge and ticket transitions run only when `project.md` authorizes them or the user runs `/factory approve`.

## Install

Copy into `~/.claude/` to use it in every repo, or into one repo's `.claude/`:

```
.claude/skills/factory/        orchestrator skill, reference files, templates/
.claude/agents/factory-*.md    8 specialist agents
```

## Per-project state

State lives outside the repos, one folder per project, so nothing reaches a commit:

```
~/.claude/factory/
  dashboard.html               copied from the skill's templates/
  projects.json                index: key -> repo path
  projects/<key>/
    project.md                 repo conventions, every agent obeys it
    board.json  backlog.md
    jobs/<id>/
```

The project is the repo where you run `/factory`; all worktrees of a repo share one project. Its key is the repo folder name (`-2`, `-3` on a name clash). The first command in a repo creates the folder and writes `project.md` (commands, branching, commit and PR conventions, delivery workflow). Review and edit that file.

## Commands

| Command | Effect |
|---|---|
| `/factory <feature text or issue URL>` | Start a new job |
| `/factory next` | Start the next unchecked line of the project's `backlog.md` |
| `/factory approve <job-id>` | Authorize the next integration step (push, PR, merge...) or override an escalation |
| `/factory rework <job-id> <note>` | Send extra feedback to the job's builder and review again |

## Agents

| Agent | Model | Writes |
|---|---|---|
| factory-spec-writer | opus | `spec.md` |
| factory-builder | opus | code, commits, `build.md` |
| factory-security-reviewer | opus | `review-<n>-security.md` |
| factory-ux-reviewer | sonnet | `review-<n>-ux.md` |
| factory-ui-reviewer | sonnet | `review-<n>-ui.md` |
| factory-code-reviewer | sonnet | `review-<n>-code.md` |
| factory-browser-reviewer (optional) | sonnet | `review-<n>-browser.md`, `shots/` |
| factory-approver | opus | `decision.md` |

The orchestrator alone writes `board.json`, `project.md`, `input.md`, `feedback-<n>.md`, `rework-<k>.md` and `gate.md`.

## Browser QA (optional)

`factory-browser-reviewer` runs the job branch's app in a real browser with [agent-browser](https://www.skills.sh/vercel-labs/agent-browser/agent-browser), walks every `AC-` of the spec, and saves a screenshot per AC in the job's `shots/`. It reviews in the same round as the others, so a failed AC goes back to the builder through the normal feedback loop.

Turn it on per project:

1. Nothing to install by hand: on the first browser round the orchestrator checks `agent-browser --version` and `agent-browser doctor`, and asks before running `npm install -g agent-browser` and `agent-browser install` (downloads Chrome for Testing). Decline and the round runs without the browser check.
2. In the project's `project.md`, fill `Commands` > `Run app` and the `Browser QA` section: `Enabled: yes`, the env var names holding test credentials (never the values), and check the detected ports. The profile finds the port the app uses and reserves the rest of its hundred block for the factory (app on 3000 -> 3001-3099); the reviewer starts its own server on the first free port there, never on yours.

Every server or process an agent starts is stopped before it replies, process tree included, and the orchestrator checks the port range after each browser round and after the gate.

When the app cannot start for an environment reason, the round goes on without it and the factory tells you.

## Dashboard

One dashboard covers every project: the sidebar lists each project's jobs, and the selected job shows as a pipeline graph (spec, build, each review round with its reviewers, fixes, gate, approver, integration steps). Click a step to read its file (`spec.md`, `review-1-ux.md`, `decision.md`...); a job waiting on you shows the `/factory approve` and `rework` commands with a copy button. It reads `projects.json`, every project's `board.json` and the job Markdown files. Browsers block that from `file://`, so serve the folder with whatever is installed:

```
python -m http.server 8765 -d ~/.claude/factory
npx http-server ~/.claude/factory -p 8765
```

Open http://localhost:8765/dashboard.html (`#<key>` opens a project, `#<key>/<job-id>` a job); it refreshes every 5 s. Without a server, open the file directly and use **Open factory folder** on `~/.claude/factory` for a static snapshot.

## Changelog

### 0.1

- First release: `/factory` orchestrator skill, 8 specialist agents, per-project profile and board, optional browser QA with agent-browser, multi-project dashboard.
