# 03 — Prompts

Every successful task ends with a call to `generateJuniePrompt(useMcp)` returning a `JunieTask`
(`models/task-extraction-result.ts:35-38`):

```ts
interface JunieTask {
    task?: string;                                     // generic: full prompt text
    codeReviewTask?: { diffCommand: string; description?: string };  // code-review mode
}
```

There are two top-level prompt "modes":

- **Generic mode** (`task`): the full instruction text assembled by `GitLabPromptFormatter`.
- **Code-review mode** (`codeReviewTask`): a `diffCommand` plus a `description`; the wrapper's
  `runJunie` treats this differently (see 04).

## Where prompt text comes from

`GitLabPromptFormatter.generatePrompt()` (`utils/gitlab-prompt-formatter.ts:37`) does, in order:

1. `buildPrompt()` — assembles the body from event-type-specific sections.
2. Appends an **MCP note** (if `USE_MCP`), via `getMcpNote()` → `generateMcpNote()`.
3. Appends a fixed `GIT_OPERATIONS_NOTE`.
4. Processes markdown attachments (downloads `/uploads/...`, rewrites URLs to local paths).
5. Runs `sanitizeContent()` over the result (prompt-injection + token redaction).

## The generic prompt template

`buildPrompt` (`gitlab-prompt-formatter.ts:64-130`) composes:

```
You were triggered as a GitLab AI Assistant by {eventKind} event. Your task is to:

<user_instruction>
{instruction}
</user_instruction>
<repository>
Project ID: {projectId}
Project: {projectName}
</repository>
<merge_request_info>...     (MR events only)
<issue_info>...             (issue events only)
<commits>...                (MR events only)
<discussions>...            (all events)
<changed_files>...          (MR events only)
<actor>
Event: {eventKind}
Pipeline ID: {pipelineId}
</actor>
```

Which sections appear depends on the event type (`gitlab-prompt-formatter.ts:79-117`):

| Section | Issue comment | MR comment | MR event |
|---------|:---:|:---:|:---:|
| `<user_instruction>` | ✅ | ✅ | ✅ |
| `<repository>` | ✅ | ✅ | ✅ |
| `<merge_request_info>` | — | ✅ | ✅ |
| `<issue_info>` | ✅ | — | — |
| `<commits>` | — | ✅ | ✅ |
| `<discussions>` | ✅ | ✅ | ✅ |
| `<changed_files>` | — | ✅ | ✅ |
| `<actor>` | ✅ | ✅ | ✅ |

### `<user_instruction>` content

The `user_instruction` is the most important, event-specific part:

- **Issue comment** (`getUserInstructionForIssueComment`, line 155):
  - custom prompt → `{customPrompt}\n\nComment: {commentText}`
  - otherwise → the raw `commentText`.
- **MR comment** (`getUserInstructionForMRComment`, line 132):
  - if `code-review` → literally `code-review`
  - else if custom prompt → `{customPrompt}\n\nDiscussion #{id}:\nComment: {commentText}`
  - else → `Discussion #{id}:\n` + `commentText`.
- **MR event** (`getUserInstructionForMREvent`, line 168):
  - `code-review` in custom prompt → `code-review`
  - else custom prompt → the custom prompt
  - else → `Handle merge request {action}`.

## The `code-review` path

When `isMRCommandEvent(CODE_REVIEW_TRIGGER_PHRASE_REGEXP, ...)` is true
(`task-extraction-result.ts:145-149` for MR comments, `:229-233` for MR events), the task becomes:

```ts
{
  codeReviewTask: {
    diffCommand: `git diff origin/${mergeRequestTargetBranch}...`,
    description: <the full generic prompt text>
  }
}
```

The `description` is the *same* assembled prompt text; only the wrapper's handling changes
(see 04). Note the code-review branch still calls `generatePrompt`, so the MR context is still
fetched and included in `description`.

## The `fix-ci` path (fixed template)

