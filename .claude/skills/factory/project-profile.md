# Project profile

Build `P/project.md` (paths in SKILL.md): the factory's cache of how this repo works. Every fact cites its source (file path or command). A fact you cannot find is written `unknown`; an unknown that blocks git or integration becomes an `Open questions` item.

## Sources

Read every one that exists:

- Docs: `README*`, `CONTRIBUTING*`, `CLAUDE.md`, `AGENTS.md`, `docs/` contribution pages. The user's global `~/.claude/CLAUDE.md` counts only for sections scoped to this project by name.
- Ports: dev/start scripts in the manifests, `.env*` (`PORT=`), framework configs (`vite.config.*`, `next.config.*`, `angular.json`, `application.yml`...), `docker-compose*`, `Procfile`, launch settings. Note every port the project uses and which one `Run app` serves.
- Manifests and tooling: `package.json`, `pyproject.toml`, `go.mod`, `Cargo.toml`, `pom.xml`, `build.gradle*`, `Makefile`, `justfile`, lockfiles, lint/format/type configs.
- CI: `.github/workflows/`, `.gitlab-ci.yml`, `azure-pipelines.yml`, `bitbucket-pipelines.yml`, `.circleci/`.
- PR and review: `.github/pull_request_template*`, `.github/PULL_REQUEST_TEMPLATE/`, `.gitlab/merge_request_templates/`, `CODEOWNERS`.
- Commit rules: `commitlint*`, `.husky/`, `.pre-commit-config.yaml`, `lefthook.yml`.
- Git: `git remote -v`, `git symbolic-ref refs/remotes/origin/HEAD`, `git branch -a`, `git log --oneline -30` (infer branch and commit patterns from real examples).
- Hosting: `gh repo view`, branch protection (`gh api repos/{owner}/{repo}/branches/{branch}/protection`) or the GitLab equivalent, when the CLI is installed and authenticated. Check which CLIs and MCP servers are present; never assume.

Not a git repo: add an open question asking whether to `git init` or to run without branches and commits.

## Template

```markdown
# Project profile
Generated <ISO date> by the factory orchestrator. Edit freely; the factory reads it on every job.

## Stack
Languages, frameworks, package manager, runtime versions.

## Commands
- Install:
- Lint:
- Typecheck:
- Test (full):
- Test (related/single file):
- Build:
- Run app (for UX/UI checks):

## Architecture
Key directories, layers, patterns, where new code of each kind goes.

## Conventions
Style sources, naming, error handling, test layout, i18n, design system / UI component library.

## Git
- Strategy: (git flow / trunk-based / other, with evidence)
- Feature base branch:
- Branch naming: (pattern plus 2 real examples)
- Commit convention: (rule plus 2 real examples)
- Hooks:

## Pull requests
- Hosting and integration: (gh / glab / MCP server name / none)
- Target branch(es):
- Template:
- Draft or ready:
- Required reviewers / CODEOWNERS:
- Required CI checks:
- Branch protection:
- Merge strategy:

## Browser QA
Optional review by `factory-browser-reviewer` (needs the `agent-browser` CLI). Default `Enabled: no`; the user turns it on.
- Enabled: no
- Default port: (port `Run app` serves, with source; e.g. 3000 from `package.json` `dev`)
- Port range: (the hundred block of the default port minus its first port, the default port and any other port the project uses; 3000 -> 3001-3099). The factory starts its own server only inside this range, so it never collides with the developer's.
- Port override: (how `Run app` takes another port, e.g. `PORT=<port> npm run dev` or `npm run dev -- --port <port>`)
- URL: (with `<port>` as placeholder, e.g. http://localhost:<port>/login)
- Login: (env var names holding test credentials, never the values; `none` when no login)

## Delivery workflow
Ordered integration steps for an approved job, e.g. `push`, `pr:develop`, `pr:main`, `jira:transition Ready to QA`, `merge`. Source of each step.

## Factory authorization
Steps the human pre-authorized the factory to run without `/factory approve`. Default: none.

## Open questions
Each unresolved or conflicting convention. Empty before any git operation.
```
