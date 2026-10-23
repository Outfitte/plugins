---
name: implement-task
description: Use this skill when the user invokes /implement-task or asks to implement a GitHub issue, work on a task/ticket, or start working on an issue by number. Always use this skill when the user says "implement task", "work on issue #N", "/implement-task", or similar.
---

# Implement Task

Full workflow for implementing a GitHub issue from start to opening a PR.
Follow all branch, commit, and PR conventions from CLAUDE.md.

## Step 0 — Detect current repo

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

Use `$REPO` for all `gh` commands throughout this workflow.

## Step 1 — Get the issue number

If the user didn't provide an issue number, ask: "Which issue number should I work on?"

## Step 2 — Read the issue

```bash
gh issue view <number> --repo $REPO
```

Read the full issue body and any linked context before proceeding.

> **Network failures:** For any git or gh CLI command that contacts the network
> (git pull, git push, gh pr create, etc.), retry up to 3 times with a 5-second
> delay between attempts before giving up.

## Step 3 — Sync main

```bash
git checkout main && git pull
```

## Step 4 — Create branch

Use the branch naming convention from CLAUDE.md. Derive the short name (2–4 lowercase hyphenated words) from the issue title.

```bash
git checkout -b <username>/<number>-short-name
```

To get the current git username: `git config user.name` (slugify if needed).

## Step 5 — Implement

Read relevant existing code before writing. Follow the architecture rules in CLAUDE.md. Keep changes minimal and focused on the issue scope.

If the task involves writing or modifying code, load and follow the `tdd` skill **before writing any tests or production code**. The TDD loop must be followed strictly: write one failing test, make it pass with minimal code, then move to the next test. Never write multiple tests upfront, and never implement the full body before each individual test is red.

If the change alters user-visible or API behaviour, grep the e2e specs for what you changed (route paths, UI strings, selectors, response fields, function names) — e.g. `grep -rn "<changed symbol or string>" e2e/ tests/ --include='*.ts' --include='*.js'` — and update every spec that asserts the old behaviour. Those specs must be run in Step 7, not left for CI to find.

## Step 6 — Commit

Stage only the files changed for this issue. Follow the commit message format from CLAUDE.md.

```bash
git add <files>
git commit -m "<number>: one sentence message"
```

## Step 7 — Pre-push gate (mandatory)

CI failures that could have been caught locally waste a CI run. **Do not push — the first push or any later one — until every check below passes locally.** Re-run this whole step before every push, including pushes that only add a fix commit.

### 7a. Run the repo's CI mirror

1. If `package.json` (or the Makefile / task runner) has a `ci:local` script, run it: `npm run ci:local`. Also run `ci:local:full` when the change touches DB / migrations, API handlers / routes, or e2e-covered areas — it adds the integration and e2e suites.
2. Otherwise derive the commands from the CI workflow: read `.github/workflows/*.yml` and run, in the same order, every check step of the PR job(s) (lint, format check, audits, typecheck, tests with coverage, and so on). Skip only steps that cannot run locally (uploads, cache, runner setup). Run e2e / integration steps when the change touches areas they cover.
3. Fix every failure and re-run until the whole gate is green. Never push on a "will fix in CI" basis.

### 7b. Check the PR title

If the repo has commitlint (`commitlint.config.*` or a `commitlint` dependency), lint the exact PR title you will use in Step 9:

```bash
echo "<pr title>" | npx commitlint
```

The commit-message and PR-title format in CLAUDE.md wins over the example in Step 9. Also make sure every commit message passes the commit-msg hook. **Never use `--no-verify`** on commit or push; if a hook fails, fix the cause.

### 7c. Check coverage of the change (before the push)

Skip only if the project has no coverage tooling at all. Coverage applies **exclusively to modified lines/files** — unmodified code is irrelevant to this task.

- If the repo has a `coverage:patch` script, run it (after the coverage run that produces its input; `ci:local` already does both). It lists uncovered changed lines and the patch %, and fails below the target.
- Otherwise run the project's coverage command and check that **every changed source line is covered**:

| Language / toolchain | Coverage command | How to scope to modified files |
|---|---|---|
| Go | `go test -coverprofile=coverage.out ./<pkg>/...` then `go tool cover -func=coverage.out` | Pass only the packages that contain modified files |
| TypeScript / Vitest | `npx vitest run --coverage` | Runs all; filter reported output to modified files |
| TypeScript / Jest | `npx jest --coverage` | Runs all; filter reported output to modified files |
| Python / pytest | `pytest --cov=<module> --cov-report=term-missing` | Pass only the modules that contain modified files |
| Rust | `cargo tarpaulin --out Stdout` | Runs all; filter to modified crates |
| Ruby / RSpec | `bundle exec rspec --format documentation` with SimpleCov | Filter to modified files |

If the project uses a different tool, infer the right command from `package.json`, `Makefile`, `pyproject.toml`, `Cargo.toml`, or similar config files.

Store the coverage output — it will be embedded in the PR body in Step 9.

**If any changed line is uncovered:**

