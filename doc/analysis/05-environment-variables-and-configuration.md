# 05 — Environment variables & configuration reference

All wrapper configuration flows through environment variables, declared in `src/webhook-env.ts`.
Each `Variable` reads `process.env[key]`; the `mappedValue` field is the GitLab custom-webhook-
template placeholder used during `init` to map webhook payload fields into pipeline variables.

## Full variable table

Source: `src/webhook-env.ts:74-118`.

| Variable | Type | Mapped webhook value | Purpose |
|----------|------|----------------------|---------|
| `JUNIE_WEBHOOK` | string | `"true"` | Marker; used to detect the Junie webhook among existing hooks. |
| `USE_MCP` | boolean | `"true"` | Enable GitLab MCP (`~/.junie/mcp/mcp.json` + MCP note in prompt). |
| `USE_PIPELINE_REDIRECT` | boolean | `"false"` | Redirect pipeline into the user project (`executor.ts:51`). |
| `JUNIE_MODEL` | string | — | Passed to `junie --model=…`. |
| `JUNIE_GUIDELINES_FILENAME` | string | — | Passed to `junie --guidelines-file=…`. |
| `JUNIE_CUSTOM_PROMPT` | string | — | Overrides/prepends user instruction; required for MR-event runs. |
| `CI_PROJECT_ID` (`junieProjectId`) | number | — | The Junie Workspace project ID. |
| `CI_DEFAULT_BRANCH` | string | — | Junie project default branch. |
| `CI_API_V4_URL` | string | — | GitLab API base URL. |
| `PROJECT_ID` (`projectId`) | number | `{{project.id}}` | The **user** project that fired the event. |
| `CI_PIPELINE_ID` (`pipelineId`) | number | — | Current pipeline id (for cleanup / redirect). |
| `PROJECT_ACCESS_TOKEN_ACCESS_LEVEL` | access-level | — | Access level for the token created during `init` (default `MAINTAINER`). |
| `EVENT_KIND` | string | `{{object_kind}}` | `note` or `merge_request`. |
| `GITLAB_TOKEN_FOR_JUNIE` | string | — | GitLab token used by the wrapper (API, glab, git remote, MCP). |
| `JUNIE_API_KEY` | string | — | Junie auth token. |
| `ISSUE_ID` | number | `{{issue.iid}}` | Issue IID (note events on issues). |
| `COMMENT_TEXT` | string | `{{object_attributes.note}}` | Comment body (note events). |
| `ISSUE_URL` | string | `{{issue.url}}` | Issue URL. |
| `MERGE_REQUEST_ID` | number | `{{merge_request.iid}}` | MR IID (note events on MRs). |
| `MERGE_REQUEST_SOURCE_BRANCH` | string | `{{merge_request.source_branch}}` | MR source branch. |
| `MERGE_REQUEST_TARGET_BRANCH` | string | `{{merge_request.target_branch}}` | MR target branch. |
| `DISCUSSION_ID` | string | `{{object_attributes.discussion_id}}` | Discussion/thread id. |
| `MR_EVENT_ID` | number | `{{object_attributes.iid}}` | MR IID (merge_request events). |
| `MR_EVENT_SOURCE_BRANCH` | string | `{{object_attributes.source_branch}}` | |
| `MR_EVENT_TARGET_BRANCH` | string | `{{object_attributes.target_branch}}` | |
| `MR_EVENT_TITLE` | string | `{{object_attributes.title}}` | |
| `MR_EVENT_DESCRIPTION` | string | `{{object_attributes.description}}` | |
| `MR_EVENT_ACTION` | string | `{{object_attributes.action}}` | `open`/`update`/`reopen`/`close`/`merge`. |
| `MR_EVENT_URL` | string | `{{object_attributes.url}}` | |
| `OBJECT_ID` | number | `{{object_attributes.id}}` | Note/comment id (used as `commentId`). |

## Boolean parsing note

`BooleanVariable` (`webhook-env.ts:26`) is `process.env[key] === "true"` — so `USE_MCP` and
`USE_PIPELINE_REDIRECT` must be the literal string `"true"` (any other value, including missing,
is `false`).

## Access-level parsing

`AccessLevelVariable` (`webhook-env.ts:52`) accepts either a number or a `AccessLevel` enum name
(`MAINTAINER`, `DEVELOPER`, …). The sample YAML maps `project_token_access_level` input to
`PROJECT_ACCESS_TOKEN_ACCESS_LEVEL`, defaulting to `MAINTAINER` (`script-sample.yaml:7-9, 18-20`).

## CI/CD inputs (`script-sample.yaml`)

The pipeline defines two `spec.inputs`:

- `project_token` (string) — injected as `INPUT_TOKEN` → `GITLAB_TOKEN_FOR_JUNIE` (the user
  project's access token, supplied by the webhook's `inputs[project_token]` URL variable).
- `project_token_access_level` (string, default `"MAINTAINER"`) — access level for tokens created
  during `init`.

## Non-config constants (hard-coded)

- `cacheDir = "/junieCache"` (`executor.ts:38`) — Junie `--cache-dir`, input/output files.
- `literalMentions = ['@junie', '#junie']` (`executor.ts:39`).
- `PROJECT_ACCESS_TOKEN_NAME = "Junie by JetBrains"` (`constants/gitlab.ts:17`) — the bot token
  name used both for the `@project_N_bot` mention detection and `init` token creation.
- Git remote URL built as `oauth2:<token>@<host>/<project>.git` (`git-api.ts:85`).
- MCP server: `npx -y @zereight/mcp-gitlab` (`mcp.ts`).

## Secrets (never place in user-project pipeline)

`GITLAB_TOKEN_FOR_JUNIE` is a high-privilege token. The README explicitly warns to keep it scoped
to the `init` environment so it doesn't leak into the pipelines that run actual Junie jobs
(`README.md:14`). In `script-sample.yaml` it is defined only under `junie-init`/`junie-cleanup`
(`environment: init`), while `junie-run` receives the per-project token via `$INPUT_TOKEN`
(`script-sample.yaml:67`).
