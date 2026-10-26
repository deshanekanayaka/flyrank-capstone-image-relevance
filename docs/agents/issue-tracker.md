# Issue tracker: GitHub

The issues and the specifications for this repository are GitHub issues. Use the `gh` CLI
for every operation.

`gh` infers the repository from `git remote -v`. Inside a clone, `gh` does this without a
flag.

## Conventions

| Operation | Command |
|---|---|
| Create an issue | `gh issue create --title "..." --body "..."`. Use a heredoc for a body of more than one line. |
| Read an issue | `gh issue view <number> --comments`. Filter the comments with `jq`, and fetch the labels as well. |
| List issues | `gh issue list --state open --json number,title,body,labels,comments --jq '[.[] \| {number, title, body, labels: [.labels[].name], comments: [.comments[].body]}]'`. Add the `--label` and `--state` filters that you need. |
| Comment on an issue | `gh issue comment <number> --body "..."` |
| Apply a label | `gh issue edit <number> --add-label "..."` |
| Remove a label | `gh issue edit <number> --remove-label "..."` |
| Close an issue | `gh issue close <number> --comment "..."` |

## Pull requests as a triage surface

Pull requests as a request surface: no.

If this repository treats an external pull request as a feature request, set that flag to
yes. The `/triage` skill reads the flag.

If the flag is yes, pull requests go through the same labels and states as issues, with
the `gh pr` commands:

| Operation | Command |
|---|---|
| Read a pull request | `gh pr view <number> --comments`, and `gh pr diff <number>` for the diff. |
| List external pull requests for triage | `gh pr list --state open --json number,title,body,labels,author,authorAssociation,comments`. Then keep only an `authorAssociation` of `CONTRIBUTOR`, `FIRST_TIME_CONTRIBUTOR`, or `NONE`. Drop `OWNER`, `MEMBER`, and `COLLABORATOR`. |
| Comment, label, or close | `gh pr comment`, `gh pr edit --add-label`, `gh pr edit --remove-label`, `gh pr close` |

GitHub shares one number space across issues and pull requests, so a bare `#42` can be
either one. To resolve it, operate `gh pr view 42` first, then `gh issue view 42`.

## When a skill says "publish to the issue tracker"

Create a GitHub issue.

## When a skill says "fetch the relevant ticket"

Operate `gh issue view <number> --comments`.

## Wayfinding operations

The `/wayfinder` skill uses these operations. The map is a single issue. The tickets are
child issues of that map.

| Term | Instruction |
|---|---|
| Map | A single issue with the label `wayfinder:map`. It holds the Notes, the Decisions-so-far, and the Fog in its body. Create it with `gh issue create --label wayfinder:map`. |
| Child ticket | An issue linked to the map as a GitHub sub-issue. Use `gh api` on the sub-issues endpoint. If sub-issues are not enabled, add the child to a task list in the map body. Then put `Part of #<map>` at the top of the child body. The labels are `wayfinder:<type>`, where the type is `research`, `prototype`, `grilling`, or `task`. After a claim, the ticket is assigned to the developer who drives it. |
| Blocking | Use the native issue dependencies of GitHub. They are the canonical representation, and the UI shows them. Add an edge with `gh api --method POST repos/<owner>/<repo>/issues/<child>/dependencies/blocked_by -F issue_id=<blocker-db-id>`. The `<blocker-db-id>` is the numeric database id of the blocker, from `gh api repos/<owner>/<repo>/issues/<n> --jq .id`. It is not the `#number` and not the `node_id`. GitHub reports `issue_dependencies_summary.blocked_by`, which counts open blockers only and is the live gate. If dependencies are not available, put a `Blocked by: #<n>, #<n>` line at the top of the child body. When every blocker is closed, the ticket is unblocked. |
| Frontier query | List the open children of the map with `gh issue list --state open`. Scope the list to the sub-issues or the task list of the map. Drop every child with an open blocker, which is `issue_dependencies_summary.blocked_by > 0` or an open issue in the `Blocked by` line. Drop every child with an assignee. The first child in map order wins. |
| Claim | `gh issue edit <n> --add-assignee @me`. This is the first write of the session. |
| Resolve | `gh issue comment <n> --body "<answer>"`, then `gh issue close <n>`. Then append a context pointer, which is a gist and a link, to the Decisions-so-far of the map. |
