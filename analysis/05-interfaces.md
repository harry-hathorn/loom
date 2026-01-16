# Loom Repository - Phase 5 API & Interface Documentation

**Iteration:** 7
**Date:** 2025-01-16
**Status:** IN PROGRESS

---

## REST API Endpoints

### Base URL
```
Production: https://loom.ghuntley.com
Development: http://localhost:8080
```

### Authentication
Most endpoints require Bearer token authentication:
```
Authorization: Bearer <session_token>
```

WebSocket endpoints use a short-lived ws_token:
```
ws://localhost:8080/ws?token=<ws_token>
```

---

### Thread Endpoints

#### List Threads
```http
GET /api/threads?workspace={path}&limit=50&offset=0
```

**Query Parameters:**
- `workspace` (optional): Filter by workspace root path
- `limit` (optional): Max results (default: 50, max: 1000)
- `offset` (optional): Pagination offset (default: 0)

**Response:**
```json
{
  "threads": [
    {
      "id": "T-0123456789abcdef0",
      "version": 5,
      "created_at": "2025-01-16T10:00:00Z",
      "updated_at": "2025-01-16T12:30:00Z",
      "last_activity_at": "2025-01-16T12:30:00Z",
      "title": "Fix authentication bug",
      "workspace_root": "/home/user/project",
      "git_branch": "main",
      "git_remote_url": "https://github.com/user/repo",
      "git_current_commit_sha": "abc123",
      "provider": "anthropic",
      "model": "claude-3-5-sonnet-20241022",
      "tags": ["bugfix", "auth"],
      "message_count": 15,
      "is_pinned": false,
      "visibility": "organization"
    }
  ],
  "total": 42,
  "limit": 50,
  "offset": 0
}
```

**Error Responses:**
- `401 Unauthorized`: Invalid or missing session token
- `500 Internal Server Error`: Database error

---

#### Get Thread
```http
GET /api/threads/{thread_id}
```

**Path Parameters:**
- `thread_id`: Thread identifier (e.g., "T-0123456789abcdef0")

**Response:** Full `Thread` object with conversation, agent state, and metadata

**Error Responses:**
- `401 Unauthorized`: Invalid session
- `403 Forbidden`: Thread is private, requester not owner
- `404 Not Found`: Thread doesn't exist or was deleted

---

#### Create/Update Thread
```http
PUT /api/threads/{thread_id}
```

**Request Body:** Full `Thread` object

**Response:** Updated `Thread` object

**Business Rules:**
- Optimistic concurrency: Include `version` in request
- Returns `409 Conflict` if version mismatch
- Auto-increments version on update

---

#### Delete Thread
```http
DELETE /api/threads/{thread_id}
```

**Response:** `204 No Content`

**Business Rules:**
- Soft delete (sets `deleted_at` timestamp)
- Only thread owner can delete
- Irreversible

---

#### Search Threads
```http
GET /api/threads/search?q={query}&workspace={path}&limit=20&offset=0
```

**Query Parameters:**
- `q` (required): Search query (text, branch name, commit SHA)
- `workspace` (optional): Filter by workspace
- `limit` (optional): Max results (default: 20)
- `offset` (optional): Pagination offset

**Response:**
```json
{
  "hits": [
    {
      "id": "T-0123456789abcdef0",
      "title": "Thread title",
      "score": 0.95,
      ... // other ThreadSummary fields
    }
  ],
  "limit": 20,
  "offset": 0
}
```

**Search Behavior:**
- Full-text search using SQLite FTS5
- Searches title, conversation content
- BM25 relevance scoring
- Results ordered by relevance

---

#### Update Thread Visibility
```http
PATCH /api/threads/{thread_id}/visibility
```

**Request Body:**
```json
{
  "visibility": "organization"  // "private" | "organization" | "public"
}
```

**Response:** `204 No Content`

---

### Authentication Endpoints

#### Get Auth Providers
```http
GET /api/auth/providers
```

**Response:**
```json
{
  "providers": ["github", "google", "okta", "magiclink"]
}
```

---

#### Get Current User
```http
GET /api/auth/me
```

**Response:**
```json
{
  "id": "user_abc123",
  "display_name": "Jane Doe",
  "username": "jane",
  "email": "jane@example.com",
  "avatar_url": "https://avatar.url/image.png",
  "locale": "en",
  "global_roles": ["system_admin"],
  "created_at": "2025-01-01T00:00:00Z"
}
```

