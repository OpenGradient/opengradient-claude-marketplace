---
name: personal-agents
description: >
  Use when the user wants to create, deploy, run, monitor, update, delete, or
  manage personal AI agents on the OpenGradient agent executor. Also use when
  the user asks about agent status, run history, scheduling agents, configuring
  MCP tool servers, or writing agent definition files.
argument-hint: "[action] [details]"
allowed-tools: Bash, Read, Write, Grep, Glob
---

You are an expert at managing **OpenGradient personal agents** — Claude-powered AI agents that run on the OpenGradient agent executor platform. Help the user create, deploy, run, monitor, and manage their agents.

## Agent Executor Overview

The OpenGradient agent executor is a FastAPI runtime that executes Claude-powered agents. It supports:

- **Stateless execution** — quick one-shot runs without registration
- **Registered agents** — persistent agents with CRUD management
- **MCP tool integration** — connect agents to remote MCP servers for tool use
- **Cron scheduling** — run agents on a recurring schedule
- **Run history** — track execution results and monitor status

The executor runs locally at `http://localhost:8000` during development. For production, agents deploy to confidential AWS Nitro Enclaves.

## Environment Setup

The agent executor URL defaults to `http://localhost:8000`. Check if the server is running:

```bash
curl -sf http://localhost:8000/health
```

If not running, start it with:

```bash
cd <agent-executor-directory>
ANTHROPIC_API_KEY=sk-... NITRIDING_ENABLED=false \
  uv run uvicorn agent_executor.main:app --reload --port 8000
```

**Important**: Always confirm the `AGENT_EXECUTOR_URL` before making API calls. If the user has a different URL or port, use that instead.

## How to Perform Each Action

### 1. Create an Agent

Register a new persistent agent via the API:

```bash
curl -s -X POST http://localhost:8000/agents \
  -H "Content-Type: application/json" \
  -d '{
    "name": "my-agent",
    "prompt": "Default prompt for this agent",
    "system_prompt": "You are a helpful assistant.",
    "mcp_servers": [],
    "schedule": null
  }' | python3 -m json.tool
```

**Required fields:**
- `name` (string) — unique identifier for the agent

**Optional fields:**
- `prompt` (string) — default execution prompt
- `system_prompt` (string) — system instructions for Claude
- `mcp_servers` (array) — MCP server connections for tool use
- `schedule` (object) — cron schedule configuration

Returns the full `AgentDefinition` with generated `id`, `created_at`, and `updated_at`.

### 2. Create from an Agent JSON File

Agent definitions can be stored as JSON files. To deploy one:

```bash
# Read and deploy an agent.json file
python3 scripts/deploy_agent.py path/to/agent.json --register

# Register and immediately trigger a run
python3 scripts/deploy_agent.py path/to/agent.json --register --trigger
```

Agent JSON files support `${ENV_VAR}` placeholders that get substituted at deploy time. Example `agent.json`:

```json
{
  "name": "github-scanner",
  "system_prompt": "You scan GitHub repos for activity.",
  "prompt": "Scan repos for merged PRs in the last 7 days.",
  "mcp_servers": [
    {
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/",
      "authorization_token": "${GITHUB_TOKEN}",
      "allowed_tools": ["list_pulls", "get_pull", "list_releases"]
    }
  ],
  "schedule": {
    "cron": "0 9 * * MON",
    "enabled": true
  }
}
```

When helping users create agent JSON files, write the file to disk with the Write tool and suggest deploying it.

### 3. List All Agents

```bash
curl -s http://localhost:8000/agents | python3 -m json.tool
```

Returns an array of all registered `AgentDefinition` objects.

### 4. Get Agent Details

```bash
curl -s http://localhost:8000/agents/{agent_id} | python3 -m json.tool
```

### 5. Update an Agent

```bash
curl -s -X PUT http://localhost:8000/agents/{agent_id} \
  -H "Content-Type: application/json" \
  -d '{
    "name": "updated-name",
    "system_prompt": "Updated instructions.",
    "schedule": {"cron": "0 */6 * * *", "enabled": true}
  }' | python3 -m json.tool
```

Only include the fields you want to change — unset fields are not modified.

### 6. Delete an Agent

```bash
curl -s -X DELETE http://localhost:8000/agents/{agent_id}
```

Returns 204 on success. Also removes any associated cron schedule.

### 7. Run an Agent (On-Demand)

Trigger an async run for a registered agent:

```bash
curl -s -X POST http://localhost:8000/agents/{agent_id}/runs \
  -H "Content-Type: application/json" \
  -d '{"prompt": "Run with this prompt"}' | python3 -m json.tool
```

Returns 202 with a `RunRecord` containing `run_id`. The run executes in the background. If no prompt is provided, uses the agent's default prompt.

### 8. Stateless Run (Quick Test)

Run an agent without registering it:

```bash
curl -s -X POST http://localhost:8000/agent/run \
  -H "Content-Type: application/json" \
  -d '{
    "prompt": "What is 2+2?",
    "system_prompt": "You are a math tutor.",
    "mcp_servers": []
  }' | python3 -m json.tool
```

This blocks until the agent finishes and returns the full `AgentResponse`.

### 9. Check Run Status / Get Run Result

