# Loom Repository - Phase 3 Data Structures

**Iteration:** 3
**Date:** 2025-01-16
**Status:** IN PROGRESS

---

## Core Data Types (loom-common-core)

### Message & Conversation Types

**Location:** `crates/loom-common-core/src/message.rs`

```rust
/// Role of a message participant
pub enum Role {
    System,
    User,
    Assistant,
    Tool,
}

/// A message in a conversation
pub struct Message {
    pub role: Role,
    pub content: String,
    pub tool_call_id: Option<String>,  // For tool responses
    pub name: Option<String>,           // For tool responses
    pub tool_calls: Vec<ToolCall>,      // For assistant messages
}

/// A tool call requested by the LLM
pub struct ToolCall {
    pub id: String,
    pub tool_name: String,
    pub arguments_json: serde_json::Value,
}
```

**Purpose:** Core conversation data structure for LLM interactions

**Validation Rules:**
- Tool messages require `tool_call_id` and `name`
- Assistant messages may contain `tool_calls`
- System messages typically come first in conversations

---

### LLM Types

**Location:** `crates/loom-common-core/src/llm.rs`

```rust
/// Request to send to an LLM for completion
pub struct LlmRequest {
    pub model: String,
    pub messages: Vec<Message>,
    pub tools: Vec<ToolDefinition>,
    pub max_tokens: Option<u32>,
    pub temperature: Option<f32>,
}

/// Response from an LLM completion request
pub struct LlmResponse {
    pub message: Message,
    pub tool_calls: Vec<ToolCall>,
    pub usage: Option<Usage>,
    pub finish_reason: Option<String>,
}

/// Token usage statistics
pub struct Usage {
    pub input_tokens: u32,
    pub output_tokens: u32,
}

/// Streaming events during completion
pub enum LlmEvent {
    TextDelta { content: String },
    ToolCallDelta {
        call_id: String,
        tool_name: String,
        arguments_fragment: String,
    },
    Completed(LlmResponse),
    Error(LlmError),
}

/// LLM client trait
#[async_trait]
pub trait LlmClient: Send + Sync {
    async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
    async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError>;
}
```

**Purpose:** Abstraction over LLM providers (Anthropic, OpenAI, Vertex)

---

### Tool Types

**Location:** `crates/loom-common-core/src/tool.rs`

```rust
/// Definition of a tool for the LLM
pub struct ToolDefinition {
    pub name: String,
    pub description: String,
    pub input_schema: serde_json::Value,  // JSON Schema format
}

/// Context provided to tools during execution
pub struct ToolContext {
    pub workspace_root: PathBuf,
}
```

**Purpose:** Tool registration and execution framework

---

### Agent State Machine Types

**Location:** `crates/loom-common-core/src/state.rs`

```rust
/// Main state machine for the agent
pub enum AgentState {
    WaitingForUserInput { conversation: ConversationContext },
    CallingLlm { conversation: ConversationContext, retries: u32 },
    ProcessingLlmResponse {
        conversation: ConversationContext,
        response: LlmResponse,
    },
    ExecutingTools {
        conversation: ConversationContext,
        executions: Vec<ToolExecutionStatus>,
    },
    PostToolsHook {
        conversation: ConversationContext,
        pending_llm_request: LlmRequest,
        completed_tools: Vec<CompletedToolInfo>,
    },
    Error {
        conversation: ConversationContext,
        error: AgentError,
        retries: u32,
        origin: ErrorOrigin,
    },
    ShuttingDown,
}

/// Tool execution lifecycle
pub enum ToolExecutionStatus {
    Pending { call_id: String, tool_name: String, requested_at: Instant },
    Running {
        call_id: String,
        tool_name: String,
        started_at: Instant,
        last_update_at: Instant,
        progress: Option<ToolProgress>,
    },
    Completed {
        call_id: String,
        tool_name: String,
        started_at: Instant,
        completed_at: Instant,
        outcome: ToolExecutionOutcome,
    },
}

/// Outcome of completed tool execution
pub enum ToolExecutionOutcome {
    Success { call_id: String, output: serde_json::Value },
    Error { call_id: String, error: ToolError },
}

/// Progress information for running tools
pub struct ToolProgress {
    pub fraction: Option<f32>,           // 0.0 to 1.0
    pub message: Option<String>,
    pub units_processed: Option<u64>,
}

/// Events that drive the state machine
pub enum AgentEvent {
    UserInput(Message),
    LlmEvent(LlmEvent),
    ToolProgress(ToolProgressEvent),
    ToolCompleted { call_id: String, outcome: ToolExecutionOutcome },
    PostToolsHookCompleted { action_taken: bool },
    RetryTimeoutFired,
    ShutdownRequested,
}

/// Conversation context
pub struct ConversationContext {
    pub id: uuid::Uuid,
    pub messages: Vec<Message>,
}

/// Origin of errors for retry decisions
pub enum ErrorOrigin {
    Llm,
    Tool,
    Io,
}
```