---

#### Magic Link Authentication
```http
POST /api/auth/magiclink
```

**Request Body:**
```json
{
  "email": "user@example.com"
}
```

**Response:** `202 Accepted` (email sent)

**Flow:**
1. User requests magic link
2. Server sends email with one-time token
3. User clicks link (valid for 15 minutes)
4. Server creates session and redirects

---

#### Device Code Flow (Headless Devices)

**Start Device Code Flow:**
```http
POST /api/auth/device/start
```

**Response:**
```json
{
  "device_code": "device_abc123",
  "user_code": "ABCD-1234",
  "verification_url": "https://loom.ghuntley.com/device",
  "expires_in": 900
}
```

**Poll for Completion:**
```http
POST /api/auth/device/poll
```

**Request Body:**
```json
{
  "device_code": "device_abc123"
}
```

**Response:**
```json
{
  "status": "pending"  // or "completed" with access_token, or "expired"
}
```

---

#### OAuth Callback (Server-Side)
```http
GET /api/auth/{provider}/callback?code={code}&state={state}
```

**Query Parameters:**
- `provider`: "github" | "google" | "okta"
- `code`: OAuth authorization code
- `state`: CSRF protection token
- `error`: Error (if auth failed)
- `error_description`: Error details

**Flow:**
1. User visits `/auth/github/start`
2. Redirected to GitHub
3. User approves authorization
4. GitHub redirects to callback URL
5. Server exchanges code for access token
6. Server creates session
7. Server redirects to success page

---

#### WebSocket Token
```http
POST /api/auth/ws-token
```

**Response:**
```json
{
  "token": "ws_abc123...",
  "expires_in": 30
}
```

**Usage:**
Token valid for 30 seconds, single-use. Used for WebSocket authentication.

---

### LLM Proxy Endpoints

#### Stream LLM Request
```http
POST /llm/stream
```

**Request Body:**
```json
{
  "model": "claude-3-5-sonnet-20241022",
  "messages": [
    {"role": "user", "content": "Hello"}
  ],
  "tools": [
    {
      "name": "read_file",
      "description": "Read a file",
      "input_schema": {
        "type": "object",
        "properties": {
          "path": {"type": "string"}
        },
        "required": ["path"]
      }
    }
  ],
  "max_tokens": 4096,
  "temperature": 0.7
}
```

**Response:** Server-Sent Events (SSE) stream

```
event: text_delta
data: {"content": "Hello"}

event: tool_call_delta
data: {"call_id": "call_123", "tool_name": "read_file", "arguments_fragment": "{\"path": "}

event: tool_call_delta
data: {"call_id": "call_123", "arguments_fragment": "\"/tmp/file.txt\"}"}

event: completed
data: {"message": {...}, "tool_calls": [...], "usage": {...}}
```

**Error Events:**
```
event: error
data: {"error": "rate_limit_exceeded", "message": "Too many requests"}
```

---

### Weaver (Remote Execution) Endpoints

#### Create Weaver
```http
POST /api/weavers
```

**Request Body:**
```json
{
  "image": "ghcr.io/ghuntley/loom-weaver:latest",
  "org_id": "org_abc123",
  "repo_id": "repo_xyz789",
  "env": {
    "GIT_TOKEN": "secret_value"
  },
  "resources": {
    "memory_limit": "8Gi",
    "cpu_limit": "4"
  },
  "tags": {
    "purpose": "debugging"
  },
  "lifetime_hours": 4,
  "command": ["/bin/bash"],
  "args": ["-c", "echo hello"],
  "workdir": "/workspace"
}
```

**Response:**
```json
{
  "id": "weaver_abc123",
  "pod_name": "weaver-weaver-abc123",
  "status": "pending",
  "created_at": "2025-01-16T10:00:00Z",
  "image": "ghcr.io/ghuntley/loom-weaver:latest",
  "lifetime_hours": 4,
  "age_hours": 0.0,
  "owner_user_id": "user_abc123"
}
```

---

#### List Weavers
```http
GET /api/weavers?org_id={org_id}&status={status}
```

**Query Parameters:**
- `org_id` (optional): Filter by organization
- `status` (optional): Filter by status

**Response:**
```json
{
  "weavers": [...],
  "count": 5
}
```

---

