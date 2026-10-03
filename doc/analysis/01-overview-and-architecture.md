# 01 — Overview & Architecture

## What this project is

`junie-gitlab` is a **GitLab CI wrapper around the JetBrains "Junie" coding-agent CLI**.
It is *not* a GitLab integration that calls Junie directly; instead it runs as a **Node.js CLI
(`gitlab-cli-wrapper`) inside a GitLab pipeline job**, and from there shells out to the real
`junie` binary (installed in the Docker image).

The core idea:

1. A webhook in a *user project* fires on note/MR events.
2. The webhook triggers a pipeline in a dedicated **"Junie Workspace" project**.
3. That pipeline runs the wrapper (`node dist/cli.js run`), which:
   - extracts the event context from environment variables,
   - fetches rich context (MR/issue/commits/discussions/diffs) from the GitLab API,
   - assembles a **prompt**,
   - invokes the **`junie` CLI** with that prompt,
   - collects the result and posts feedback (comments / MRs) back to the user project.

## Component map

| File | Role |
|------|------|
| `src/cli.ts` | Entry point. Defines `init`, `run`, `cleanup` subcommands. |
| `src/executor.ts` | Orchestrates a `run`: task extraction → prompt generation → `junie` invocation → git push/MR → feedback. |
| `src/context.ts` | Parses webhook env vars into a typed `GitLabExecutionContext`. |
| `src/webhook-env.ts` | Declares every environment variable the wrapper reads (with GitLab custom-webhook-template mappings). |
| `src/models/task-extraction-result.ts` | The `IssueCommentTask` / `MergeRequestCommentTask` / `MergeRequestEventTask` classes; decide branch checkout, prompt, title, feedback. |
| `src/utils/gitlab-prompt-formatter.ts` | **Builds the actual prompt text** sent to Junie. |
| `src/constants/gitlab.ts` | Trigger phrases (`code-review`, `fix-ci`, `minor-fix`), message templates, and the two fixed prompt templates. |
| `src/api/gitlab-data-fetcher.ts` | Fetches MR/issue data (parallel) for prompt context. |
| `src/api/gitlab-api.ts` | All GitLab REST calls (webhooks, tokens, pipelines, MRs, notes). |
| `src/api/git-api.ts` | `simple-git` wrapper for checkout / commit / push. |
| `src/mcp.ts` | Writes the `~/.junie/mcp/mcp.json` config so Junie can use the GitLab MCP server. |
| `src/feedback.ts` | Posts "started"/"finished" comments & reactions back to GitLab. |
| `src/initializer.ts` | `init` — creates webhook + trigger token + project access token per project. |
| `src/utils/sanitizer.ts` | Strips prompt-injection vectors and redacts tokens from user content. |
| `src/utils/attachment-downloader.ts` | Downloads GitLab uploads (`/uploads/...`) and rewrites them to local paths in the prompt. |

## Execution flow (a `run`)

`cli.ts:run` → `extractGitLabContext()` → `executor.execute()`:

1. `extractTaskFromEnv()` (`executor.ts:170`) builds a `TaskExtractionResult`:
   - detects the event kind (`note` on issue / `note` on MR / `merge_request`),
   - decides whether it's a "successful task" or a "failed/no-task" (no mention, no custom prompt, etc.).
2. If successful, and `USE_PIPELINE_REDIRECT=true`, it redirects the pipeline into the user project
   (`executor.ts:51`) instead of running locally.
3. Configures `glab` auth (`executor.ts:91`) and, if `USE_MCP`, writes the MCP config (`mcp.ts`).
4. Posts "started" feedback (`executor.ts:110`).
5. Checks out the target branch (`checkoutBranch`).
6. `generateJuniePrompt()` builds the prompt (a `JunieTask`), then `runJunie()` invokes the CLI.
7. Parses Junie's JSON output; commits/pushes either as a new MR or appends to the same branch
   (controlled by `--mr-mode`).
8. Posts "finished" feedback.

## Two deployment modes

- **Default (pipeline runs in the Junie Workspace project):** the wrapper cleans the workdir
  (`rm -rf * .*`) and checks out the *user* project's code via the GitLab API (`checkoutBranch`),
  then operates in-place. Configured by `script-sample.yaml`.
- **Pipeline redirect (`USE_PIPELINE_REDIRECT=true`):** the wrapper temporarily sets the user
  project's `ci_config_path` to the wrapper's `child-pipeline.yml`, re-triggers the pipeline
  *inside the user project*, then restores the config path (`executor.ts:51-86`).

## The `junie` CLI

Installed in the runtime image via
`curl -fsSL https://junie.jetbrains.com/install.sh | bash` (`Dockerfile:37`) and placed on
`PATH` at `/root/.local/bin`. The wrapper drives it through stdin/stdout with JSON input/output
formats (see [04 — Junie CLI invocation](./04-junie-cli-invocation.md)).