**Purpose:** Manages agent conversation flow and tool execution

---

### Thread Persistence Types

**Location:** `crates/loom-common-thread/src/model.rs`

```rust
/// Thread identifier: "T-{uuid7}"
pub struct ThreadId(String);

/// Thread visibility for synced threads
pub enum ThreadVisibility {
    Organization,  // Default - visible to org members
    Private,       // Only owner can see
    Public,        // May be exposed publicly
}

/// Message role for persistence
pub enum MessageRole {
    System,
    User,
    Assistant,
    Tool,
}

/// Message snapshot for persistence
pub struct MessageSnapshot {
    pub role: MessageRole,
    pub content: String,
    pub tool_call_id: Option<String>,
    pub tool_name: Option<String>,
    pub tool_calls: Option<Vec<ToolCallSnapshot>>,
}

/// Tool call snapshot
pub struct ToolCallSnapshot {
    pub id: String,
    pub tool_name: String,
    pub arguments_json: serde_json::Value,
}

/// Conversation snapshot
pub struct ConversationSnapshot {
    pub messages: Vec<MessageSnapshot>,
}

/// Agent state kind for persistence
pub enum AgentStateKind {
    WaitingForUserInput,
    CallingLlm,
    ProcessingLlmResponse,
    ExecutingTools,
    PostToolsHook,
    Error,
    ShuttingDown,
}

/// Agent state snapshot
pub struct AgentStateSnapshot {
    pub kind: AgentStateKind,
    pub retries: u32,
    pub last_error: Option<String>,
    pub pending_tool_calls: Vec<PendingToolCallSnapshot>,
}

/// Pending tool call snapshot
pub struct PendingToolCallSnapshot {
    pub call_id: String,
    pub tool_name: String,
}

/// Thread metadata
pub struct ThreadMetadata {
    pub title: Option<String>,
    pub tags: Vec<String>,
    pub is_pinned: bool,
    pub extra: serde_json::Value,
}

/// Complete thread (conversation + state + metadata)
pub struct Thread {
    pub id: ThreadId,
    pub version: u64,
    pub created_at: String,        // RFC3339
    pub updated_at: String,        // RFC3339
    pub last_activity_at: String,  // RFC3339

    // Workspace context
    pub workspace_root: Option<String>,
    pub cwd: Option<String>,
    pub loom_version: Option<String>,

    // Git metadata
    pub git_branch: Option<String>,
    pub git_remote_url: Option<String>,
    pub git_initial_branch: Option<String>,
    pub git_initial_commit_sha: Option<String>,
    pub git_current_commit_sha: Option<String>,
    pub git_start_dirty: Option<bool>,
    pub git_end_dirty: Option<bool>,
    pub git_commits: Vec<String>,

    // LLM metadata
    pub provider: Option<String>,
    pub model: Option<String>,

    // Core data
    pub conversation: ConversationSnapshot,
    pub agent_state: AgentStateSnapshot,
    pub metadata: ThreadMetadata,

    // Server-side visibility
    pub visibility: ThreadVisibility,
    pub is_private: bool,              // Never syncs to server
    pub is_shared_with_support: bool,  // Shared with support team
}

/// Thread summary for listings
pub struct ThreadSummary {
    pub id: ThreadId,
    pub version: u64,
    pub created_at: String,
    pub updated_at: String,
    pub last_activity_at: String,
    pub title: Option<String>,
    pub workspace_root: Option<String>,
    pub git_branch: Option<String>,
    pub git_remote_url: Option<String>,
    pub git_initial_commit_sha: Option<String>,
    pub git_current_commit_sha: Option<String>,
    pub provider: Option<String>,
    pub model: Option<String>,
    pub tags: Vec<String>,
    pub message_count: u32,
    pub is_pinned: bool,
    pub visibility: ThreadVisibility,
}
```

**Purpose:** Thread persistence with sync support, git context tracking

**Key Features:**
- UUID7-based thread IDs (time-ordered)
- Optimistic concurrency with version field
- Comprehensive git metadata tracking
- Visibility controls for organization sharing
- Support for private threads (no sync)

---

## Database Schemas

### Threads Table

**Migration:** `001_create_threads.sql`