#### Get Weaver
```http
GET /api/weavers/{weaver_id}
```

**Response:** `WeaverApiResponse` object

---

#### Delete Weaver
```http
DELETE /api/weavers/{weaver_id}
```

**Response:** `204 No Content`

**Behavior:**
- Terminates Kubernetes pod
- Removes from database
- Irreversible

---

#### Stream Weaver Logs
```http
GET /api/weavers/{weaver_id}/logs?follow=true&tail=100
```

**Query Parameters:**
- `follow`: Keep connection open for new logs
- `tail`: Number of recent lines to include

**Response:** Chunked transfer encoding with log lines

---

#### Attach to Weaver (WebSocket)
```ws
ws://localhost:8080/api/weavers/{weaver_id}/attach?token={ws_token}
```

**WebSocket Messages:**

**Client → Server:**
```json
{
  "type": "resize",
  "rows": 24,
  "cols": 80
}
```

```json
{
  "type": "input",
  "data": "ls -la\n"
}
```

**Server → Client:**
```json
{
  "type": "output",
  "data": "total 24\n"}
```

```json
{
  "type": "exit",
  "exit_code": 0
}
```

---

### Feature Flags Endpoints

#### Create Environment
```http
POST /api/flags/environments
```

**Request Body:**
```json
{
  "name": "production",
  "color": "#10b981"
}
```

**Validation:**
- `name`: 2-50 chars, lowercase alphanumeric with underscores
- `color`: Optional hex color code

---

#### List Environments
```http
GET /api/flags/environments
```

**Response:**
```json
{
  "environments": [
    {
      "id": "env_abc123",
      "org_id": "org_xyz789",
      "name": "production",
      "color": "#10b981",
      "created_at": "2025-01-01T00:00:00Z"
    }
  ]
}
```

---

#### Create SDK Key
```http
POST /api/flags/sdk-keys
```

**Request Body:**
```json
{
  "environment_id": "env_abc123",
  "key_type": "client_side",  // or "server_side"
  "name": "Web SDK"
}
```

**Response:**
```json
{
  "id": "sdk_key_abc123",
  "key": "client-sdk_abc123...",  // Only shown once
  "key_preview": "....xyz8",  // Last 4 chars
  "environment_id": "env_abc123",
  "environment_name": "production",
  "key_type": "client_side",
  "name": "Web SDK",
  "created_at": "2025-01-16T10:00:00Z"
}
```

**Security:**
- `key` field only shown on creation
- Store it securely (cannot be retrieved later)

---

#### Evaluate Flag (Server-Side)
```http
POST /api/flags/evaluate
```

**Request Body:**
```json
{
  "flag_key": "new_ui_enabled",
  "environment": "production",
  "user_id": "user_abc123",
  "org_id": "org_xyz789",
  "session_id": "session_def456",
  "attributes": {
    "plan": "pro",
    "signup_date": "2025-01-01"
  },
  "geo": {
    "country": "US",
    "region": "CA",
    "city": "San Francisco"
  }
}
```

**Response:**
```json
{
  "enabled": true,
  "variant": "treatment_a",
  "reason": "rollout"
}
```

**Reason Values:**
- `kill_switch`: Flag disabled by kill switch
- `whitelist`: User in whitelist
- `rollout`: Deterministic hash-based rollout
- `rule`: Matched targeting rule
- `default`: Flag not found (disabled)

---

### Analytics Endpoints

#### Capture Event
```http
POST /api/analytics/capture
```

**Request Body:**
```json
{
  "events": [
    {
      "event_name": "button_clicked",
      "user_id": "user_abc123",
      "person_id": "person_def456",
      "event_properties": {
        "button_name": "checkout",
        "page": "/pricing"
      },
      "timestamp": "2025-01-16T10:00:00Z"
    }
  ]
}
```

**Response:** `202 Accepted` (events queued for async processing)

**Validation:**
- `event_name`: Max 255 chars, alphanumeric + underscore + hyphen
- `event_properties`: Max 1MB per event
- `timestamp`: Cannot be in future

---

#### Identify (Link Anonymous to Authenticated)
```http
POST /api/analytics/identify
```

**Request Body:**
```json
{
  "distinct_id": "anon_abc123",
  "user_id": "user@example.com",
  "properties": {
    "plan": "pro",
    "signup_date": "2025-01-16"
  }
}
```

