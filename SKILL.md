---
name: i-am-the-author
description: Always invoke this skill right before committing, pushing, or creating a pull request. Use it before running git commit, git push, or gh pr create (including editing a PR body). It automatically removes Claude co-author attribution (Co-Authored-By trailers with noreply@anthropic.com, Claude-Session, 🤖 Generated with Claude Code and the session URL) from commit messages and PR bodies without asking, strips the same lines from unpushed commits, and adds settings to ~/.claude/settings.json that empty the attribution. Works on macOS, Linux, and Windows. The user is the only author.
---

# i-am-the-author

The user is the only author. Do not list Claude as a co-author.

Do everything below **without asking for confirmation**, then continue with the original task (commit, push, or PR creation).

## Lines to remove

- `Co-Authored-By:` lines whose email address is `noreply@anthropic.com` (case-insensitive). Lines that only have `Claude` in the name are kept, because they may be human co-authors.
- Lines starting with `Claude-Session:`
- Lines starting with `🤖 Generated with [Claude Code]`
- The session URL line that follows it (`https://claude.ai/code...`)

Keep human co-authors. For example, keep `Co-authored-by: Claude Martin <claude@example.com>`, but remove `Co-Authored-By: Claude Sonnet 5.5 <noreply@anthropic.com>`.

## Works on macOS, Linux, and Windows

On native Windows, Claude Code runs shell commands in Git Bash (Bash tool) or in PowerShell (PowerShell tool, which is on by default and is the only shell when Git for Windows is not installed). On macOS, Linux, and WSL it uses bash or zsh. To work in all of these shells:

- Use only `git` and `gh` commands. Do not use `perl`, `sed`, `head`, pipes, `$(...)`, or `if` blocks in the commands you run.
- Quote any argument that contains spaces or special characters with single quotes. Do not put double quotes or emoji in command arguments (Windows PowerShell 5.1 mangles them).
- Do the text filtering inside a small script that git runs with its own bundled `sh`, which Git for Windows includes. Write files with the Write tool, not with shell redirection.
- Run all commands from the top-level directory of the repository.

### The strip script

Find the git directory with `git rev-parse --absolute-git-dir`. Then use the Write tool to create (or overwrite) `<git-dir>/i-am-the-author-strip.sh` with exactly this content. Files inside the git directory are never committed.

```sh
#!/bin/sh
# Message filter: reads a commit message on stdin and prints it without Claude attribution lines.
# git stripspace trims the blank lines left behind.
grep -viE '^(co-authored-by:.*noreply@anthropic\.com|claude-session:|🤖 generated with \[claude code\]|[[:space:]]*https://claude\.ai/code)' \
  | git stripspace
```

### The rewrite command

This command rewrites the messages of unpushed commits on the current branch, and nothing else:

```
git filter-branch -f -d .git/i-am-the-author-tmp --msg-filter 'sh ../../i-am-the-author-strip.sh' -- <branch> --not --remotes
```

- `<branch>` is the current branch name from `git branch --show-current`. If it prints nothing (detached HEAD), do not rewrite. Tell the user instead.
- `<branch> --not --remotes` selects only commits that are not reachable from any remote branch. Pushed commits are never rewritten, even when they appear in merges.
- Only the messages change. Trees, parents, and merge commits are kept exactly as they are, so merges are not redone and conflicts cannot come back.
- git runs the filter with its own `sh` inside `.git/i-am-the-author-tmp/t`, so `../../i-am-the-author-strip.sh` is the script in the git directory. This works whichever shell Claude Code uses.
- git prints a warning about `filter-branch` and waits a few seconds before it starts. This is expected. To skip the wait, set the environment variable `FILTER_BRANCH_SQUELCH_WARNING=1` first (`FILTER_BRANCH_SQUELCH_WARNING=1 git filter-branch ...` in bash, `$env:FILTER_BRANCH_SQUELCH_WARNING = '1'` before the command in PowerShell).
- If `git rev-parse --git-dir` does not print `.git` (for example, in a linked worktree), use the path it prints instead of `.git` in `-d` and in the file paths in steps 1 and 3. Use forward slashes in that path.

