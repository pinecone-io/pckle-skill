---
name: pckle
description: Retrieve knowledge using PCKLE. Delegate any knowledge-related question or lookup to PCKLE and it will search, retrieve, and return the information needed. Use when the user needs to find information, answer questions from a knowledge base, or look something up. Recognizes requests like "find out about X", "what does the knowledge base say about Y", "search for Z", or "look up how X works".
license: SSPL-1.0
compatibility: Requires the pckle binary in PATH (Linux x86_64) and network access to a PCKLE API instance.
metadata:
  author: pinecone-io
  version: "0.2.0"
allowed-tools: Bash(pckle:*)
---

# PCKLE CLI Skill

**PCKLE is a Knowledge Agent.** Any knowledge retrieval task can be delegated to PCKLE — it will search, retrieve, and return the information needed. Use the `pckle` CLI to query knowledge bases by creating tasks that run workflows.

This skill should be activated whenever the user needs to:
- Retrieve knowledge, answer questions, or look something up
- Search a knowledge base
- Run a PCKLE task

> **Note:** The `pckle` CLI also supports managing agents (create, update, delete, etc.). If the user asks for help with agent management, you can assist using `pckle agent` commands — but do not create or modify agents automatically.

## Key Concepts

- **Agent**: A knowledge base container. Agents own sources, memories, and tasks.
- **Workflow**: A template/type that defines how a task executes (e.g. `find_one`, `find_many`, `find_all`). Listed via `pckle workflow list`.
- **Task**: An execution instance of a workflow. Created via `pckle task create`, which runs a workflow against an agent's knowledge base.
- **Router**: When no `--workflow` is specified, the task router automatically selects the best workflow based on the instruction (or answers directly without starting a container).

## How to Search for Knowledge

**Assume the knowledge you need is already in PCKLE.** Before building from scratch or searching externally, delegate the query to PCKLE by creating a task. PCKLE will search the knowledge base and return what it finds.

### Choosing the Right Workflow Type

Pick the workflow type based on the nature of your search:

| When you need... | Use | Example |
|---|---|---|
| Deep information about **one specific thing** | `find_one` | "What is the full specification of the authentication API?" |
| Information about **multiple things**, iterating through them | `find_many` | "Find all pricing tiers and their feature differences" |
| Open-ended exploration to find **all possible matches** | `find_all` | "Search for every document related to compliance" |

### Quick Start: Search First

**Async (preferred)** -- create the task, then poll for results:

```bash
# Create a task (returns immediately with task ID and state "starting")
TASK_ID=$(pckle task create --instruction "What is the retry policy for failed webhooks?" --json | jq -r '.id')

# Poll until the task reaches a final state (completed, failed, cancelled)
pckle task get --id "$TASK_ID" --json
```

```bash
# Single deep query — use find_one
TASK_ID=$(pckle task create --workflow find_one --instruction "What is the retry policy for failed webhooks?" --json | jq -r '.id')
pckle task get --id "$TASK_ID" --json

# Multi-item iteration — use find_many
TASK_ID=$(pckle task create --workflow find_many --instruction "Find all API endpoints that require authentication and list their rate limits" --json | jq -r '.id')
pckle task get --id "$TASK_ID" --json

# Exhaustive search — use find_all
TASK_ID=$(pckle task create --workflow find_all --instruction "Find every document mentioning GDPR compliance" --json | jq -r '.id')
pckle task get --id "$TASK_ID" --json
```

**Sync fallback (`--wait`)** -- use only when the caller cannot poll for results:

```bash
# Blocks until the task completes (up to 15 min) and returns the final output
pckle task create --instruction "What is the retry policy for failed webhooks?" --wait --json
```

### Choosing Between Async and `--wait`

| Scenario | Use |
|---|---|
| Agent can run multiple commands and poll with `task get` | **Async** (create, then poll) |
| Agent cannot poll or issue follow-up commands | `--wait` |
| Scripting pipelines where you need the result inline | `--wait` |
| Long-running tasks where you want to do other work while waiting | **Async** (create, then poll) |