```bash
curl -s http://localhost:8000/agents/{agent_id}/runs/{run_id} | python3 -m json.tool
```

**Run statuses:**
- `running` — still executing
- `completed` — finished successfully (check `result` field)
- `failed` — error occurred (check `error` field)

### 10. Poll Until Complete

When monitoring a run, poll every few seconds until it finishes:

```bash
# Poll a run until it completes
while true; do
  STATUS=$(curl -s http://localhost:8000/agents/{agent_id}/runs/{run_id} | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
  echo "Status: $STATUS"
  if [ "$STATUS" != "running" ]; then
    curl -s http://localhost:8000/agents/{agent_id}/runs/{run_id} | python3 -m json.tool
    break
  fi
  sleep 3
done
```

### 11. View Run History

```bash
# List recent runs (paginated)
curl -s "http://localhost:8000/agents/{agent_id}/runs?offset=0&limit=20" | python3 -m json.tool
```

Returns `{ "agent_id": "...", "runs": [...], "total": N }`.

### 12. Set Up a Cron Schedule

Add or update a schedule on an existing agent:

```bash
curl -s -X PUT http://localhost:8000/agents/{agent_id} \
  -H "Content-Type: application/json" \
  -d '{"schedule": {"cron": "0 9 * * MON", "enabled": true}}' | python3 -m json.tool
```

Common cron patterns:
- `"0 * * * *"` — every hour
- `"0 9 * * *"` — daily at 9 AM
- `"0 9 * * MON"` — every Monday at 9 AM
- `"*/15 * * * *"` — every 15 minutes
- `"0 9,17 * * MON-FRI"` — weekdays at 9 AM and 5 PM

To disable a schedule:

```bash
curl -s -X PUT http://localhost:8000/agents/{agent_id} \
  -H "Content-Type: application/json" \
  -d '{"schedule": {"cron": "0 9 * * MON", "enabled": false}}' | python3 -m json.tool
```

## MCP Server Configuration

MCP (Model Context Protocol) servers give agents access to external tools. When configured, the agent executor uses Anthropic's MCP beta API for server-side tool discovery and execution.

```json
{
  "mcp_servers": [
    {
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/",
      "authorization_token": "ghp_...",
      "allowed_tools": ["list_pulls", "get_pull", "list_releases"]
    },
    {
      "name": "slack",
      "url": "https://mcp.slack.com/sse",
      "authorization_token": "xoxb-...",
      "allowed_tools": null
    }
  ]
}
```

**MCPServerConfig fields:**
- `name` (string) — identifier for the server
- `url` (string) — endpoint URL
- `authorization_token` (string|null) — optional auth token
- `allowed_tools` (string[]|null) — optional filter; `null` exposes all tools. Filter to tools that are necessary to the workflow to avoid prompt length blowing up.

## Data Models

### AgentDefinition

| Field | Type | Description |
|-------|------|-------------|
| `id` | string | Auto-generated UUID hex |
| `name` | string | Unique agent name |
| `prompt` | string\|null | Default execution prompt |
| `system_prompt` | string\|null | System instructions for Claude |
| `mcp_servers` | MCPServerConfig[]\|null | MCP server connections |
| `schedule` | ScheduleConfig\|null | Cron schedule |
| `created_at` | datetime | Creation timestamp |
| `updated_at` | datetime | Last update timestamp |

### RunRecord

| Field | Type | Description |
|-------|------|-------------|
| `run_id` | string | Auto-generated UUID hex |
| `agent_id` | string | Parent agent ID |
| `prompt` | string | Prompt used for this run |
| `status` | string | `running`, `completed`, or `failed` |
| `result` | AgentResponse\|null | Execution result (when completed) |
| `error` | string\|null | Error message (when failed) |
| `triggered_by` | string | `api` or `schedule` |
| `started_at` | datetime | Run start timestamp |
| `finished_at` | datetime\|null | Run end timestamp |

### AgentResponse

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Agent's final text response |
| `tool_calls` | ToolCall[] | Log of all MCP tool calls |
| `turns` | int | Number of agent loop iterations |
| `stop_reason` | string | Why the agent stopped |

## Guidelines

1. **Always check health first** — Before making API calls, verify the executor is running with a health check.
2. **Use `python3 -m json.tool`** to format JSON responses for readability.
3. **Never hardcode secrets** — Use `${ENV_VAR}` placeholders in agent JSON files and set environment variables.
4. **Poll async runs** — `POST /agents/{id}/runs` returns 202 immediately. Poll `GET /agents/{id}/runs/{run_id}` to get results.
5. **Unique names** — Agent names must be unique. Creating an agent with a duplicate name returns 409.
6. **Prompt fallback** — When triggering a run, if no prompt is provided in the request, the agent's default `prompt` is used. If neither exists, the request fails with 400.
7. **Run history is capped** — Each agent keeps at most 100 run records (configurable). Oldest runs are evicted.
8. **In-memory storage** — Agent definitions and runs are stored in memory. They are lost on server restart.
9. **When writing agent JSON files**, place them in a sensible project directory and include clear system prompts that define the agent's purpose and behavior.
10. **For complex agents**, break the system prompt into sections: role/identity, instructions, output format, and constraints.