After a successful rewrite, git keeps a backup of the old branch in `refs/original/`. Delete it with `git update-ref -d refs/original/refs/heads/<branch>` once the check in step 2 prints nothing.

## Steps

### 1. When committing

Do not include the lines above when writing the message.
If a message you already wrote contains them, remove them before committing. Write the message to `<git-dir>/i-am-the-author-msg.txt` with the Write tool and run `git commit -F .git/i-am-the-author-msg.txt`.
Right after committing, check with `git log -1 --format=%B`. If any lines remain, follow step 2 to rewrite the unpushed commits.

### 2. When pushing (rewrite unpushed commits only)

Never rewrite commits that have already been pushed. Only commits that are not reachable from any remote branch are in scope.

Check whether any unpushed commit contains the lines:

```
git log HEAD --not --remotes -i -E --format=%H --grep='^co-authored-by:.*noreply@anthropic\.com' --grep='^claude-session:' --grep='^[^a-z0-9]*generated with \[claude code\]'
```

(This check uses `[^a-z0-9]*` instead of the emoji so the argument stays ASCII. It may also flag a commit that has the line without the emoji. The strip script still removes only the exact lines listed above.)

If nothing is printed, push as usual.
If any commits are printed, run the rewrite command above.

- If the working tree has uncommitted changes, run `git stash` first and `git stash pop` when done. (`git filter-branch` refuses to run with uncommitted changes.)
- If `git filter-branch` fails, the branch is left unchanged. Tell the user what happened. Never force-push. If you ran `git stash` beforehand, also run `git stash pop` to restore the work in progress.
- After rewriting, run the check above again to confirm nothing is printed, delete the backup in `refs/original/`, then run a normal `git push`.

### 3. When creating or editing a PR

Remove the lines above from the PR body yourself, write the result to `<git-dir>/i-am-the-author-pr.md` with the Write tool, and pass it with `gh pr create --body-file .git/i-am-the-author-pr.md`.
Also check that the PR title does not contain the lines.
If an existing PR body contains them, read it with `gh pr view <number> --json body --jq .body`, write the cleaned body to the same file, and run `gh pr edit <number> --body-file .git/i-am-the-author-pr.md`.

Handle any unpushed commits in the PR with step 2 before pushing.

### 4. Check and add the attribution settings

Check whether `~/.claude/settings.json` (on Windows, `%USERPROFILE%\.claude\settings.json`) has the following settings that empty the attribution.

```json
{
  "attribution": {
    "commit": "",
    "pr": "",
    "sessionUrl": false
  }
}
```

- `attribution.commit`: the `Co-Authored-By` trailer added to commits. `""` disables it.
- `attribution.pr`: the `🤖 Generated with Claude Code` text added to PR bodies. `""` disables it.
- `attribution.sessionUrl`: the `Claude-Session` trailer and the session URL in PRs. `false` disables it.
- You can also write `"attribution": false`, but Claude Code versions older than v2.1.281 ignore the entire settings file when it is there. Use the form above instead.
- The legacy key `includeCoAuthoredBy` is deprecated. Do not add it (but keep any existing value).

Proceed as follows:

1. Read the settings file with Read. If the file does not exist, treat it as `{}`.
2. If it already has `"attribution": false`, or all three keys have the values above, do nothing.
3. Otherwise, use Edit to add the `attribution` object or fill in only the missing keys. Do not change any other keys, values, or ordering.
4. Read the file again and confirm it is still valid JSON (matching braces and quotes, commas between entries, no trailing comma).

The settings take effect from the next session. In the current session, remove the lines using steps 1 to 3.

## When done

Report in one or two lines what you removed, which commits you rewrote, and whether you added the settings, then continue with the original task.
