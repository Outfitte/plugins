# plugins

A Claude Code plugin marketplace for Outfitte contributors. Install it to get skills and slash commands tailored to the Outfitte codebase.

## Install

```shell
/plugin marketplace add Outfitte/plugins
```

Then install individual plugins:

```shell
/plugin install tdd@outfitte-plugins
/plugin install go-commands@outfitte-plugins
/plugin install gh-cli@outfitte-plugins
/plugin install create-issues@outfitte-plugins
/plugin install split-issue@outfitte-plugins
```

## Plugins

| Plugin | Description |
| --- | --- |
| `tdd` | RED-GREEN-REFACTOR cycle, test naming conventions, stub pattern, coverage rules |
| `go-commands` | Go build, test, coverage, and Lambda cross-compile command reference |
| `gh-cli` | GitHub operations via `gh` CLI — issues, PRs, milestones, sub-issues, releases |
| `create-issues` | Bulk-create GitHub issues from a JSON task file and link them to a milestone |
| `split-issue` | Decompose a GitHub issue into sub-issues per method, endpoint, or handler |
