# 02 — Triggers

Everything starts with a GitLab **webhook** that fires a pipeline in the Junie Workspace project.
That pipeline runs `node /app/dist/cli.js run`, and the wrapper figures out *why* it was triggered
from environment variables (`EVENT_KIND`, comment text, MR action, etc.).

There are two orthogonal layers that determine whether/when the wrapper does something:

1. **Webhook / pipeline rules** (GitLab CI `rules:`) — *whether the pipeline runs at all*.
2. **Task extraction** (`executor.ts:extractTaskFromEnv`) — *whether the run becomes a real task*.

## 1. Webhook triggers configured by `init`

`initializer.ts` creates a project webhook with these event subscriptions
(`initializer.ts:89-106`):

- `noteEvents: true` — comments on issues and MRs
- `mergeRequestsEvents: true` — MR lifecycle events (open/update/reopen/close/merge)
- `issuesEvents: false`, `pushEvents: false` — deliberately off

The webhook URL is a trigger-pipeline URL with two dynamic variables
(`initializer.ts:73`):

```
{api}/projects/{junieProjectId}/trigger/pipeline?ref={defaultBranch}&token={trigger_token}&inputs[project_token]={project_token}
```

Every event payload is mapped into pipeline variables via the **custom webhook template**
(`initializer.ts:75-84` + `webhook-env.ts`), so the wrapper sees GitLab payload fields
(`{{object_attributes.note}}`, `{{merge_request.iid}}`, …) as env vars.

## 2. Pipeline trigger rules (`script-sample.yaml`)

The `junie-run` job only runs on:

```yaml
- if: *if-junie-note      # EVENT_KIND == "note" AND comment matches /#junie.../ or bot mention
- if: *if-mr-open         # EVENT_KIND == "merge_request" AND MR_EVENT_ACTION == "open"
```

Where (`script-sample.yaml:28-29`):

```yaml
.trigger-conditions:
  - &if-junie-note '$CI_PIPELINE_SOURCE == "trigger" && $EVENT_KIND == "note" && ($COMMENT_TEXT =~ /#junie(\s|$)/i || $COMMENT_TEXT =~ /@project_[0-9]+_bot/i)'
  - &if-mr-open '$CI_PIPELINE_SOURCE == "trigger" && $EVENT_KIND == "merge_request" && $MR_EVENT_ACTION == "open"'
```

So at the **pipeline** level there are exactly two triggering cases:

| Case | Condition | Extra variables injected |
|------|-----------|--------------------------|
| Comment mentions `#junie` (or the bot handle `@project_N_bot`) | note event, comment matches regex | none |
| MR opened | merge_request event, action = `open` | `JUNIE_CUSTOM_PROMPT: "code-review"` |

The `junie-cleanup` job then deletes "idle" pipelines (comments that did not mention Junie, or
non-`open` MR actions), so only real work is kept (`script-sample.yaml:83-96`).

## 3. Task extraction (what the wrapper actually does)

Even after a pipeline runs, `extractTaskFromEnv` (`executor.ts:170`) decides the concrete task type:

### 3a. Issue comment (`eventKind === 'note'`, issue context)

- Requires a Junie mention (`checkTextForJunieMention`, `executor.ts:176`). A "mention" is either:
  - a literal `@junie` or `#junie` substring, or
  - the presence of an *active* project access token named `"Junie by JetBrains"`
    (`PROJECT_ACCESS_TOKEN_NAME`) — i.e. tagging the bot created during `init`.
- Result: `IssueCommentTask` (generic task; checkout `defaultBranch`).

### 3b. MR comment (`eventKind === 'note'`, MR context)

- Requires a mention (same check).
- Result: `MergeRequestCommentTask` (checkout `mergeRequestSourceBranch`).
- The *content* of the comment further selects a specialized prompt — see below.

### 3c. MR event (`eventKind === 'merge_request'`)

- **Only runs if `customPrompt` (i.e. `JUNIE_CUSTOM_PROMPT`) is set** (`executor.ts:211`).
- Without a custom prompt it returns `FailedTaskExtractionResult("…no custom prompt set")`.
- With `JUNIE_CUSTOM_PROMPT=code-review` (injected by the `if-mr-open` rule) → code-review task.
- Result: `MergeRequestEventTask` (checkout `mrEventSourceBranch`).

## 4. Trigger → task → prompt selection matrix

| Trigger surface | Detection point | Task class | Specialized prompt? |
|-----------------|-----------------|------------|---------------------|
| Issue comment `#junie <text>` | `checkTextForJunieMention` | `IssueCommentTask` | generic |
| MR comment `#junie code-review` | `CODE_REVIEW_TRIGGER_PHRASE_REGEXP` | `MergeRequestCommentTask` | code-review (`codeReviewTask`) |
| MR comment `#junie fix-ci` | `FIX_CI_TRIGGER_PHRASE_REGEXP` | `MergeRequestCommentTask` | fix-ci template |
| MR comment `#junie minor-fix <req>` | `MINOR_FIX_TRIGGER_PHRASE_REGEXP` | `MergeRequestCommentTask` | minor-fix template |
| MR comment `#junie <anything else>` | fallthrough | `MergeRequestCommentTask` | generic |
| MR opened (webhook) + `JUNIE_CUSTOM_PROMPT=code-review` | `customPrompt` set | `MergeRequestEventTask` | code-review |
| MR opened, no custom prompt | — | `FailedTaskExtractionResult` | nothing |
| MR updated/reopened/closed/merged | cleanup rule / no custom prompt | (usually nothing) | nothing |

## 5. Note on trigger phrases

The "command" phrases are defined as loose, case-insensitive regexes
(`constants/gitlab.ts:5-15`):

- `code-review` → `CODE_REVIEW_TRIGGER_PHRASE_REGEXP = /code-review/i`
- `fix-ci` → `FIX_CI_TRIGGER_PHRASE_REGEXP = /fix-ci/i`
- `minor-fix` → `MINOR_FIX_TRIGGER_PHRASE_REGEXP = /minor-fix/i`

Detection is a substring match (`isMRCommandEvent`, `context.ts:277`) against **either** the
custom prompt **or** the raw comment text. This means the literal phrases can appear anywhere in
the comment, not just at the start.

The `minor-fix` extraction is stricter: it captures text *after* `minor-fix` using
`/(minor-fix)\s+(.+)/i` (`gitlab-prompt-formatter.ts:362`), so `minor-fix` must be followed by a
space and a request to be useful (otherwise `userRequest` is `undefined` and a generic minor-fix
prompt is used).

## 6. How triggering affects the prompt

- **Issue comment** → generic prompt with `<issue_info>` + discussions.
- **MR comment (generic)** → generic prompt with `<merge_request_info>` + commits + discussions + changed files.
- **MR comment `code-review`** → a `codeReviewTask` with a `diffCommand` (`git diff origin/<target>...`)
  and the MR-context text as `description`. Junie is run in "code review" mode (see 04).
- **MR comment `fix-ci`** → the whole generic prompt is **replaced** by `createFixCIFailuresPrompt(...)`.
- **MR comment `minor-fix`** → the whole generic prompt is **replaced** by `createMinorFixPrompt(...)`.
- **MR event `code-review`** → same code-review branch as the comment path.

Details of the prompt text are in [03 — Prompts](./03-prompts.md).