**Behavior:**
- Links anonymous session to authenticated user
- Merges all events from anonymous to authenticated person
- Updates person properties

---

#### List Events
```http
POST /api/analytics/events/list
```

**Request Body:**
```json
{
  "person_id": "person_def456",
  "event_name": "button_clicked",
  "limit": 100,
  "offset": 0
}
```

**Response:**
```json
{
  "events": [
    {
      "id": "evt_abc123",
      "event_name": "button_clicked",
      "person_id": "person_def456",
      "properties": {...},
      "timestamp": "2025-01-16T10:00:00Z"
    }
  ],
  "total": 500
}
```

---

#### Count Events
```http
POST /api/analytics/events/count
```

**Request Body:**
```json
{
  "person_id": "person_def456",
  "event_name": "button_clicked",
  "start_date": "2025-01-01T00:00:00Z",
  "end_date": "2025-01-16T23:59:59Z"
}
```

**Response:**
```json
{
  "count": 42
}
```

---

### User & Organization Endpoints

#### Get User Profile
```http
GET /api/users/me
```

**Response:**
```json
{
  "id": "user_abc123",
  "display_name": "Jane Doe",
  "username": "jane",
  "email": "jane@example.com",
  "avatar_url": "https://avatar.url/image.png",
  "locale": "en",
  "created_at": "2025-01-01T00:00:00Z"
}
```

---

#### Update User Profile
```http
PATCH /api/users/me
```

**Request Body:**
```json
{
  "display_name": "Jane Smith",
  "username": "janesmith",
  "locale": "es"
}
```

---

#### List Organizations
```http
GET /api/orgs
```

**Response:**
```json
{
  "orgs": [
    {
      "id": "org_abc123",
      "name": "Acme Corp",
      "slug": "acme-corp",
      "visibility": "public",
      "role": "owner",
      "created_at": "2025-01-01T00:00:00Z"
    }
  ]
}
```

---

#### Create Organization
```http
POST /api/orgs
```

**Request Body:**
```json
{
  "name": "New Company",
  "slug": "new-company",
  "visibility": "private"
}
```

**Validation:**
- `slug`: 2-50 chars, lowercase alphanumeric with hyphens, unique globally

---

### Secrets Endpoints

#### Create Secret
```http
POST /api/secrets
```

**Request Body:**
```json
{
  "name": "github_token",
  "value": "ghp_abc123...",
  "scope": "repo",  // or "org"
  "repo_id": "repo_xyz789"  // Required if scope="repo"
}
```

**Behavior:**
- Encrypts value using envelope encryption
- Never logs or exposes the secret value
- Returns metadata only

**Response:**
```json
{
  "id": "secret_abc123",
  "name": "github_token",
  "scope": "repo",
  "repo_id": "repo_xyz789",
  "created_at": "2025-01-16T10:00:00Z",
  "updated_at": "2025-01-16T10:00:00Z"
}
```

---

#### List Secrets
```http
GET /api/secrets?scope={scope}&repo_id={repo_id}
```

**Response:**
```json
{
  "secrets": [
    {
      "id": "secret_abc123",
      "name": "github_token",
      "scope": "repo",
      "repo_id": "repo_xyz789",
      "created_at": "2025-01-16T10:00:00Z",
      "updated_at": "2025-01-16T10:00:00Z"
    }
  ]
}
```

**Note:** Secret values NOT included (only metadata)

---

### Health Check

#### Health Endpoint
```http
GET /health
```

**Response:**
```json
{
  "status": "healthy",
  "version": "1.2.3",
  "commit": "abc123"
}
```

**Used by:** Kubernetes liveness probes, load balancers

---

## CLI Commands

### Basic Commands

#### Login
```bash
loom login
```

**Behavior:**
- Opens browser for OAuth authentication
- Stores session token locally
- Supports: GitHub, Google, Okta, Magic Link

**Options:**
```bash
loom login --server-url https://custom.loom.com
```

---

#### Logout
```bash
loom logout
```

**Behavior:**
- Deletes local session token
- Revokes session on server

---

#### List Threads
```bash
loom list
```

**Output:** Table of local threads with ID, title, updated time

**Options:**
```bash
loom list --workspace /path/to/project
```

---

#### Resume Thread
```bash
loom resume [thread_id]
```

**Behavior:**
- Resumes most recent thread if no ID specified
- Loads conversation history
- Restores agent state

---