Triggered by `FIX_CI_TRIGGER_PHRASE_REGEXP` in an MR comment
(`gitlab-prompt-formatter.ts:92-100`). It looks up the last **failed** pipeline for the MR
(`getLastCompletedPipelineForMR`) and **replaces** the whole prompt with
`createFixCIFailuresPrompt(projectId, pipelineId, mergeRequestId)`
(`constants/gitlab.ts:126-171`). That template instructs Junie to:

1. use `gitlab.list_pipeline_jobs` / `gitlab.get_pipeline_job_output` MCP tools to find failed jobs,
2. stop immediately if no failures,
3. correlate failures with the MR diff,
4. implement a minimal fix, or submit analysis if uncertain.

It hard-codes the GitLab MCP tool names (`gitlab.list_pipeline_jobs`, `gitlab.get_pipeline_job_output`,
`gitlab.get_merge_request_diffs`) — so `fix-ci` **requires MCP to be enabled**.

## The `minor-fix` path (fixed template)

Triggered by `MINOR_FIX_TRIGGER_PHRASE_REGEXP` in an MR comment
(`gitlab-prompt-formatter.ts:80-90`). It extracts text after `minor-fix`
(`extractMinorFixRequest`) and **replaces** the whole prompt with
`createMinorFixPrompt(projectId, mergeRequestId, userRequest)`
(`constants/gitlab.ts:78-117`). This template instructs Junie to fetch the MR diff and make a
minimal, scoped change.

## Appended notes

After the body, `generatePrompt` appends (in order):

1. **MCP note** (`constants/gitlab.ts:generateMcpNote`), only when `useMcp`:
   ```
   Content for MCP usage (if needed):
   current project ID: {projectId}
   current issue ID: {issueId}        (issues)
   current merge request ID: {mrId}   (MRs)
   current comment ID: {commentId}    (comments)
   IMPORTANT: Do NOT post your summary as a comment. ...
   ```
   plus, for MR comments, a note not to create new comments when replying to a thread.
2. **`GIT_OPERATIONS_NOTE`** (`constants/gitlab.ts:36`), always:
   ```
   IMPORTANT: Do NOT commit or push changes. The system will handle all git operations (staging, committing, and pushing) automatically.
   ```

## Sanitization & attachments

- `sanitizeContent` (`utils/sanitizer.ts`) strips HTML comments, invisible/zero-width chars,
  markdown image alt text, markdown link titles, HTML hidden attributes (`alt`, `title`,
  `aria-label`, `data-*`, `placeholder`), normalizes HTML entities, and **redacts GitLab tokens**
  (`glpat-…`, `gldt-…`, `GR13…`).
- `processMarkdownAttachments` (`utils/attachment-downloader.ts`) downloads any
  `/uploads/<hash>/<file>` references and replaces them with local `/tmp/gitlab-attachments/<file>`
  paths so Junie can actually read the files.

## How to influence the prompt

There are several levers, in decreasing order of "power":

1. **The comment text itself** — for issue/MR comments the user instruction is largely the raw
   comment (with `#junie` prefix). Writing a precise comment directly shapes the prompt.
2. **Trigger phrases** — `code-review`, `fix-ci`, `minor-fix <request>` switch to completely
   different prompt templates.
3. **`JUNIE_CUSTOM_PROMPT` env var** — set via the pipeline rule (`if-mr-open`) or injected
   manually. For MR events it is *required* for any run and becomes the user instruction; for
   comments it is prepended before the comment text.
4. **`JUNIE_GUIDELINES_FILENAME`** — passes `--guidelines-file` to Junie (see 04), adding a
   project guidelines file that Junie loads separately (not part of this prompt text).
5. **MCP enablement (`USE_MCP`)** — toggles the MCP note and the MCP config file; required for
   `fix-ci`/`code-review` inline comments.
6. **`JUNIE_MODEL`** — selects the model (does not change prompt text, but changes behavior).

> **Where to edit the actual prompt templates:** the fixed `minor-fix` and `fix-ci` templates
> live in `src/constants/gitlab.ts` (`createMinorFixPrompt`, `createFixCIFailuresPrompt`). The
> generic prompt is built in `src/utils/gitlab-prompt-formatter.ts`.
