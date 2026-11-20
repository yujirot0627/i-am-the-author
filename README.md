# i-am-the-author

A Claude Code plugin (with one skill) that stops Claude from listing itself as a co-author when it commits, pushes, or creates pull requests. The user is the only author.

The name `i-am-the-author` is a bit of humor: "I'm the author (not Claude)."

## What it does

It works in two layers.

### The setting: prevents the signature from being added

The skill makes sure `~/.claude/settings.json` (on Windows, `%USERPROFILE%\.claude\settings.json`) has the following `attribution` setting. If it is missing, the skill adds it without breaking the existing content.

```json
{ "attribution": { "commit": "", "pr": "", "sessionUrl": false } }
```

With this setting, Claude Code stops adding its signature to new commits and pull requests: the `Co-Authored-By` trailer, the `Claude-Session` trailer, and the `🤖 Generated with [Claude Code]` text with its session URL. The setting takes effect from the next session. It does not change commits that already exist.

### The skill: removes signatures that were already added

The skill is invoked right before a commit, push, or PR creation, and does the following automatically without asking:

1. Removes these lines from commit messages and PR bodies:
   - `Co-Authored-By:` lines whose email address is `noreply@anthropic.com` (case-insensitive)
   - `Claude-Session: ...` lines
   - `🤖 Generated with [Claude Code]...` lines and the session URL that follows
2. Before a push, finds unpushed commits on the current branch (commits not reachable from any remote branch) that already carry these lines, and removes the lines from their messages. Commits that have already been pushed are never rewritten. See [How unpushed commits are rewritten](#how-unpushed-commits-are-rewritten).

This covers signatures the setting cannot prevent: commits made before the setting was added, or in the current session before it takes effect.

Human co-authors are kept. A `Co-Authored-By:` line that only has `Claude` in the name is kept too, because it may be a human. For example, `Co-authored-by: Claude Martin <claude@example.com>` is kept, but `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>` is removed.

It is designed to work on macOS, Linux, and Windows. The commands it runs use only `git` and `gh`, so they work whether Claude Code runs them in bash, zsh, Git Bash, or PowerShell. The text filtering runs inside `sh`, which git provides on every platform.

## How unpushed commits are rewritten

The skill uses `git filter-branch` to rewrite the messages of unpushed commits. The official Git documentation does not recommend this command, but this skill uses it only to rewrite the messages of unpushed commits:

- It selects only commits that are not reachable from any remote branch (`<branch> --not --remotes`), so pushed commits are never touched.
- It changes only the commit messages. File contents, parents, and merge commits stay exactly as they are, so merges are not redone and conflicts cannot come back.
- It never force-pushes. After the rewrite, a normal `git push` is used.

## Testing status

Tested on macOS (bash, git 2.55.0), including linear history, merge commits, merges with resolved conflicts, and recovery from a failed rewrite. **Not yet tested on Windows** (Git Bash or PowerShell).

If you have a Windows machine and can help test it, please get in touch or contribute by opening a GitHub issue or pull request.

## Installation

Run these two commands in Claude Code:

```
/plugin marketplace add yujirot0627/i-am-the-author
/plugin install i-am-the-author@i-am-the-author
```

## Usage

After installation, just ask Claude to commit, push, or create a PR, and the skill is invoked automatically.
To invoke it manually, type `/i-am-the-author:i-am-the-author`.

## Requirements

- `git` (on Windows, Git for Windows, which includes the `sh` the skill uses)
- `gh` (GitHub CLI) if you work with PRs

## License

[MIT](LICENSE)