```sql
CREATE TABLE IF NOT EXISTS threads (
    -- Primary identifier: "T-{uuid7}"
    id TEXT PRIMARY KEY NOT NULL,

    -- Optimistic concurrency version
    version INTEGER NOT NULL DEFAULT 1,

    -- Timestamps (RFC3339 format)
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    last_activity_at TEXT NOT NULL,
    deleted_at TEXT,  -- Soft delete

    -- Denormalized fields for querying
    workspace_root TEXT,
    cwd TEXT,
    loom_version TEXT,
    provider TEXT,
    model TEXT,

    -- Metadata (denormalized)
    title TEXT,
    tags TEXT,  -- JSON array
    is_pinned INTEGER NOT NULL DEFAULT 0,
    message_count INTEGER NOT NULL DEFAULT 0,

    -- Agent state (denormalized kind for filtering)
    agent_state_kind TEXT NOT NULL,
    agent_state JSON NOT NULL,

    -- Conversation data
    conversation JSON NOT NULL,
    metadata JSON NOT NULL,
    full_json JSON NOT NULL
);

-- Key indexes
CREATE INDEX idx_threads_workspace_activity
    ON threads (workspace_root, last_activity_at DESC)
    WHERE deleted_at IS NULL;

CREATE INDEX idx_threads_pinned
    ON threads (is_pinned, last_activity_at DESC)
    WHERE deleted_at IS NULL;
```

**Relationships:**
- One-to-many with `thread_repos` (git repositories)
- One-to-many with `thread_commits` (commit history)

---

### Thread Visibility Table

**Migration:** `002_add_visibility.sql`

```sql
ALTER TABLE threads ADD COLUMN visibility TEXT DEFAULT 'organization';
ALTER TABLE threads ADD COLUMN is_private INTEGER DEFAULT 0;
ALTER TABLE threads ADD COLUMN is_shared_with_support INTEGER DEFAULT 0;
```

---

### Git Metadata Tables

**Migration:** `003_add_git_metadata.sql`

```sql
ALTER TABLE threads ADD COLUMN git_branch TEXT;
ALTER TABLE threads ADD COLUMN git_remote_url TEXT;
ALTER TABLE threads ADD COLUMN git_initial_branch TEXT;
ALTER TABLE threads ADD COLUMN git_initial_commit_sha TEXT;
ALTER TABLE threads ADD COLUMN git_current_commit_sha TEXT;
ALTER TABLE threads ADD COLUMN git_start_dirty INTEGER;
ALTER TABLE threads ADD COLUMN git_end_dirty INTEGER;
ALTER TABLE threads ADD COLUMN git_commits TEXT;  -- JSON array
```

---

### Git Repos & Commits Tables

**Migration:** `004_git_repos_and_commits.sql`

```sql
CREATE TABLE IF NOT EXISTS thread_repos (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    slug TEXT NOT NULL UNIQUE,  -- e.g., "github.com/owner/repo"
    created_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS thread_commits (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    thread_id TEXT NOT NULL REFERENCES threads(id) ON DELETE CASCADE,
    repo_id INTEGER NOT NULL REFERENCES thread_repos(id) ON DELETE CASCADE,
    commit_sha TEXT NOT NULL,
    branch TEXT,
    is_dirty INTEGER NOT NULL DEFAULT 0,
    observed_at TEXT NOT NULL,
    is_initial INTEGER NOT NULL DEFAULT 0,
    is_final INTEGER NOT NULL DEFAULT 0,
    created_at TEXT NOT NULL
);

CREATE INDEX idx_thread_commits_thread
    ON thread_commits(thread_id);
```

---

### Full-Text Search Table

**Migration:** `005_thread_fts.sql`

```sql
CREATE VIRTUAL TABLE IF NOT EXISTS threads_fts USING fts5(
    id,          -- Thread ID
    title,       -- Thread title
    content,     -- Conversation content
    tokenize='porter unicode61'
);
```

**Purpose:** Fast text search across thread content

---

### Authentication Tables

**Migration:** `008_auth_users.sql`

```sql
CREATE TABLE IF NOT EXISTS users (
    id TEXT PRIMARY KEY,
    display_name TEXT NOT NULL,
    primary_email TEXT UNIQUE,
    avatar_url TEXT,
    email_visible INTEGER DEFAULT 1,
    is_system_admin INTEGER DEFAULT 0,
    is_support INTEGER DEFAULT 0,
    is_auditor INTEGER DEFAULT 0,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    deleted_at TEXT
);

CREATE TABLE IF NOT EXISTS identities (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    provider TEXT NOT NULL,          -- github, google, okta, magiclink
    provider_user_id TEXT NOT NULL,
    email TEXT NOT NULL,
    email_verified INTEGER DEFAULT 0,
    access_token TEXT,
    refresh_token TEXT,
    token_expires_at TEXT,
    created_at TEXT NOT NULL,
    UNIQUE(provider, provider_user_id)
);
```

