# Code review broken after updating `main` — analysis

## Scope

Comparing the previously-working checkout (`more-configuration`, `HEAD`, commit `775158b`)
against the latest upstream `main` (`origin/main`, commit `7a9908f`).

The reported symptoms:

1. **Diffs don't work** — Junie no longer obtains/uses the MR diff during code review.
2. **No comments are created, especially none for concrete lines with suggestions** — the
   inline (per-line) review comments disappear.

Both symptoms are the *code-review* path. This directory documents what changed and why.

## How code review works (recap)

The code-review feature does **not** use the generic `task` prompt. It emits a special
`codeReviewTask` object (`src/models/task-extraction-result.ts`):

```ts
{
  codeReviewTask: {
    diffCommand: "git diff origin/<target-branch>...",   // how Junie gets the diff
    description: "<full generated prompt>",               // context for the review
  }
}
```

- The **diff** is obtained by Junie running `git diff origin/<target>...` (the `diffCommand`).
- The **inline comments** are posted by Junie itself via the GitLab MCP server
  (`@zereight/mcp-gitlab`, configured in `src/mcp.ts`), on concrete lines.
- The `description` is the same assembled prompt as a generic task: rich MR context +
  **`GIT_OPERATIONS_NOTE`** (appended by `GitLabPromptFormatter.generatePrompt`).

## TL;DR — root cause

The only change that alters the **content of the prompt sent to Junie for code review** is the
rewrite of `GIT_OPERATIONS_NOTE` in commit `8ad285f` ("…updated git operations constants for
stricter enforcement"). It changed from a mild

> "Do NOT commit or push changes. The system will handle all git operations…"

to a blanket

> "**GIT OPERATIONS ARE STRICTLY FORBIDDEN**: Do NOT run **ANY** git commands yourself under any
> circumstances … `git add`, `git commit`, `git push`, `git pull`, `git fetch`, `git merge`,
> `git rebase`, `git checkout`, `git switch`, `git branch`, `git reset`, `git revert`,
> `git cherry-pick`, `git tag`, `git stash`, `git remote`, `git config` — and any equivalent
> operations performed via shell, scripts, MCP tools, or any other means…"

This note is injected into the `description` of the `codeReviewTask`, while the *same* task
instructs Junie to run `git diff origin/<target>...`. The two instructions are now in direct
contradiction:

- structured `diffCommand` ⇒ "run `git diff` to get the diff";
- injected `description` ⇒ "do NOT run any git commands".

When Junie resolves this contradiction against the stricter instruction, it does not run the
`git diff` (or treats it as forbidden), so **no diff is obtained → no review → no inline comments
with line-level suggestions**.

Details and evidence in [01-root-cause.md](./01-root-cause.md).

## What else changed (and why it is *not* the cause)

| Change | Commit | Effect on code review |
|--------|--------|-----------------------|
| `fetchBranch(targetBranch)` before checkout | `352c023` | **Fix**, not break — makes `origin/<target>` exist so `git diff origin/<target>...` can resolve. |
| Pre-create isolated `junie/<ts>` working branch | `8ad285f` | Skipped for code-review tasks (`if (!junieTask.codeReviewTask && …)`) — no effect. |
| `neutralizeJunieTriggers` on outgoing text | `5e46bf1`, `9fda5ce` | Applies to *outgoing* comments only; not to inline MCP comments or the diff. |
| BYOK API keys (`--openai-api-key` etc.) | `a7a735b` | Only affects auth/`--auth` arg; unrelated to diff/inline comments. |
| Retry on 409 "resource lock" | `8253872` | Unrelated. |
| `getLastFailedPipelineForMR` rename | `4e72161` | `fix-ci` only, not code review. |
| Removed `project_token_access_level` / `AccessLevelVariable` | — (only on old branch) | `init` only, not code review. |

Full commit-by-commit breakdown in [02-diff-analysis.md](./02-diff-analysis.md).

## Environment factor worth checking

Two dependencies are **unpinned** and change independently of this repo's history:

- **Junie CLI** — installed in the Docker image via
  `curl -fsSL https://junie.jetbrains.com/install.sh | bash` (always latest).
- **GitLab MCP server** — `npx -y @zereight/mcp-gitlab` (always latest).

So "updated main" may also have picked up a newer Junie/MCP-server at rebuild time. If the
root-cause in this document doesn't fully reproduce, pin these versions in the Dockerfile and
`mcp.ts` and bisect.

## Files

- [01-root-cause.md](./01-root-cause.md) — the `GIT_OPERATIONS_NOTE` contradiction, with evidence.
- [02-diff-analysis.md](./02-diff-analysis.md) — full commit-by-commit diff of the code-review path.
