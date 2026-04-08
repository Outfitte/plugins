---
name: gh-cli
description: Use this skill for any GitHub-related task using the gh CLI tool. Trigger when users ask to search for repos, list issues, view pull requests, check workflow runs, clone repos, create branches, manage releases, view file contents on GitHub, check repo stats, find repos by a user or org, or do anything involving GitHub. The gh CLI is already installed and authenticated. Use this skill whenever GitHub, repos, issues, PRs, Actions, releases, or any GitHub-related concept comes up.
---

# GitHub CLI (gh) Skill

The `gh` CLI is installed and authenticated. The active account is `maxtreaming`.

## Core Commands

### Repos
```bash
# Search repos by user/org
gh repo list <user-or-org> --limit 30

# Search repos globally
gh search repos <query> --limit 20

# View a repo
gh repo view <owner>/<repo>

# Clone a repo
gh repo clone <owner>/<repo>
```

### Issues
```bash
gh issue list --repo <owner>/<repo>
gh issue view <number> --repo <owner>/<repo>
gh issue create --repo <owner>/<repo> --title "..." --body "..."
```

### Pull Requests
```bash
gh pr list --repo <owner>/<repo>
gh pr view <number> --repo <owner>/<repo>
gh pr create --title "..." --body "..."
gh pr merge <number>
```

### Workflow Runs (Actions)
```bash
gh run list --repo <owner>/<repo>
gh run view <run-id> --repo <owner>/<repo>
gh run watch <run-id>
```

### Releases
```bash
gh release list --repo <owner>/<repo>
gh release view <tag> --repo <owner>/<repo>
gh release create <tag> --title "..." --notes "..."
```

### File Contents
```bash
gh api repos/<owner>/<repo>/contents/<path> --jq '.content' | base64 -d
```

### Milestones
> `gh milestone` does NOT exist. Use `gh api` for all milestone operations.

```bash
# List milestones
gh api repos/<owner>/<repo>/milestones --jq '.[] | {number: .number, title: .title}'

# Create a milestone
gh api repos/<owner>/<repo>/milestones -X POST -f title="M1 — ..." -f state="open"

# Assign a milestone to an issue (use milestone number, not title)
gh api repos/<owner>/<repo>/issues/<number> -X PATCH -f milestone=<milestone_number>
```

### Sub-Issues
> The REST endpoint `POST /issues/{number}/sub_issues` returns 404 and does NOT work. Use the GraphQL `addSubIssue` mutation instead.

```bash
# Step 1: get the node IDs for both the parent and child issues
gh api graphql -f query='{ repository(owner: "<owner>", name: "<repo>") { issue(number: <parent>) { id } } }'
gh api graphql -f query='{ repository(owner: "<owner>", name: "<repo>") { issue(number: <child>) { id } } }'

# Step 2: link child as sub-issue of parent
gh api graphql -f query='
mutation {
  addSubIssue(input: { issueId: "<parent-node-id>", subIssueId: "<child-node-id>" }) {
    issue { number }
    subIssue { number }
  }
}'

# List existing sub-issues (REST GET works fine)
gh api repos/<owner>/<repo>/issues/<number>/sub_issues --jq '[.[] | .number]'
```

### Gists
```bash
gh gist list
gh gist view <id>
gh gist create <file>
```

## Tips

- Always use `--json` flag with `--jq` for structured output when parsing results programmatically.
- Use `gh api` for any GitHub API endpoint not covered by higher-level commands.
- For pagination, use `--limit` or `--paginate` flags.
- Use `gh search repos --owner <user>` to filter by owner when searching.
- When listing a user's repos, `gh repo list <user>` is faster than `gh search repos`.

## Common Patterns

**Find all repos from a user:**
```bash
gh repo list maxtreaming --limit 100 --json name,description,url,pushedAt \
  --jq '.[] | "\(.name) - \(.description // "no description")"'
```

**Search repos with a keyword:**
```bash
gh search repos "topic:python user:maxtreaming" --limit 20
```

**Get open PRs across a repo:**
```bash
gh pr list --repo <owner>/<repo> --state open --json number,title,author,createdAt
```

**Check latest workflow status:**
```bash
gh run list --repo <owner>/<repo> --limit 5 --json status,conclusion,name,createdAt
```
