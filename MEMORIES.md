# Agent memories

- Project artifacts are in English: code, docs, CONTEXT.md, ADRs, issues and commit messages. The owner chats in Spanish, but that does not carry over to repo content, except for the occasional doc they ask for in Spanish.
- Create worktrees with `wtnew <branch>`, the owner's PowerShell profile function that wraps `herdr worktree create --branch <branch> --base main`. Never use `git worktree add` or the Agent tool's `isolation: "worktree"`, because Herdr doesn't list those. For a subagent, create the worktree first and pass it the path from the JSON output (`.result.worktree`).
