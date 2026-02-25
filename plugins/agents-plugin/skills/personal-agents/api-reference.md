# Personal Agents — API Reference

Complete API reference for the OpenGradient agent executor. Use this alongside
the main SKILL.md when you need specifics about request/response formats,
error codes, or edge cases.

---

## Base URL

| Environment | URL |
|-------------|-----|
| Local dev | `http://localhost:8000` |
| Custom | Set via `AGENT_EXECUTOR_URL` env var |

---

## Endpoints

### `GET /health`

Health check. Returns `{"status": "ok"}` when the server is ready.

---

### `POST /agent/run`

Stateless agent execution. Runs the agent loop synchronously and returns the
result. No registration required.

**Request body:**

```json
{
  "prompt": "What is 2+2?",
  "system_prompt": "You are a math tutor.",
  "mcp_servers": [
    {
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/",
      "authorization_token": "ghp_...",
      "allowed_tools": ["list_pulls"]
    }
  ]
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | yes | User prompt for the agent |
| `system_prompt` | string | no | System instructions |
| `mcp_servers` | MCPServerConfig[] | no | MCP servers for tool access |

**Response (200):**

```json
{
  "response": "2+2 equals 4.",
  "tool_calls": [],
  "turns": 1,
  "stop_reason": "end_turn"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `response` | string | Agent's final text response |
| `tool_calls` | ToolCall[] | All MCP tool calls made during execution |
| `turns` | int | Number of agent loop iterations |
| `stop_reason` | string | `end_turn`, `max_tokens`, `mcp_tool_use`, etc. |

**Timeout:** This is a synchronous call. For long-running agents, use registered agents with async runs instead.

---

### `POST /agents`

Register a new persistent agent.

**Request body:**

```json
{
  "name": "my-agent",
  "prompt": "Default prompt text",
  "system_prompt": "You are a helpful assistant.",
  "mcp_servers": [
    {
      "name": "github",
      "url": "https://api.githubcopilot.com/mcp/",
      "authorization_token": "ghp_...",
      "allowed_tools": ["list_pulls", "get_pull"]
    }
  ],
  "schedule": {
    "cron": "0 9 * * MON",
    "enabled": true
  }
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | yes | Unique agent name |
| `prompt` | string | no | Default prompt for runs |
| `system_prompt` | string | no | System instructions |
| `mcp_servers` | MCPServerConfig[] | no | MCP server connections |
| `schedule` | ScheduleConfig | no | Cron schedule |

**Response (201):** Full `AgentDefinition` with generated `id`, `created_at`, `updated_at`.

**Errors:**
- `409 Conflict` — Agent with this name already exists

---

### `GET /agents`

List all registered agents.

**Response (200):** Array of `AgentDefinition` objects.

```json
[
  {
    "id": "a1b2c3d4...",
    "name": "my-agent",
    "prompt": "Default prompt",
    "system_prompt": "System instructions",
    "mcp_servers": [],
    "schedule": null,
    "created_at": "2025-01-01T00:00:00",
    "updated_at": "2025-01-01T00:00:00"
  }
]
```

---

### `GET /agents/{agent_id}`

Get a single agent by ID.

**Response (200):** `AgentDefinition` object.

**Errors:**
- `404 Not Found` — Agent does not exist

---

### `PUT /agents/{agent_id}`

Update an existing agent. Only include fields you want to change.

**Request body:**

```json
{
  "name": "new-name",
  "system_prompt": "Updated instructions.",
  "schedule": {"cron": "0 */6 * * *", "enabled": true}
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `name` | string | no | New agent name |
| `prompt` | string | no | New default prompt |
| `system_prompt` | string | no | New system instructions |
| `mcp_servers` | MCPServerConfig[] | no | New MCP server config |
| `schedule` | ScheduleConfig | no | New schedule config |

**Response (200):** Updated `AgentDefinition`.

**Errors:**
- `404 Not Found` — Agent does not exist
- `409 Conflict` — New name conflicts with existing agent

**Side effects:**
- If `schedule` is provided and `enabled: true`, the cron job is created or updated
- If `schedule` is removed or `enabled: false`, the cron job is removed

---

### `DELETE /agents/{agent_id}`

Delete an agent and its associated schedule.

**Response:** `204 No Content`

**Errors:**
- `404 Not Found` — Agent does not exist

**Side effects:**
- Removes the agent's cron schedule if one exists
- Run history for the agent is also deleted

---

### `POST /agents/{agent_id}/runs`

Trigger an on-demand run. Returns immediately (202) while the agent executes in the background.

**Request body:**

```json
{
  "prompt": "Override prompt for this run"
}
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `prompt` | string | no | Prompt override. Falls back to agent's default prompt |

**Response (202):**

```json
{
  "run_id": "abc123...",
  "agent_id": "def456...",
  "prompt": "The prompt used",
  "status": "running",
  "result": null,
  "error": null,
  "triggered_by": "api",
  "started_at": "2025-01-01T00:00:00",
  "finished_at": null
}
```

**Errors:**
- `400 Bad Request` — No prompt provided and agent has no default prompt
- `404 Not Found` — Agent does not exist

---

### `GET /agents/{agent_id}/runs`

List run history for an agent. Paginated.

**Query parameters:**

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `offset` | int | 0 | Skip this many runs |
| `limit` | int | 20 | Max runs to return (1–100) |

**Response (200):**

```json
{
  "agent_id": "def456...",
  "runs": [
    {
      "run_id": "abc123...",
      "agent_id": "def456...",
      "prompt": "...",
      "status": "completed",
      "result": { "response": "...", "tool_calls": [], "turns": 2, "stop_reason": "end_turn" },
      "error": null,
      "triggered_by": "api",
      "started_at": "2025-01-01T00:00:00",
      "finished_at": "2025-01-01T00:01:00"
    }
  ],
  "total": 42
}
```

**Errors:**
- `404 Not Found` — Agent does not exist

---

### `GET /agents/{agent_id}/runs/{run_id}`

Get a specific run record.

**Response (200):** `RunRecord` object.

**Errors:**
- `404 Not Found` — Agent or run does not exist

---

## Data Types

### MCPServerConfig

```json
{
  "name": "github",
  "url": "https://api.githubcopilot.com/mcp/",
  "authorization_token": "ghp_...",
  "allowed_tools": ["list_pulls", "get_pull"]
}
```

| Field | Type | Description |
|-------|------|-------------|
| `name` | string | Identifier for the MCP server |
| `url` | string | Server endpoint URL |
| `authorization_token` | string\|null | Optional auth token |
| `allowed_tools` | string[]\|null | Tool filter; `null` = all tools |

### ScheduleConfig

```json
{
  "cron": "0 9 * * MON",
  "enabled": true
}
```

| Field | Type | Description |
|-------|------|-------------|
| `cron` | string | Cron expression (5-field: minute hour day month weekday) |
| `enabled` | bool | Whether the schedule is active (default: `true`) |

### ToolCall

```json
{
  "tool_name": "mcp:list_pulls",
  "tool_input": {"repo": "my-org/my-repo", "state": "closed"},
  "tool_result": "(executed by MCP server)"
}
```

| Field | Type | Description |
|-------|------|-------------|
| `tool_name` | string | Tool identifier (prefixed with `mcp:`) |
| `tool_input` | object | Arguments passed to the tool |
| `tool_result` | string | Execution result |

---

## Cron Expression Reference

The `cron` field uses standard 5-field cron syntax:

```
┌───────────── minute (0–59)
│ ┌───────────── hour (0–23)
│ │ ┌───────────── day of month (1–31)
│ │ │ ┌───────────── month (1–12)
│ │ │ │ ┌───────────── day of week (0–6, 0=Sunday)
│ │ │ │ │
* * * * *
```

| Pattern | Description |
|---------|-------------|
| `* * * * *` | Every minute |
| `*/15 * * * *` | Every 15 minutes |
| `0 * * * *` | Every hour on the hour |
| `0 9 * * *` | Daily at 9:00 AM |
| `0 9 * * MON` | Every Monday at 9:00 AM |
| `0 9 * * MON-FRI` | Weekdays at 9:00 AM |
| `0 9,17 * * *` | Daily at 9:00 AM and 5:00 PM |
| `0 0 1 * *` | First day of every month at midnight |

---

## Configuration (Environment Variables)

These control the agent executor server behavior:

| Variable | Default | Description |
|----------|---------|-------------|
| `ANTHROPIC_API_KEY` | (required) | Anthropic API key for Claude |
| `ANTHROPIC_MODEL` | `claude-opus-4-20250514` | Claude model to use |
| `MAX_AGENT_TURNS` | `10` | Max iterations per agent run |
| `MAX_TOKENS` | `4096` | Max tokens per Claude response |
| `HOST` | `127.0.0.1` | Server listen address |
| `PORT` | `8080` | Server listen port |
| `NITRIDING_ENABLED` | `true` | Enable Nitro Enclave integration |
| `MAX_RUNS_PER_AGENT` | `100` | Max run records kept per agent |

For local development, set `NITRIDING_ENABLED=false`.

---

## Error Responses

All errors follow this format:

```json
{
  "detail": "Error description"
}
```

| Status | Meaning |
|--------|---------|
| `400` | Bad request (missing required fields) |
| `404` | Agent or run not found |
| `409` | Conflict (duplicate agent name) |
| `422` | Validation error (invalid field types) |
| `500` | Internal server error |

---

## Deploy Script (`scripts/deploy_agent.py`)

A helper script for deploying agents from JSON files with environment variable substitution.

**Usage:**

```bash
# One-shot stateless run
python3 scripts/deploy_agent.py agent.json

# Register as persistent agent
python3 scripts/deploy_agent.py agent.json --register

# Register and immediately trigger a run
python3 scripts/deploy_agent.py agent.json --register --trigger

# Override the prompt
python3 scripts/deploy_agent.py agent.json --prompt "Custom prompt"
```

**Environment variables:**
- `AGENT_EXECUTOR_URL` — Base URL (default: `http://localhost:8000`)
- Any `${VAR}` referenced in the agent JSON must be set

**Agent JSON format:**

```json
{
  "name": "agent-name",
  "system_prompt": "System instructions...",
  "prompt": "Default prompt...",
  "mcp_servers": [
    {
      "name": "server-name",
      "url": "https://...",
      "authorization_token": "${AUTH_TOKEN}",
      "allowed_tools": ["tool1", "tool2"]
    }
  ],
  "schedule": {
    "cron": "0 9 * * MON",
    "enabled": true
  }
}
```
