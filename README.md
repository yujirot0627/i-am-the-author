# i-am-the-author

A Claude Code skill that stops Claude from listing itself as a co-author when it commits, pushes, or creates pull requests. The user is the only author.

The name `i-am-the-author` is a bit of humor: "I'm the author (not Claude)."

## What it does

It is invoked right before a commit, push, or PR creation, and does the following automatically without asking:

1. Removes these lines from commit messages and PR bodies:
   - `Co-Authored-By:` lines whose email address is `noreply@anthropic.com` (case-insensitive)
   - `Claude-Session: ...` lines
   - `🤖 Generated with [Claude Code]...` lines and the session URL that follows
2. Before a push, removes the same lines only from unpushed commits on the current branch (commits not reachable from any remote branch). Commits that have already been pushed are never rewritten. See [How unpushed commits are rewritten](#how-unpushed-commits-are-rewritten).
3. If `~/.claude/settings.json` (on Windows, `%USERPROFILE%\.claude\settings.json`) does not have the following attribution settings, adds them without breaking the existing content:

   ```json
   { "attribution": { "commit": "", "pr": "", "sessionUrl": false } }
   ```

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

Put this folder in a Claude Code skills directory.

```bash
# macOS / Linux: to use it in all projects
cp -R imtheauthor ~/.claude/skills/i-am-the-author

# macOS / Linux: to use it in one project only
cp -R imtheauthor <project>/.claude/skills/i-am-the-author
```

```powershell
# Windows (PowerShell): to use it in all projects
Copy-Item -Recurse imtheauthor "$env:USERPROFILE\.claude\skills\i-am-the-author"
```

## Usage

After installation, just ask Claude to commit, push, or create a PR, and the skill is invoked automatically.
To invoke it manually, type `/i-am-the-author`.

The attribution settings take effect from the next session. Until then, the skill removes the lines every time.

## Requirements

- `git` (on Windows, Git for Windows, which includes the `sh` the skill uses)
- `gh` (GitHub CLI) if you work with PRs

## License

[MIT](LICENSE)
