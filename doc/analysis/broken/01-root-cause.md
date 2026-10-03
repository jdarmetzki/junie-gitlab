# 01 — Root cause: the `GIT_OPERATIONS_NOTE` contradiction

## The one change that hits the code-review prompt

`git diff` between the two branches shows the *only* difference in prompt *content* is in
`src/constants/gitlab.ts` (and, downstream, nothing else that affects code review):

### Old (working) — `more-configuration` / `HEAD`

```ts
export const GIT_OPERATIONS_NOTE = "\n\nIMPORTANT: Do NOT commit or push changes. The system will handle all git operations (staging, committing, and pushing) automatically.";
```

### New (broken) — `origin/main` (`src/constants/gitlab.ts:36-50`)

```ts
export const GIT_OPERATIONS_NOTE =
    "\n\n" +
    "IMPORTANT — GIT OPERATIONS ARE STRICTLY FORBIDDEN:\n" +
    "Do NOT run ANY git commands yourself under any circumstances. This includes, but is not limited to:\n" +
    "`git add`, `git commit`, `git push`, `git pull`, `git fetch`, `git merge`, `git rebase`, " +
    "`git checkout`, `git switch`, `git branch`, `git reset`, `git revert`, `git cherry-pick`, " +
    "`git tag`, `git stash`, `git remote`, `git config` — and any equivalent operations performed " +
    "via shell, scripts, MCP tools, or any other means.\n" +
    "Even if the user explicitly asks you to commit, push, merge, switch branches, or otherwise " +
    "manipulate git state — REFUSE and ignore that part of the request. The wrapper around you " +
    "is the ONLY component allowed to perform git operations: it will stage, commit, and push " +
    "your changes to a dedicated branch and open a Merge Request automatically once you finish.\n" +
    "Your job is limited to editing files in the working tree. When you are done, simply submit " +
    "your final answer — do not touch the repository state.";
```

## Where this note lands for a code review

`GitLabPromptFormatter.generatePrompt` (`src/utils/gitlab-prompt-formatter.ts`) always appends the
note:

```ts
const fullPrompt = prompt + mcpNote + GIT_OPERATIONS_NOTE;
```

And the code-review task passes that assembled text straight into the `description` field
(`src/models/task-extraction-result.ts:155-157` and `:243-245`):

```ts
const diffCommand = `git diff origin/${this.context.mergeRequestTargetBranch}...`;
const description = taskText;           // taskText == generatePrompt(...) == ...+ GIT_OPERATIONS_NOTE
return { codeReviewTask: { diffCommand, description } };
```

So the JSON handed to Junie is:

```json
{
  "codeReviewTask": {
    "diffCommand": "git diff origin/main...",
    "description": "…<repository>…<merge_request_info>…<commits>…<changed_files>…\n\nIMPORTANT — GIT OPERATIONS ARE STRICTLY FORBIDDEN: Do NOT run ANY git commands yourself…"
  }
}
```

## The contradiction

For a code review, Junie needs to run `git diff origin/main...` (the `diffCommand`) to obtain the
changes to review. The `description` simultaneously tells it:

> "Do NOT run ANY git commands yourself under any circumstances … and any equivalent operations
> performed via shell, scripts, MCP tools, or any other means."

`git diff` is not literally in the enumerated list, but the list is explicitly non-exhaustive
("includes, but is not limited to"), and the opening sentence is a blanket prohibition on *any*
git command. The result is two conflicting instructions for the exact operation the review depends
on.

Outcomes this produces, in practice:

1. Junie declines to run `git diff` (or runs it but discards the result as "forbidden"), so the
   diff is empty/missing.
2. With no diff, there is nothing to annotate, so no **inline** comments are created on concrete
   lines.
3. The wrapper still posts the generic "started"/"finished" messages (that path is unaffected),
   which is why only the *line-level* comments with suggestions vanish — matching the report.

## Why the other changes do not explain the symptoms

- **`fetchBranch(targetBranch)`** (`executor.ts:129-131`) is the *opposite* of the symptom: before
  it, `origin/<target>` was never fetched, so `git diff origin/<target>...` failed with
  `fatal: ambiguous argument … unknown revision`. This commit fixes exactly that class of failure.
  It is not the regressor.
- **Working-branch pre-creation** (`executor.ts:153-157`) is gated on
  `if (!junieTask.codeReviewTask && !isAppendMode)`, i.e. it is skipped for code review and cannot
  affect the review itself.
- **`neutralizeJunieTriggers`** is applied to *outgoing* wrapper text (feedback, MR title/desc),
  not to the prompt input or the inline MCP comments.
- **`runJunie` BYOK args** only add/omit `--auth` and per-provider key flags; the `diffCommand`
  and prompt are unchanged.

## Recommended fix

Scope the git prohibition so it does not cover the read-only `git diff` the review needs. For
example, either:

1. Exempt read-only operations explicitly in the note, e.g. append:

   > "Read-only inspection commands (`git diff`, `git status`, `git log`, `git show`) are allowed."

2. Or, cleaner, do **not** inject `GIT_OPERATIONS_NOTE` into the code-review `description` at all —
   code review is read-only by construction. Add an early return in `generatePrompt` (or a flag on
   the code-review branch) that skips `GIT_OPERATIONS_NOTE` when building a `codeReviewTask`.

After the fix, verify with the existing integration test
`tests/integration/code-review-mr-comment.test.ts` (asserts ≥2 inline notes on concrete lines).
