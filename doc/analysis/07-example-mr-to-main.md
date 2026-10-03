# 07 — Concrete examples: new MR on a branch → `main`

This is the **merge_request event** case. When someone opens an MR from a feature branch
(`feature/login`) into `main`, GitLab fires a `merge_request` webhook with
`action: "open"`. The sample pipeline's `if-mr-open` rule runs the job and injects
`JUNIE_CUSTOM_PROMPT: "code-review"` (`script-sample.yaml:59-64`).

What the wrapper does (`executor.ts:extractTaskFromEnv` → `MergeRequestEventTask`):

1. `eventKind === 'merge_request'` and `customPrompt` is set → task succeeds.
2. Checks out `mrEventSourceBranch` (`feature/login`).
3. Since `customPrompt` matches `code-review`, `generateJuniePrompt` returns a **code-review task**:
   - `diffCommand = git diff origin/main...` (target branch = `main`)
   - `description =` the full assembled prompt text below.

---

## Example A — automatic code review (the default for a new MR)

### The `JunieTask` JSON written to `/junieCache/junie_input.json`

```json
{
  "codeReviewTask": {
    "diffCommand": "git diff origin/main...",
    "description": "You were triggered as a GitLab AI Assistant by merge_request event. Your task is to:\n\n<user_instruction>\ncode-review\n</user_instruction>\n<repository>\nProject ID: 42\nProject: backend\n</repository>\n<merge_request_info>\nMR !17\nTitle: Add login form\nDescription:\nImplements the login page with email/password validation.\nAuthor: @jdoe\nState: opened\nBranch: feature/login -> main\nBase SHA: 3f9c2b1a7d0e4f5\nHead SHA: 8a1b2c3d4e5f6a7b\nChanges: 5\nDiscussions: 2\nUpvotes: 1 / Downvotes: 0\n\n</merge_request_info>\n<commits>\n[2026-09-30] a1b2c3d - Add login form component\n[2026-09-30] e4f5g6h - Add validation logic\n</commits>\n<discussions>\nDiscussion #5581:\n[Sep 30, 2026, 02:14 PM] @jdoe:\nShould we extract the validation into a helper?\n\n---\n\n</discussions>\n<changed_files>\nsrc/LoginForm.tsx (added)\nsrc/validation.ts (added)\nsrc/App.tsx (modified)\nsrc/api/auth.ts (modified)\nsrc/types.ts (modified)\n</changed_files>\n<actor>\nEvent: merge_request\nPipeline ID: 1234567\n</actor>\n\nContent for MCP usage (if needed):\ncurrent project ID: 42\ncurrent merge request ID: 17\n\nIMPORTANT: Do NOT post your summary as a comment. The summary will be posted automatically by the system.\n\nIMPORTANT: Do NOT commit or push changes. The system will handle all git operations (staging, committing, and pushing) automatically."
  }
}
```

### The readable prompt text (same `description`, unescaped)

```
You were triggered as a GitLab AI Assistant by merge_request event. Your task is to:

<user_instruction>
code-review
</user_instruction>
<repository>
Project ID: 42
Project: backend
</repository>
<merge_request_info>
MR !17
Title: Add login form
Description:
Implements the login page with email/password validation.
Author: @jdoe
State: opened
Branch: feature/login -> main
Base SHA: 3f9c2b1a7d0e4f5
Head SHA: 8a1b2c3d4e5f6a7b
Changes: 5
Discussions: 2
Upvotes: 1 / Downvotes: 0

</merge_request_info>
<commits>
[2026-09-30] a1b2c3d - Add login form component
[2026-09-30] e4f5g6h - Add validation logic
</commits>
<discussions>
Discussion #5581:
[Sep 30, 2026, 02:14 PM] @jdoe:
Should we extract the validation into a helper?

---

</discussions>
<changed_files>
src/LoginForm.tsx (added)
src/validation.ts (added)
src/App.tsx (modified)
src/api/auth.ts (modified)
src/types.ts (modified)
</changed_files>
<actor>
Event: merge_request
Pipeline ID: 1234567
</actor>

Content for MCP usage (if needed):
current project ID: 42
current merge request ID: 17

IMPORTANT: Do NOT post your summary as a comment. The summary will be posted automatically by the system.

IMPORTANT: Do NOT commit or push changes. The system will handle all git operations (staging, committing, and pushing) automatically.
```

