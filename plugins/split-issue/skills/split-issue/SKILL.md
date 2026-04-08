---
name: split-issue
description: Split a GitHub issue into sub-issues, one per method, endpoint, handler, or logical unit. Use this skill whenever the user asks to break down an issue, create sub-issues, split a task into sub-tasks, or decompose a GitHub issue by method/endpoint/handler. Trigger on phrases like "split issue", "create sub-issues", "break down issue", "sub-issues per method", "add sub-issues to", or any request to decompose an issue into smaller tracked pieces.
---

# Split Issue into Sub-Issues

> Before using any `gh` commands, read the `gh-cli` skill — it has the full command reference and the critical note on how sub-issue linking works.

## Workflow

### 1. Read the parent issue

```bash
gh issue view <number> --repo <owner>/<repo>
```

Identify the list of items to split by. Common patterns:
- HTTP handlers / endpoints (`POST /items`, `GET /items/{id}`, …)
- Service methods
- Repository methods
- Test scenarios

If the split criteria aren't obvious from the issue body, ask the user before proceeding.

### 2. Create one sub-issue per item

For each item, create an issue with:
- **Title**: `<parent-label>: <verb> <item>` — e.g. `M1-015a: Implement POST /items handler`
- **Body**: include `Part of #<parent>` on the first line, then a short description of the specific work

```bash
gh issue create --repo <owner>/<repo> \
  --title "..." \
  --body "$(cat <<'EOF'
Part of #<parent>

<description>
EOF
)"
```

Note the issue number returned by each create call — you'll need them all for linking.

### 3. Link each sub-issue to the parent via GraphQL

The REST endpoint for sub-issues does not work. Use the GraphQL `addSubIssue` mutation.

**Step 3a — get node IDs** (fetch all at once to save time):

```bash
gh api graphql -f query='{
  repository(owner: "<owner>", name: "<repo>") {
    parent: issue(number: <parent>) { id }
    s1: issue(number: <sub1>) { id }
    s2: issue(number: <sub2>) { id }
  }
}'
```

**Step 3b — link each sub-issue**:

```bash
gh api graphql -f query='
mutation {
  addSubIssue(input: { issueId: "<parent-node-id>", subIssueId: "<child-node-id>" }) {
    issue { number }
    subIssue { number }
  }
}'
```

Run one mutation per sub-issue. They can be done sequentially in a shell loop if there are many.

### 4. Verify

```bash
gh api repos/<owner>/<repo>/issues/<parent>/sub_issues --jq '[.[] | .number]'
```

Confirm all expected sub-issue numbers appear in the output.
