# 04 — Junie CLI invocation

The wrapper never speaks to Junie over a network API — it invokes the **`junie` command-line
binary** as a subprocess. All of this happens in `executor.ts:runJunie` (lines 226-255).

## The command

`runJunie` writes the task to a temp file, then runs (with `execSync` via `runCommand`):

```bash
junie --auth "${token}" --cache-dir="/junieCache" --output-format="json" --input-format="json" --json-output-file="/junieCache/junie_output.json"${modelArg}${guidelinesArg} < "/junieCache/junie_input.json"
```

Breaking it down (see `executor.ts:226-255`):

| Argument | Source | Meaning |
|----------|--------|---------|
| `junie` | `PATH` (installed at `/root/.local/bin`, `Dockerfile:37,40`) | the Junie binary |
| `--auth "${token}"` | `JUNIE_API_KEY` (`context.junieApiKey`) | Junie auth token (usually `perm-…`) |
| `--cache-dir="/junieCache"` | hard-coded `cacheDir` const (`executor.ts:38`) | working/cache dir |
| `--output-format="json"` | hard-coded | JSON output |
| `--input-format="json"` | hard-coded | JSON input on stdin |
| `--json-output-file=".../junie_output.json"` | hard-coded | where results are written |
| `--model="${model}"` | `JUNIE_MODEL` (optional; only if set) | e.g. `sonnet`, `opus`, `gemini-flash`, `gpt` |
| `--guidelines-file="${filename}"` | `JUNIE_GUIDELINES_FILENAME` (optional) | project guidelines file |
| `< junie_input.json` | stdin redirect | the `JunieTask` JSON, avoiding `ARG_MAX` |

### Optional args (`executor.ts:239-240`)

```ts
const modelArg = model ? ` --model="${model}"` : "";
const guidelinesArg = guidelinesFilename ? ` --guidelines-file="${guidelinesFilename}"` : "";
```

So `--model` and `--guidelines-file` are appended only when the corresponding env vars are set.

## The input (`junie_input.json`)

Written as `JSON.stringify(junieTask, null, 2)` (`executor.ts:235`), where `junieTask` is the
result of `generateJuniePrompt()`:

- **Generic tasks** (`task`): `{ "task": "<full prompt text>" }`
- **Code-review tasks** (`codeReviewTask`): `{ "codeReviewTask": { "diffCommand": "git diff origin/<target>...", "description": "<prompt text>" } }`

> Note: for the `--input-format=json` path, `task` and `codeReviewTask` are the JSON top-level
> keys the wrapper emits. They are consumed by Junie's JSON input parser.

## The output (`junie_output.json`)

`runJunie` reads the output file back and `JSON.parse`s it (`executor.ts:248, 122`). It expects
a top-level object with at least:

```ts
{ "result": string | null, "taskName": string | null }
```

These are used to build the commit message (`generated changes by Junie: {taskName}`,
`executor.ts:129`) and the "finished" feedback text.

## Model selection (what `--model` maps to)

`JUNIE_MODEL` is passed straight through to `--model`. The sample config sets `JUNIE_MODEL: "sonnet"`
(`script-sample.yaml:70`, `child-pipeline.yml:18`) "to use Claude instead of Gemini for MCP
compatibility". Valid values per the README: `sonnet`, `opus`, `gemini-flash`, `gpt`
(`README.md:49`). It is otherwise unvalidated by the wrapper — Junie interprets it.

## MCP config (feeds Junie's tools)

When `USE_MCP=true`, the wrapper writes `~/.junie/mcp/mcp.json` (`mcp.ts`) before invoking Junie:

```json
{
  "mcpServers": {
    "gitlab": {
      "command": "npx",
      "args": ["-y", "@zereight/mcp-gitlab"],
      "env": {
        "GITLAB_PERSONAL_ACCESS_TOKEN": "<gitlabToken>",
        "GITLAB_API_URL": "<apiV4Url>",
        "GITLAB_READ_ONLY_MODE": "false",
        "USE_GITLAB_WIKI": "false",
        "USE_MILESTONE": "false",
        "USE_PIPELINE": "true",
        "GITLAB_ALLOWED_PROJECT_IDS": "<projectId>"
      }
    }
  }
}
```

This is what enables the `gitlab.*` MCP tools referenced by the `fix-ci`/`code-review` prompts
(`gitlab.list_pipeline_jobs`, `gitlab.get_pipeline_job_output`, `gitlab.get_merge_request_diffs`,
`gitlab.create_inline_comment`, etc.).

## Full lifecycle in the job

`script-sample.yaml` `junie-run`:

1. `rm -rf * .*` — clean the workdir.
2. `node /app/dist/cli.js run --verbose` — the wrapper (above).
3. `after_script` copies `/junieCache` and `~/.junie/{logs,sessions}` into `junie-artifacts/…`
   and exports `wrapper-outputs.env` as a dotenv artifact (used by the cleanup stage's
   `DELETE_PIPELINE` flag).

## Summary of env → CLI mapping

| Env var | CLI effect |
|---------|------------|
| `JUNIE_API_KEY` | `--auth` token |
| `JUNIE_MODEL` | `--model=<value>` (only if set) |
| `JUNIE_GUIDELINES_FILENAME` | `--guidelines-file=<value>` (only if set) |
| `USE_MCP` | writes `~/.junie/mcp/mcp.json` before the call |
| `GITLAB_TOKEN_FOR_JUNIE` | used for `glab` auth and MCP `GITLAB_PERSONAL_ACCESS_TOKEN` |