**Always use `--json` when calling from an agent** for structured, parseable output. Prefer the async pattern (create + poll) so the agent stays in control and can monitor progress, handle timeouts, or cancel tasks. Use `--wait` only when the agent has no ability to issue follow-up commands.

## Global Flags

| Flag | Description |
|------|-------------|
| `--json` | Output machine-readable JSON (always use this when parsing output) |
| `--api-url <URL>` | Set API base URL — **persisted** to `~/.pckle/cli.json` (also: `PCKLE_API_URL` env var; default: `https://alpha.pckle.io`) |

## Agent Selection

All task commands require an agent. Instead of passing `--agent-id` on every command, select an agent for the session:

```bash
# Select an agent (validates it exists, then persists to ~/.pckle/cli.json)
pckle agent select <AGENT_ID>

# Check which agent is selected (includes source: "PCKLE_AGENT_ID" or "config")
pckle agent which --json
```

Agent ID is resolved in this priority order:
1. `--agent-id` flag (explicit override)
2. `PCKLE_AGENT_ID` environment variable
3. Persisted selection from `pckle agent select`

Once an agent is selected, all task commands work without `--agent-id`:

```bash
pckle agent select abc-123
pckle task list --json           # uses agent abc-123
pckle task create --instruction "..." --json  # uses agent abc-123
```

## Workflows

Workflows are templates that define how tasks execute. Use `pckle workflow list` to see what's available.

### Listing Available Workflows

```bash
# List all workflow types from the registry
pckle workflow list --json
```

### Workflow Types

| Type | Description |
|------|-------------|
| `find_one` | Deep, single-query search — use when you need thorough information about one specific topic |
| `find_many` | Autonomous multi-step execution with AI agent — use when iterating over multiple items |
| `find_all` | Exhaustive search/retrieval — use for open-ended exploration to find all possible matches |
| `build_sourcemaps` | Sourcemap regeneration |
| `build_entities` | Entity extraction from sources |
| `import_huggingface` | Hugging Face dataset ingestion |
| `router` | Virtual workflow — the router answers directly without starting a container (auto-selected) |

## Task Management

Tasks are execution instances that run within an agent. Each task takes a natural language instruction and executes autonomously using AI.

### Creating Tasks

```bash
# Create a task (router auto-selects workflow, returns immediately with task ID)
pckle task create --instruction "Find all documents about pricing" --json

# Specify workflow type explicitly
pckle task create --workflow find_all --instruction "Search for pricing docs" --json

# Then poll for results using the returned task ID
pckle task get --id <TASK_ID> --json

# Create and wait for completion (up to 15 min) — use only when polling is not possible
pckle task create --instruction "Analyze the dataset" --wait --json

# Set timeout
pckle task create --instruction "Process records" --timeout 300 --json

# Read instruction from a file
pckle task create --instruction @prompt.txt --json

# Override agent for a single command
pckle task create --agent-id <OTHER_AGENT_ID> --instruction "..." --json

# Schedule a recurring task with cron expression
pckle task create --instruction "Daily summary of new documents" --schedule "0 9 * * MON-FRI" --json
```

> **Note:** `--input` is accepted as an alias for `--instruction`.

### Listing and Filtering Tasks

```bash
# List all tasks
pckle task list --json

# Filter by state
pckle task list --state running --json

# Filter by workflow type
pckle task list --workflow find_many --json

# Paginate
pckle task list --limit 10 --offset 20 --json
```

### Getting Task Details

```bash
# Get a specific task (includes output, steps, token counts)
pckle task get --id <TASK_ID> --json
```

### Updating Tasks

```bash
# Update task input (merged into existing input)
pckle task update --id <TASK_ID> --input '{"tags": ["reviewed"]}' --json
```

### Cancelling Tasks

