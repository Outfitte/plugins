---
name: create-issues
description: Create GitHub issues in bulk from a structured JSON task file and link them to an existing milestone. Use this skill whenever the user wants to create issues from a tasks JSON file, populate a GitHub milestone with issues, bulk-create issues from a planning file, or sync a task list to GitHub. Trigger on phrases like "create issues from", "create issues on github", "populate the milestone", "create GitHub issues", "sync tasks to GitHub", or any time a .json tasks file is mentioned alongside GitHub issues or milestones.
---

# Create GitHub Issues from Task File

This skill creates GitHub issues from a structured JSON task file and links them all to an existing milestone. Use Python subprocesses to handle special characters safely — never shell-interpolate the task descriptions directly.

## Input

The user either:
- Invokes `/create-issues` with no args → auto-detect the tasks file
- Provides a path: `/create-issues path/to/tasks.json`

## Expected JSON structure

```json
{
  "milestone": "F1",
  "repo": "owner/repo",
  "tasks": [
    {
      "id": "F1-001",
      "title": "Task title",
      "description": "Full task description (may contain backticks, newlines, markdown)",
      "area": "scaffold",
      "status": "todo",
      "depends_on": []
    }
  ]
}
```

Other fields (e.g. `methodology`, `description` at root level) may be present — ignore them.

## Step 1: Find the tasks file

If no path given, glob for `.claude/tasks/*.json` from the current working directory. If multiple files match, list them and ask the user which one to use. If exactly one matches, proceed with it.

## Step 2: Parse and validate

Load the JSON. Extract:
- `repo` — the GitHub repo (`owner/repo`)
- `milestone` — the milestone identifier string (e.g. `"F1"`)
- `tasks` — the array of task objects

## Step 3: Find the milestone number

GitHub API uses numeric milestone IDs, not titles. Fetch the repo's milestones and match by title prefix — the `milestone` field in the JSON (e.g. `"F1"`) should match the start of the milestone title on GitHub (e.g. `"F1 — Frontend Foundation"`).

```bash
gh api repos/{repo}/milestones
```

Parse the JSON response and find the milestone whose `title` starts with the `milestone` field from the tasks file. Extract its `number`. If no match is found, list available milestones and ask the user to confirm the name.

## Step 4: Ensure labels exist

Before creating issues, collect all unique `area` values from the tasks and ensure each one exists as a label on the repo. Use the GitHub API to list existing labels, then create any that are missing.

```python
import json, subprocess

# Fetch existing labels
result = subprocess.run(
    ['gh', 'api', f'repos/{repo}/labels', '--paginate'],
    capture_output=True, text=True
)
existing = {l['name'] for l in json.loads(result.stdout)}

# Collect areas needed
areas = {t['area'] for t in tasks if t.get('area')}

# Create missing labels (neutral grey color, no description — user can customize later)
for area in areas:
    if area not in existing:
        subprocess.run(
            ['gh', 'api', f'repos/{repo}/labels',
             '-X', 'POST', '-f', f'name={area}', '-f', 'color=cccccc'],
            capture_output=True, text=True
        )
        print(f"  + Created label: {area}")
    else:
        print(f"  ✓ Label exists: {area}")
```

## Step 5: Create the issues

Use Python subprocess to avoid shell quoting issues with backticks, newlines, and special characters in descriptions. Apply the `area` label when creating each issue:

```python
import time

created = []  # list of (task_id, issue_number)

for i, task in enumerate(tasks):
    title = f"{task['id']}: {task['title']}"
    body = task['description']
    area = task.get('area')

    cmd = ['gh', 'issue', 'create', '--title', title, '--body', body, '--repo', repo]
    if area:
        cmd += ['--label', area]

    result = subprocess.run(cmd, capture_output=True, text=True, timeout=15)
    
    if result.returncode == 0:
        url = result.stdout.strip()
        issue_num = int(url.rstrip('/').split('/')[-1])
        created.append((task['id'], issue_num))
        print(f"[{i+1}/{len(tasks)}] ✓ {task['id']} → {url}")
    else:
        print(f"[{i+1}/{len(tasks)}] ✗ {task['id']}: {result.stderr.strip()}")
    
    time.sleep(0.3)  # avoid rate limiting
```

## Step 6: Link issues to the milestone

After all issues are created, patch each one to set the milestone. Batch API calls — no need to wait between them:

```python
milestone_number = ...  # from Step 3

for task_id, issue_num in created:
    result = subprocess.run(
        ['gh', 'api', f'repos/{repo}/issues/{issue_num}',
         '-X', 'PATCH', '-f', f'milestone={milestone_number}'],
        capture_output=True, text=True
    )
    status = '✓' if result.returncode == 0 else '✗'
    print(f"{status} #{issue_num} ({task_id}) linked to milestone")
```

## Step 7: Report

Print a summary:
- Total tasks in file
- Issues successfully created
- Issues linked to milestone
- Link to the milestone page: `https://github.com/{repo}/milestone/{milestone_number}`

## Notes

- If the repo already has some issues from a previous partial run, there's no built-in deduplication — warn the user if they're re-running on the same file
- The `area` field maps to GitHub labels. Labels are auto-created (grey, no description) if missing — the user can customize colors/descriptions later via the GitHub UI or `gh api`
- `depends_on` is metadata only — GitHub issues don't have native dependency links; ignore unless the user asks you to encode them as text in the issue body