#### Start New Session
```bash
loom private
```

**Behavior:**
- Creates new thread with `is_private=true`
- Thread never syncs to server

---

#### Share Thread
```bash
loom share [thread_id] --visibility organization
loom share [thread_id] --visibility private
loom share [thread_id] --visibility public
loom share [thread_id] --support  # Share with support team
```

---

#### Search Threads
```bash
loom search "authentication bug" --limit 20
```

**Searches:**
- Thread titles
- Conversation content
- Git branch names
- Repository URLs
- Commit SHAs (prefix match)

---

#### Version
```bash
loom version
```

**Output:** Version, commit SHA, build date

---

#### Update
```bash
loom update
```

**Behavior:**
- Fetches latest version from server
- Self-updates CLI binary

---

### Weaver Commands

#### Create Weaver
```bash
loom new --image ghcr.io/ghuntley/loom-weaver:latest \
          --org org_abc123 \
          --repo https://github.com/user/repo \
          --branch main \
          -e GIT_TOKEN=secret \
          -e ANOTHER_VAR=value \
          --ttl 4
```

**Options:**
- `--image`: Container image (default: latest Loom weaver image)
- `--org`: Organization ID
- `--repo`: Git repository to clone
- `--branch`: Branch to checkout
- `-e`: Environment variables (repeatable)
- `--ttl`: Lifetime in hours (default: 4, max: 48)

---

#### Attach to Weaver
```bash
loom attach weaver_abc123
```

**Behavior:**
- Interactive shell to weaver
- Supports terminal resizing
- Forwards exit code

---

#### List Weavers
```bash
loom weaver ps
```

**Output:** Table of weavers with ID, status, age

---

#### Delete Weaver
```bash
loom weaver delete weaver_abc123
```

---

### ACP Mode (Editor Integration)

```bash
loom acp-agent
```

**Behavior:**
- Runs Loom as Agent Communication Protocol agent
- Communicates via stdio (JSON messages)
- Used by editor integrations (VSCode, etc.)

---

### Spool Commands

```bash
loom spool init      # Initialize spool repository
loom spool status    # Show spool status
loom spool log       # Show commit log
loom spool sync      # Sync with remote
```

---

### WireGuard Tunnel Commands

```bash
loom tunnel start    # Start WireGuard tunnel
loom tunnel status   # Show tunnel status
loom tunnel stop     # Stop tunnel
```

---

#### SSH to Weaver
```bash
loom ssh weaver_abc123
```

**Behavior:**
- Connects via WireGuard tunnel
- Provides secure SSH access

---

#### WireGuard Device Management
```bash
loom wg devices list     # List devices
loom wg devices add       # Add new device
loom wg devices remove    # Remove device
```

---

## WebSocket Interfaces

### Agent WebSocket

**Endpoint:** `ws://localhost:8080/ws?token={ws_token}`

**Purpose:** Real-time bidirectional communication for agent conversation

**Client → Server Messages:**

**Send User Message:**
```json
{
  "type": "user_message",
  "content": "Help me fix this bug"
}
```

**Tool Result:**
```json
{
  "type": "tool_result",
  "call_id": "call_abc123",
  "output": {...},
  "error": null
}
```

**Post-Tools Hook Complete:**
```json
{
  "type": "post_tools_hook_complete",
  "action_taken": true
}
```

**Shutdown:**
```json
{
  "type": "shutdown"
}
```

**Server → Client Messages:**

**Text Delta (Streaming):**
```json
{
  "type": "text_delta",
  "content": "I'll help you fix"
}
```

**Tool Call:**
```json
{
  "type": "tool_call",
  "call_id": "call_abc123",
  "tool_name": "read_file",
  "arguments_json": {"path": "/tmp/file.txt"}
}
```

**Run Post-Tools Hook:**
```json
{
  "type": "run_post_tools_hook",
  "completed_tools": [
    {"tool_name": "edit_file", "succeeded": true}
  ]
}
```

**Error:**
```json
{
  "type": "error",
  "error": "Tool execution failed: ..."
}
```

**Connection Close:**
```json
{
  "type": "close",
  "reason": "Conversation complete"
}
```

---

## Configuration Options

### Server Configuration

**File:** `/etc/loom/server.toml` or `LOOM_SERVER_*` environment variables

**Environment Variable Convention:**
```bash
LOOM_SERVER_{SECTION}_{FIELD}
```

