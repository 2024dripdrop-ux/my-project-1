# Issue tracker: GitHub

Issues and specs for this repo live as GitHub issues in `2024dripdrop-ux/my-project-1`. Use the `gh` CLI for all operations.

> **Use the REST API (`gh api`), not `gh issue …` / `gh pr …`.** Those porcelain commands go through GitHub GraphQL, which is blocked in Claude Code cloud sessions (HTTP 403). Every command below is REST and works there as well as locally.

`R=repos/2024dripdrop-ux/my-project-1` in the commands below.

## Conventions

- **Create an issue**: `gh api $R/issues -f title="..." -f body="$(cat <<'EOF' … EOF)"`. Add labels with `-f 'labels[]=needs-triage'` (repeat per label).
- **Read an issue**: `gh api $R/issues/<n> --jq '{number, title, state, body, labels: [.labels[].name]}'` plus comments: `gh api $R/issues/<n>/comments --jq '[.[] | {user: .user.login, body}]'`.
- **List issues**: `gh api "$R/issues?state=open&per_page=100" --jq '[.[] | select(.pull_request | not) | {number, title, body, labels: [.labels[].name]}]'`. Filter with `&labels=<a>,<b>` and `&state=open|closed|all`. The REST issues endpoint also returns PRs; the `select(.pull_request | not)` drops them.
- **Comment on an issue**: `gh api $R/issues/<n>/comments -f body="..."`
- **Apply / remove labels**: `gh api $R/issues/<n>/labels -f 'labels[]=<label>'` / `gh api -X DELETE $R/issues/<n>/labels/<label>`. If a label doesn't exist yet, create it first: `gh api $R/labels -f name=<label> -f color=ededed`.
- **Close**: comment, then `gh api -X PATCH $R/issues/<n> -f state=closed` (add `-f state_reason=not_planned` for wontfix).
- **Assign**: `gh api $R/issues/<n>/assignees -f "assignees[]=$(gh api user --jq .login)"` (REST has no `@me`).

## Pull requests as a triage surface

**PRs as a request surface: no.** _(Set to `yes` if this repo treats external PRs as feature requests; `/triage` reads this flag.)_

When set to `yes`, PRs run through the same labels and states as issues, using REST equivalents:

- **Read a PR**: `gh api $R/pulls/<n>` and comments via `gh api $R/issues/<n>/comments`; diff via `gh api $R/pulls/<n> -H "Accept: application/vnd.github.diff"`.
- **List external PRs for triage**: `gh api "$R/pulls?state=open&per_page=100" --jq '[.[] | {number, title, body, labels: [.labels[].name], author: .user.login, authorAssociation: .author_association}]'` then keep only `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE` (drop `OWNER`/`MEMBER`/`COLLABORATOR`).
- **Comment / label / close**: same `issues/<n>/comments` and `issues/<n>/labels` endpoints as issues; close with `gh api -X PATCH $R/pulls/<n> -f state=closed`.

GitHub shares one number space across issues and PRs, so a bare `#42` may be either: `gh api $R/issues/42 --jq '.pull_request != null'` tells you which.

## When a skill says "publish to the issue tracker"

Create a GitHub issue (see "Create an issue" above).

## When a skill says "fetch the relevant ticket"

Read the issue and its comments (see "Read an issue" above).

## Wayfinding operations

Used by `/wayfinder`. The **map** is a single issue with **child** issues as tickets.

- **Map**: a single issue labelled `wayfinder:map`, holding the Notes / Decisions-so-far / Fog body. `gh api $R/issues -f title="..." -f body="..." -f 'labels[]=wayfinder:map'`.
- **Child ticket**: an issue linked to the map as a GitHub sub-issue: `gh api -X POST $R/issues/<map>/sub_issues -F sub_issue_id=<child-db-id>`, where `<child-db-id>` is `gh api $R/issues/<child> --jq .id`. Where sub-issues aren't available, add the child to a task list in the map body and put `Part of #<map>` at the top of the child body. Labels: `wayfinder:<type>` (`research`/`prototype`/`grilling`/`task`). Once claimed, the ticket is assigned to the driving dev.
- **Blocking**: GitHub's **native issue dependencies**, the canonical, UI-visible representation. Add an edge with `gh api --method POST $R/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`, where `<blocker-db-id>` is the blocker's numeric **database id** (`gh api $R/issues/<n> --jq .id`, _not_ the `#number` or `node_id`). GitHub reports `issue_dependencies_summary.blocked_by` (open blockers only, the live gate). Where dependencies aren't available, fall back to a `Blocked by: #<n>, #<n>` line at the top of the child body. A ticket is unblocked when every blocker is closed.
- **Frontier query**: list the map's open children (`gh api $R/issues/<map>/sub_issues --jq '[.[] | select(.state=="open")]'`, or the map's task list), drop any with an open blocker (`issue_dependencies_summary.blocked_by > 0`, or an open issue in the `Blocked by` line) or an assignee; first in map order wins.
- **Claim**: assign yourself (see "Assign" above), the session's first write.
- **Resolve**: comment with the answer, then close the issue, then append a context pointer (gist + link) to the map's Decisions-so-far.