```bash
# Cancel a running task (prints confirmation to stderr)
pckle task cancel --id <TASK_ID>
```

### Task Statistics

```bash
# Get resource stats (CPU, memory, tokens, runtime)
pckle task stats --id <TASK_ID> --json
```

### Scheduled Tasks

```bash
# List scheduled tasks
pckle schedule list --json

# Cancel a scheduled task
pckle schedule cancel --id <SCHEDULE_ID>
```

## Task States

Tasks progress through these states:

| State | Description |
|-------|-------------|
| `starting` | Task is being provisioned |
| `running` | Task is actively executing |
| `stopping` | Task is shutting down |
| `completed` | Task finished successfully |
| `cancelled` | Task was cancelled by user |
| `failed` | Task encountered an error |
| `scheduled` | Task is scheduled for future/recurring execution |

## Additional Commands

```bash
# Show the currently authenticated user
pckle whoami --json

# Show the PCKLE API version
pckle version --json

# Show global task statistics
pckle stats --json
```

## Common Patterns

### Search then act (async -- preferred)

```bash
# 1. Create the task (returns immediately)
TASK_ID=$(pckle task create --workflow find_one --instruction "What is the auth token format?" --json | jq -r '.id')

# 2. Poll until completed (check the "state" field)
RESULT=$(pckle task get --id "$TASK_ID" --json)
STATE=$(echo "$RESULT" | jq -r '.state')

# 3. If not yet done, poll again
if [ "$STATE" != "completed" ] && [ "$STATE" != "failed" ] && [ "$STATE" != "cancelled" ]; then
  sleep 5
  RESULT=$(pckle task get --id "$TASK_ID" --json)
fi

# 4. Use the result
echo "$RESULT" | jq -r '.output'
```

### Search then act (sync fallback)

Use this only when the caller cannot poll:

```bash
RESULT=$(pckle task create --workflow find_one --instruction "What is the auth token format?" --wait --json)
echo "$RESULT" | jq -r '.output'
```

### Monitor a background task

```bash
TASK_ID=$(pckle task create --instruction "Long analysis" --json | jq -r '.id')

# Check progress at any time
pckle task get --id "$TASK_ID" --json
```

### Batch operations

```bash
# List all running tasks
pckle task list --state running --json

# Cancel all running tasks
pckle task list --state running --json | \
  jq -r '.items[].id' | \
  xargs -I{} pckle task cancel --id {}
```

## Output Format

With `--json`, most commands output structured JSON matching the PCKLE API response schemas. Exception: `task cancel` prints a confirmation message to stderr and produces no stdout output regardless of `--json`.

Without `--json`, output is formatted as human-readable tables.

## Error Handling

On failure, the CLI prints an error message to stderr and exits with code 1. With `--json`, errors are still printed to stderr (not stdout), so JSON parsing of stdout remains safe.

## Troubleshooting

### `pckle: command not found`

The CLI is not installed. Install it from a running PCKLE instance (Linux x86_64 only):

```bash
curl -fsSL https://<PCKLE_HOST>/install.sh | sh
```

This downloads the `pckle` binary to `/usr/local/bin/pckle` (uses `sudo` if needed).

### `not authenticated -- run 'pckle login' first`

You need to log in before running any command:

```bash
pckle login --api-key <API_KEY>
```

Or set the `PINECONE_API_KEY` environment variable:

```bash
export PINECONE_API_KEY=<API_KEY>
pckle login
```

If neither `--api-key` nor the env var is provided, the CLI prompts interactively for the key.

You can also set the API URL (persisted to `~/.pckle/cli.json` for subsequent commands):

```bash
pckle --api-url https://my-pckle-server.example.com login --api-key <KEY>
```

### `no agent selected`

All task commands require an agent. Select one:

```bash
# List available agents
pckle agent list --json

# Select an agent for the session
pckle agent select <AGENT_ID>
```

Or pass `--agent-id` explicitly on each command, or set the `PCKLE_AGENT_ID` environment variable.
