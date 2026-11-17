# board.json schema

`~/.claude/factory/dashboard.html` renders `projects.json` and each project's `board.json`, so keep the field names exact. Times are ISO 8601 UTC. Unknown values are `null`, never omitted.

## projects.json

Index at `~/.claude/factory/projects.json`. `path` is the absolute repo root with forward slashes; `key` names the folder `projects/<key>/`.

```json
{
  "version": 1,
  "projects": [
    { "key": "acme-web", "name": "acme-web", "path": "C:/Users/me/code/acme-web", "created_at": "2026-11-15T20:00:00Z" }
  ]
}
```

## Stages

`spec`, `build`, `review`, `rework`, `gate`, `approval`, `awaiting-human`, `integrating`, `done`, `escalated`, `failed`.

## Example

```json
{
  "version": 1,
  "updated_at": "2026-11-15T22:00:00Z",
  "jobs": [
    {
      "id": "PROJ-123-csv-export",
      "title": "CSV export on reports page",
      "summary": "Users download the filtered report as CSV.",
      "source": { "type": "jira", "key": "PROJ-123", "url": "https://acme.atlassian.net/browse/PROJ-123" },
      "stage": "review",
      "created_at": "2026-11-15T20:00:00Z",
      "updated_at": "2026-11-15T21:40:00Z",
      "branch": "feature/PROJ-123-csv-export",
      "base": "develop",
      "builder": {
        "agent_id": "a1b2c3",
        "replaced": false,
        "progress": "Round 1 fixes committed",
        "commits": ["4f2a9c1 feat(reports): add CSV export"]
      },
      "cycle": 1,
      "rounds": [
        { "n": 1, "cycle": 1, "verdicts": { "security": "PASS", "ux": "CHANGES", "ui": "PASS", "code": "PASS" } }
      ],
      "tests": { "status": "pass", "command": "npm test", "summary": "142 passed" },
      "gate": {
        "status": "pending",
        "checks": { "reviews": null, "tests": null, "acceptance": null, "scope": null, "docs": null, "git": null }
      },
      "decision": null,
      "integration": { "steps": ["push", "pr:develop"], "done": [], "next": null },
      "prs": [],
      "awaiting_human": null,
      "docs": ["input.md", "spec.md", "build.md", "review-1-security.md", "review-1-ux.md", "review-1-ui.md", "review-1-code.md", "feedback-1.md"],
      "history": [
        { "at": "2026-11-15T20:00:00Z", "stage": "spec", "event": "Job created from PROJ-123" }
      ]
    }
  ]
}
```

## Field notes

- `source.type`: `jira`, `linear`, `text` or `backlog`.
- Verdicts: one key per reviewer spawned that round (`security`, `ux`, `ui`, `code`, and `browser` when Browser QA is on), value `PASS`, `CHANGES` or `null` while running. Check values: `pass`, `fail`, `n/a` or `null`. `tests.status` and `gate.status`: `pass`, `fail` or `pending`.
- `decision`: `APPROVE`, `ESCALATE` or `null`.
- `prs[]`: `{ "url", "target", "status": "draft|open|merged|closed", "checks": "pending|pass|fail" }`.
- `awaiting_human`: one sentence telling the human what is needed, or `null`.
- `docs`: file names inside `projects/<key>/jobs/<id>/`, in creation order.
