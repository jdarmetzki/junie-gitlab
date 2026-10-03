# 06 — Index

In-depth analysis of the `junie-gitlab` integration.

| Doc | Contents |
|-----|----------|
| [01 — Overview & Architecture](./01-overview-and-architecture.md) | What the project is, component map, execution flow, deployment modes. |
| [02 — Triggers](./02-triggers.md) | Webhook/pipeline rules, task-extraction, trigger → task → prompt matrix. |
| [03 — Prompts](./03-prompts.md) | Prompt templates (`generic`, `code-review`, `fix-ci`, `minor-fix`), sections, appended notes, sanitization, and how to influence them. |
| [04 — Junie CLI invocation](./04-junie-cli-invocation.md) | The exact `junie` command line, input/output JSON, model/guidelines args, MCP config. |
| [05 — Environment variables & configuration](./05-environment-variables-and-configuration.md) | Complete env-var reference, CI/CD inputs, and hard-coded constants. |
| [07 — Concrete examples: new MR on a branch → `main`](./07-example-mr-to-main.md) | Realistic full prompt + JSON input + command for the `merge_request` event. |

## TL;DR

- The wrapper is a **Node.js CLI running in a GitLab pipeline** that shells out to the real
  **`junie` binary** (`executor.ts:runJunie`).
- It is triggered by **GitLab webhooks** (note + merge-request events) that fire pipelines in a
  dedicated Junie Workspace project.
- Three event kinds map to three task classes: **issue comment**, **MR comment**, **MR event**.
- Comment content selects the prompt: generic, `#junie code-review`, `#junie fix-ci`,
  `#junie minor-fix <request>`.
- The prompt is assembled by `GitLabPromptFormatter` (rich MR/issue/commits/discussions/changes
  context) or replaced by fixed templates (`fix-ci`, `minor-fix`) in `constants/gitlab.ts`.
- The actual command is:

  ```bash
  junie --auth "$JUNIE_API_KEY" --cache-dir=/junieCache \
        --output-format=json --input-format=json \
        --json-output-file=/junieCache/junie_output.json \
        [--model="$JUNIE_MODEL"] [--guidelines-file="$JUNIE_GUIDELINES_FILENAME"] \
        < /junieCache/junie_input.json
  ```