1. Identify every uncovered changed line/function
2. **If it can be covered with additional tests** — add those tests (return to the TDD loop), commit, then re-run this whole step
3. Only after exhausting all feasible tests, note a justification for each remaining uncovered line or function — it goes in the PR body in Step 9

A coverage section without a `## Why coverage is not 100%` justification is **not acceptable** in the PR body when changed code is uncovered.

### 7d. Run the e2e specs you touched

Run the e2e specs you updated or found by grep in Step 5 (and `ci:local:full` where applicable). A behaviour change with a stale e2e spec is a CI failure waiting to happen.

## Step 8 — Pre-push review

Spawn two **foreground** subagents in parallel: a `pr-reviewer` (`subagent_type: "pr-reviewer"`) and a `Code Reviewer` (`subagent_type: "Code Reviewer"`). Both review the local diff — no PR exists yet.

Wait for both agents to complete before proceeding.

### pr-reviewer subagent

Spawn with `subagent_type: "pr-reviewer"`:

```
Review the local changes for issue #<number> ("<issue title>") in the $REPO repo.

Run:
  git log main..HEAD --oneline
  git diff main

## Convention checks (CLAUDE.md)
- Read CLAUDE.md first and check that the branch name and commit messages follow the documented naming conventions
- Check for any architecture or structural rules defined in CLAUDE.md and flag violations

## TDD checks
- Failure/error case tests come before happy-path tests
- New methods start from a stub that immediately fails (evidenced by commit history if available)
- Test names are descriptive and follow the project's naming convention
- Test entities use explicit, readable values (not zero/empty defaults) where identity matters

Report any violations, concerns, or suggestions. If everything looks good, say so.
```

### Code Reviewer subagent

Spawn with `subagent_type: "Code Reviewer"`:

```
Review the local code changes for issue #<number> ("<issue title>") in the $REPO repo.

Fetch the diff using:
  git diff main

Focus on correctness, security, maintainability, and performance. Report any blockers, suggestions, or nits. If everything looks good, say so.
```

### Step 8b — Address blockers

After both reviews complete:

- If either review flags **blockers** (🔴): implement the required fixes, commit, then loop back to Step 7 (re-run the whole pre-push gate, then re-run both reviews).
- If there are only suggestions or nits (🟡 / 💭): use your judgement; address important ones before proceeding.
- When no blockers remain, continue to Step 9.

## Step 9 — Push and open PR

Step 7 must have passed on the exact commit you are about to push. Check that no CI run for this branch is in progress (`gh pr checks` / `gh run list --branch <branch>`) — see Step 11.

```bash
git push -u origin <branch>
```

Follow the PR title format from CLAUDE.md.

```bash
gh pr create --title "<number>: short description" --body "$(cat <<'EOF'
## Summary
- <bullet points>

## Test plan
- [ ] <verification steps>

## Coverage report
\`\`\`
<coverage output from Step 7c>
\`\`\`

## Why coverage is not 100%

- `src/foo.ts: bar()` — 85%: the error branch requires OS-level fault injection and cannot be triggered in unit tests
- ...one entry per under-covered file or function...

Closes #<number>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

Omit the `## Why coverage is not 100%` section entirely if every changed line is covered.

## Step 10 — Capture learnings

After the PR is open, reflect on what happened during this implementation session. Ask: **did anything non-obvious come up that would help future sessions?**

Examples of things worth capturing:
- A pattern or convention discovered from reading existing code (e.g. how errors are wrapped, how tests are structured)
- A decision made that isn't obvious from the code (e.g. why a certain approach was chosen over another)
- A gotcha or edge case that caused confusion or required backtracking
- A new rule that emerged from review feedback

### What to update

- **CLAUDE.md** — for project-wide conventions, handler guidelines, architecture rules, or patterns that every future implementation should follow
- **Memory files** — for patterns specific to a sub-system, feedback about how to collaborate, or anything that isn't a hard project rule

If nothing non-obvious came up, skip this step — do not invent learnings.

### How to update CLAUDE.md

Add the learning under the most relevant existing section, or create a new section if needed. Keep entries concise (1–3 lines). Do not duplicate what is already documented.

```bash
# Read CLAUDE.md first, then Edit to add the new entry
```

### How to update memory

Use the Write tool to create or update the relevant memory file in the project's memory directory (check `~/.claude/projects/` for the matching project path), then update `MEMORY.md` with a pointer if it's a new file.

## Step 11 — Fixing CI or review feedback after the PR is open

Every later push follows the same rules as the first:

1. **Never push while a CI run for the PR is still in progress** (`gh pr checks <pr>` shows pending, or `gh run list --branch <branch>` shows `in_progress`/`queued`). A push cancels or races that run and wastes the runner. Wait for it to finish (`gh pr checks <pr> --watch --interval 60`), collect *all* failures and review comments, and fix them in one batch.
2. Re-run the whole Step 7 gate locally for the batch, then push once.
3. If the PR title changes, re-lint it with commitlint (Step 7b). Never use `--no-verify`.