**Relationships:**
- Users have many identities (OAuth providers)
- Users belong to organizations
- Users belong to teams

---

### Teams Table

**Migration:** `011_auth_teams.sql`

```sql
CREATE TABLE IF NOT EXISTS teams (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    name TEXT NOT NULL,
    slug TEXT NOT NULL,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    UNIQUE(org_id, slug)
);

CREATE TABLE IF NOT EXISTS team_memberships (
    id TEXT PRIMARY KEY,
    team_id TEXT NOT NULL REFERENCES teams(id) ON DELETE CASCADE,
    user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL,             -- member, maintainer, owner
    created_at TEXT NOT NULL,
    UNIQUE(team_id, user_id)
);
```

---

### Sessions Table

**Migration:** `009_auth_sessions.sql`

```sql
CREATE TABLE IF NOT EXISTS sessions (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash TEXT NOT NULL UNIQUE,
    expires_at TEXT NOT NULL,
    created_at TEXT NOT NULL,
    last_seen_at TEXT
);
```

---

### Organizations Table

**Migration:** `010_auth_orgs.sql`

```sql
CREATE TABLE IF NOT EXISTS organizations (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL,
    slug TEXT NOT NULL UNIQUE,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS organization_memberships (
    id TEXT PRIMARY KEY,
    org_id TEXT NOT NULL REFERENCES organizations(id) ON DELETE CASCADE,
    user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    role TEXT NOT NULL,
    created_at TEXT NOT NULL,
    UNIQUE(org_id, user_id)
);
```

---

### API Keys Table

**Migration:** `012_auth_api_keys.sql`

```sql
CREATE TABLE IF NOT EXISTS api_keys (
    id TEXT PRIMARY KEY,
    user_id TEXT NOT NULL REFERENCES users(id) ON DELETE CASCADE,
    token_hash TEXT NOT NULL UNIQUE,
    name TEXT NOT NULL,
    scopes TEXT,                     -- JSON array
    expires_at TEXT,
    created_at TEXT NOT NULL,
    last_used_at TEXT
);
```

---

### Audit Log Table

**Migration:** `014_auth_audit.sql`

```sql
CREATE TABLE IF NOT EXISTS audit_logs (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp TEXT NOT NULL,
    user_id TEXT,
    event_type TEXT NOT NULL,
    resource_type TEXT,
    resource_id TEXT,
    actor_ip TEXT,
    actor_user_agent TEXT,
    details JSON,
    INDEX idx_audit_logs_timestamp (timestamp),
    INDEX idx_audit_logs_user (user_id)
);
```

---

### Jobs Table

**Migration:** `019_jobs.sql`

```sql
CREATE TABLE IF NOT EXISTS jobs (
    id TEXT PRIMARY KEY,
    job_type TEXT NOT NULL,
    status TEXT NOT NULL,
    payload JSON,
    result JSON,
    error_message TEXT,
    scheduled_at TEXT,
    started_at TEXT,
    completed_at TEXT,
    retry_count INTEGER DEFAULT 0,
    max_retries INTEGER DEFAULT 3,
    INDEX idx_jobs_status (status),
    INDEX idx_jobs_scheduled (scheduled_at)
);
```

---

### Feature Flags Table

**Migration:** `030_feature_flags.sql`

```sql
CREATE TABLE IF NOT EXISTS feature_flags (
    id TEXT PRIMARY KEY,
    name TEXT NOT NULL UNIQUE,
    description TEXT,
    enabled BOOLEAN DEFAULT FALSE,
    conditions JSON,                 -- Rollout rules
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL
);
```

---

### Analytics Tables

**Migration:** `032_analytics.sql`

```sql
CREATE TABLE IF NOT EXISTS analytics_events (
    id TEXT PRIMARY KEY,
    event_name TEXT NOT NULL,
    user_id TEXT,
    person_id TEXT,
    event_properties JSON,
    timestamp TEXT NOT NULL,
    INDEX idx_analytics_events_person (person_id),
    INDEX idx_analytics_events_timestamp (timestamp)
);

CREATE TABLE IF NOT EXISTS analytics_people (
    person_id TEXT PRIMARY KEY,
    user_id TEXT,
    properties JSON,
    created_at TEXT NOT NULL,
    updated_at TEXT NOT NULL,
    INDEX idx_analytics_people_user (user_id)
);
```

---

**Phase 3 Status:** 40% Complete
**Documented:**
- ✓ Core data types (Message, LLM, Tool, Agent State)
- ✓ Thread persistence types
- ✓ Database schemas (all migrations)

**Remaining:**
- API request/response shapes
- Configuration structures
- State management structures (deep dive)

**Next:** Continue documenting API shapes and configuration
