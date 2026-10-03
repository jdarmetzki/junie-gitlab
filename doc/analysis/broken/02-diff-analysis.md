# 02 — Full diff analysis (working → latest `main`)

The two compared revisions diverge at merge-base `dfac76a`:

- `HEAD` (`more-configuration`) = `dfac76a` → `7b67e7c` (`AccessLevelVariable`) → `775158b` (docs).
- `origin/main` = `dfac76a` → … → `7a9908f` (latest).

Below, every commit on `origin/main` not on `HEAD` that touches the code-review path is listed
with what it does and whether it can explain the symptoms.

## Commits on `origin/main` (not on `HEAD`), code-review-relevant

### `a7a735b` — BYOK support

Adds `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, `GROK_API_KEY`, `OPENROUTER_API_KEY`, `GOOGLE_API_KEY`
as alternatives to `JUNIE_API_KEY`; `junieApiKey` becomes nullable; `runJunie` gains per-provider
flags (`--openai-api-key`, …) and makes `--auth` conditional.

- Affects: authentication only.
- Code review impact: none (diff/prompt unchanged).
- Verdict: **not the cause.**

### `f19fcc5` / `352c023` — fetch target branch before code review

`352c023` adds `fetchBranch()` in `git-api.ts` and calls it in `executor.ts`:

```ts
const targetBranch = taskExtractionResult.targetBranch;
if (targetBranch && targetBranch !== branchToPull) {
    await fetchBranch(projectPath, targetBranch);
}
await checkoutBranch(projectPath, branchToPull);
```

Plus a `targetBranch` getter on each task class.

- Affects: makes `origin/<target>` exist so `git diff origin/<target>...` resolves.
- Code review impact: **fixes** a real diff-resolution bug (the diff previously could fail because
  only the source branch was fetched).
- Verdict: **not the cause** (opposite direction).

### `8ad285f` — isolated working branch + stricter git note

Two parts:

**(a) Working branch.** For non-code-review, non-append tasks, pre-creates `junie/<ts>` off the
checkout branch so agent-side pushes don't land on the target branch:

```ts
let workingBranch = branchToPull;
if (!junieTask.codeReviewTask && !isAppendMode) {
    workingBranch = `junie/${Date.now()}`;
    await checkoutLocalBranch(workingBranch);
}
```

Gated on `!junieTask.codeReviewTask`, so code review stays put.

**(b) `GIT_OPERATIONS_NOTE`** rewritten to a blanket "GIT OPERATIONS ARE STRICTLY FORBIDDEN" with
an exhaustive list of forbidden commands and "…and any equivalent operations performed via shell,
scripts, MCP tools, or any other means."

- Affects: (a) generic tasks only; (b) **every prompt**, including the code-review `description`.
- Code review impact: (b) injects a "no git commands" instruction into the same task that asks
  Junie to run `git diff`.
- Verdict: **(b) is the primary cause** — see [01-root-cause.md](./01-root-cause.md).

### `5e46bf1` / `9fda5ce` / `aa61aae` — `neutralizeJunieTriggers`

Adds `neutralizeJunieTriggers` to `sanitizer.ts` and applies it to **outgoing** text (feedback
notes, MR title/description) to prevent recursive pipeline triggers when a comment quotes `#junie`
or `@project_<id>_bot`.

- Affects: outgoing wrapper-authored text only; explicitly *not* prompt input (and skips fenced
  code blocks).
- Code review impact: none on the diff or on inline MCP comments (which are posted by Junie, not
  the wrapper).
- Verdict: **not the cause.**

### `8253872` — retry on 409 "resource lock"

Extends `withRetry` to retry HTTP 409 "resource lock" conflicts in addition to network errors.

- Affects: API call resilience only.
- Code review impact: none.
- Verdict: **not the cause.**

### `4e72161` — `getLastFailedPipelineForMR`

Renames `getLastCompletedPipelineForMR` → `getLastFailedPipelineForMR` and changes `fix-ci` to use
it.

- Affects: `fix-ci` flow only.
- Code review impact: none.
- Verdict: **not the cause.**

### `683710f` / `b50bc45` / build & test infra

Bun → Node, test scripts, `tsconfig.build.json` history, local-GitLab fixture. No change to the
code-review runtime path (final `Dockerfile` and `tsconfig.json` are effectively unchanged in the
relevant parts).

- Verdict: **not the cause.**

## Net diff on the code-review path (HEAD → origin/main)

| File | Change | Code-review relevance |
|------|--------|-----------------------|
| `src/constants/gitlab.ts` | `GIT_OPERATIONS_NOTE` → strict/forbidding | **The prompt change (root cause)** |
| `src/executor.ts` | `fetchBranch` + `targetBranch`; working-branch pre-create; BYOK `runJunie` args; `neutralizeJunieTriggers` on MR title/desc | fetch = fix; working-branch = skipped for review; BYOK/neutralize = unrelated |
| `src/api/git-api.ts` | + `fetchBranch()` | fix (diff resolution) |
| `src/models/task-extraction-result.ts` | + `targetBranch` getters | only used by `fetchBranch` |
| `src/utils/sanitizer.ts` | + `neutralizeJunieTriggers` | outgoing text only |
| `src/utils/gitlab-prompt-formatter.ts` | `getLastFailedPipelineForMR` rename | fix-ci only |
| `src/context.ts` / `src/webhook-env.ts` | BYOK keys | auth only |

## Conclusion

The single change that alters the **prompt Junie receives for a code review** is the strict
`GIT_OPERATIONS_NOTE`. It creates a contradiction with the code-review `diffCommand`
(`git diff origin/<target>...`), which explains both symptoms — no diff, and therefore no
line-level inline comments with suggestions. All other changes are fixes, no-ops for code review,
or auth-only. Fix and verification guidance is in
[01-root-cause.md](./01-root-cause.md).