**Example:**
```bash
LOOM_SERVER_HTTP_PORT=8080
LOOM_SERVER_DATABASE_PATH=/var/lib/loom/db.sqlite
LOOM_SERVER_AUTH_SIGNUPS_DISABLED=false
```

**Configuration Sections:**

#### HTTP Server
```toml
[http]
host = "0.0.0.0"
port = 8080
cors_origins = ["https://loom.ghuntley.com"]
```

#### Database
```toml
[database]
path = "/var/lib/loom/db.sqlite"
```

#### Authentication
```toml
[auth]
signups_disabled = false
dev_mode = false
default_locale = "en"

# OAuth providers (optional)
[auth.github]
client_id = "github_client_id"
client_secret = "github_client_secret"

[auth.google]
client_id = "google_client_id"
client_secret = "google_client_secret"

[auth.okta]
client_id = "okta_client_id"
client_secret = "okta_client_secret"
issuer = "https://dev-abc123.okta.com"
```

#### LLM Providers
```toml
[llm]
default_provider = "anthropic"

[llm.anthropic]
api_key = "sk-ant-..."
base_url = "https://api.anthropic.com"

[llm.openai]
api_key = "sk-..."
base_url = "https://api.openai.com/v1"

[llm.vertex]
project_id = "my-project"
region = "us-central1"
credentials_path = "/path/to/service-account.json"
```

#### Email (SMTP)
```toml
[smtp]
host = "smtp.gmail.com"
port = 587
username = "noreply@example.com"
from_address = "Loom <noreply@example.com>"
```

#### Jobs (Scheduler)
```toml
[jobs]
enabled = true
interval_seconds = 60
```

#### GeoIP
```toml
[geoip]
maxmind_db_path = "/var/lib/loom/GeoLite2-City.mmdb"
```

#### Logging
```toml
[logging]
level = "info"  # trace, debug, info, warn, error
format = "compact"  # compact, json
```

#### Weaver (Kubernetes)
```toml
[weaver]
enabled = true
namespace = "loom-weavers"
cleanup_interval_minutes = 5
retention_hours = 1
```

---

### CLI Configuration

**File Locations:**
- User config: `~/.config/loom/config.toml`
- Project config: `.loom/config.toml`
- Command-line flags override everything

**Example Configuration:**
```toml
[server]
url = "https://loom.ghuntley.com"

[llm]
provider = "anthropic"
model = "claude-3-5-sonnet-20241022"
max_tokens = 8192
temperature = 0.7

[workspace]
root = "/home/user/projects"

[auto_commit]
enabled = true
```

**Command-Line Overrides:**
```bash
loom --server-url https://custom.com \
    --provider openai \
    --workspace /path/to/project \
    --log-level debug
```

---

## Error Response Format

All endpoints return errors in consistent format:

```json
{
  "error": "error_type",
  "message": "Human-readable error message",
  "details": {...}  // Optional additional context
}
```

**Common Error Types:**
- `authentication_failed`: Invalid credentials
- `authorization_failed`: Insufficient permissions
- `validation_error`: Invalid request parameters
- `not_found`: Resource doesn't exist
- `conflict`: Version conflict (optimistic concurrency)
- `rate_limit_exceeded`: Too many requests
- `internal_error`: Server error

**HTTP Status Codes:**
- `200 OK`: Successful GET
- `201 Created`: Successful POST
- `202 Accepted`: Request accepted for async processing
- `204 No Content`: Successful DELETE/PATCH
- `400 Bad Request`: Validation error
- `401 Unauthorized`: Authentication required/failed
- `403 Forbidden`: Authorization failed
- `404 Not Found`: Resource doesn't exist
- `409 Conflict`: Version conflict
- `429 Too Many Requests`: Rate limit exceeded
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Server temporarily unavailable

---

**Phase 5 Status:** 60% Complete
**Documented:**
- ✓ REST API endpoints (Threads, Auth, LLM Proxy, Weavers, Feature Flags, Analytics, Users, Secrets, Health)
- ✓ CLI commands (Basic, Weaver, ACP, Spool, WireGuard)
- ✓ WebSocket interfaces (Agent, Weaver attach)
- ✓ Configuration options (Server, CLI)

**Remaining:**
- SDK client methods
- Webhook payloads
- More detailed examples

**Next:** Continue documenting remaining interfaces