### The `junie` command actually executed

```bash
junie --auth "perm-xxxxxxxxxxxx" --cache-dir="/junieCache" \
      --output-format="json" --input-format="json" \
      --json-output-file="/junieCache/junie_output.json" \
      --model="sonnet" \
      < "/junieCache/junie_input.json"
```

(`--model="sonnet"` comes from `JUNIE_MODEL` in the pipeline; `--guidelines-file` would be
appended if `JUNIE_GUIDELINES_FILENAME` were set.)

Note: the code-review path does **not** pass the prompt on the command line or as `task`; it
passes a `codeReviewTask` object. Junie then runs `git diff origin/main...` itself and reviews
those changes.

---

## Example B — custom prompt on the same event

If `JUNIE_CUSTOM_PROMPT` is set to something that is **not** `code-review`, the wrapper falls back
to a generic `task` (no `codeReviewTask`, no `diffCommand`). Example:
`JUNIE_CUSTOM_PROMPT = "Run the full test suite and report any failures"`.

```json
{
  "task": "You were triggered as a GitLab AI Assistant by merge_request event. Your task is to:\n\n<user_instruction>\nRun the full test suite and report any failures\n</user_instruction>\n<repository>\nProject ID: 42\nProject: backend\n</repository>\n<merge_request_info>\nMR !17\nTitle: Add login form\n...\n</merge_request_info>\n<commits>\n...\n</commits>\n<discussions>\n...\n</discussions>\n<changed_files>\n...\n</changed_files>\n<actor>\nEvent: merge_request\nPipeline ID: 1234567\n</actor>\n\nContent for MCP usage (if needed):\ncurrent project ID: 42\ncurrent merge request ID: 17\n\nIMPORTANT: Do NOT post your summary as a comment. ...\n\nIMPORTANT: Do NOT commit or push changes. ..."
}
```

The `<user_instruction>` is now literally the custom prompt (not `code-review`).

---

## Example C — no custom prompt (nothing happens)

Without `JUNIE_CUSTOM_PROMPT`, `extractTaskFromEnv` returns
`FailedTaskExtractionResult("MR event action 'open' no custom prompt set")` and no Junie call is
made (`executor.ts:218-220`). This is why the sample pipeline *must* inject the variable.

---

## Section-by-section mapping for this event

| Prompt section | Built by | For a new MR |
|----------------|----------|--------------|
| `You were triggered …` | `buildPrompt` header | `merge_request` |
| `<user_instruction>` | `getUserInstructionForMREvent` | `code-review` (or custom prompt) |
| `<repository>` | `getRepositoryInfo` | project id + name |
| `<merge_request_info>` | `getMRInfo` | MR iid, title, description, author, state, `feature/login -> main`, base/head SHA, changes, discussions, votes |
| `<commits>` | `getCommitsInfo` | `[date] short-id - title` per commit |
| `<discussions>` | `getDiscussionsInfo` | threaded, non-system notes |
| `<changed_files>` | `getChangedFilesInfo` | `path (added/modified/deleted/renamed)` |
| `<actor>` | `getActorInfo` | event + pipeline id |
| MCP note | `getMcpNote` | project id + MR id (no comment id) |
| `GIT_OPERATIONS_NOTE` | constant | "Do NOT commit or push…" |

Because `discussions` are fetched with `fetchMergeRequestData` and no `triggerTime` filter is
passed here, all discussions at fetch time are included (`gitlab-data-fetcher.ts:37-42`).
