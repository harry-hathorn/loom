# Loom Repository - Phase 6 Architecture & Patterns

**Iteration:** 8
**Date:** 2025-01-16
**Status:** IN PROGRESS

---

## Overall Architectural Pattern

### Architecture Style: Layered Modular Monolith

Loom follows a **layered modular monolith** architecture with clear separation of concerns:

```
┌─────────────────────────────────────────────────────────────────┐
│                     Presentation Layer                          │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   CLI    │  │  Server  │  │   TUI    │  │   Web    │   │
│  │loom-cli  │  │loom-server│  │loom-tui-*│  │loom-web  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      API & Routing Layer                        │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         Typed Router (Axum)                              │  │
│  │  - PublicRouter (unauthenticated)                        │  │
│  │  - AuthedRouter (authenticated)                          │  │
│  │  - Middleware (Auth, ABAC, Audit, Tracing)              │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                    Business Logic Layer                         │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Auth   │  │   LLM    │  │  Weaver  │  │ Feature   │   │
│  │          │  │ Service  │  │ Provision│  │  Flags    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │Analytics │  │ Secrets  │  │    SCM   │  │   Jobs    │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Data Access Layer                           │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │     Repositories (ThreadStore, SecretStore, etc.)       │  │
│  │     Database: SQLite with connection pooling             │  │
│  └──────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              │
┌─────────────────────────────────────────────────────────────────┐
│                      Common Libraries Layer                     │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │   Core   │  │  Thread  │  │   HTTP   │  │  Secret  │   │
│  │  Types   │  │Persist   │  │  Client  │  │Handling  │   │
│  └──────────┘  └──────────┘  └──────────┘  └──────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

**Key Architectural Principles:**

1. **Horizontal Layering:** Clear separation between presentation, API, business logic, data access, and common layers
2. **Vertical Slicing:** Feature-specific modules (auth, LLM, weaver, etc.) with internal cohesion
3. **Shared Foundation:** Common libraries provide reusable types and utilities
4. **Dependency Inversion:** Business logic depends on abstractions (traits), not concrete implementations
5. **Module Boundaries:** Each crate has a single, well-defined responsibility

---

## Design Patterns

### 1. State Machine Pattern

**Location:** `loom-common-core/src/state.rs`, `agent.rs`

**Purpose:** Manage agent conversation flow and tool execution

**Implementation:**
- `AgentState` enum with 7 states (WaitingForUserInput, CallingLlm, ProcessingLlmResponse, ExecutingTools, PostToolsHook, Error, ShuttingDown)
- `AgentEvent` enum for state transitions
- `handle_event()` method processes events and returns `AgentAction`
- State transitions are explicit and validated

**Benefits:**
- Explicit state representation
- Type-safe transitions
- Easy to test and verify
- Clear error handling paths

---

### 2. Repository Pattern

**Location:** `loom-server-db/src/*.rs`

**Purpose:** Abstract data access and provide CRUD operations

**Implementation:**
```rust
#[async_trait]
pub trait ThreadStore: Send + Sync {
    async fn upsert(&self, thread: &Thread, expected_version: Option<u64>)
        -> Result<Thread, DbError>;
    async fn get(&self, id: &ThreadId) -> Result<Option<Thread>, DbError>;
    async fn delete(&self, id: &ThreadId) -> Result<bool, DbError>;
    // ...
}
```

**Benefits:**
- Swappable implementations (SQLite, PostgreSQL, etc.)
- Testable with mock repositories
- Centralized query logic
- Optimistic concurrency support

---

### 3. Strategy Pattern

**Location:** `loom-server-llm-service/src/lib.rs`

**Purpose:** Support multiple LLM providers with unified interface

**Implementation:**
```rust
#[async_trait]
pub trait LlmClient: Send + Sync {
    async fn complete(&self, request: LlmRequest) -> Result<LlmResponse, LlmError>;
    async fn complete_streaming(&self, request: LlmRequest) -> Result<LlmStream, LlmError>;
}

// Implementations:
pub struct AnthropicClient { ... }
pub struct OpenAIClient { ... }
pub struct VertexClient { ... }
pub struct ProxyLlmClient { ... }  // Routes to server
```

**Benefits:**
- Easy to add new providers
- Runtime provider selection
- Consistent interface across providers
- Graceful fallback to proxy

---

### 4. Builder Pattern

**Location:** Configuration loading, complex request construction

**Purpose:** Construct complex objects step-by-step

**Implementation:**
```rust
impl EvaluationContext {
    pub fn new(environment: impl Into<String>) -> Self {
        Self {
            environment: environment.into(),
            ..Default::default()
        }
    }

    pub fn with_user_id(mut self, user_id: impl Into<String>) -> Self {
        self.user_id = Some(user_id.into());
        self
    }

    pub fn with_org_id(mut self, org_id: impl Into<String>) -> Self {
        self.org_id = Some(org_id.into());
        self
    }
}
```

**Benefits:**
- Fluent API for object construction
- Optional parameters with sensible defaults
- Readable and self-documenting

---

### 5. Middleware Pattern

**Location:** `loom-server/src/*_middleware.rs`

**Purpose:** Cross-cutting concerns (authentication, authorization, audit)

**Implementation:**
```rust
// Authentication middleware
pub async fn auth_middleware(
    req: Request,
    next: Next,
) -> Result<Response, StatusCode> {
    let token = extract_token(&req)?;
    let session = validate_session(token).await?;
    req.extensions_mut().insert(session);
    Ok(next.run(req).await)
}

// ABAC middleware
pub struct RequireRole(pub &'static [&static str]);

impl<S> Layer<S> for RequireRole {
    fn call(&self, req: Request, next: Next) -> Result<Response, StatusCode> {
        let user = req.extensions().get::<User>();
        if user.has_any_role(self.0) {
            Ok(next.run(req).await)
        } else {
            Err(StatusCode::FORBIDDEN)
        }
    }
}
```

**Benefits:**
- Reusable cross-cutting logic
- Composable middleware stack
- Request/response interception
- Clean separation from handlers

---

### 6. Observer/Publisher-Subscriber Pattern

**Location:** `loom-server-flags/src/flags_broadcaster.rs`

**Purpose:** Real-time flag updates to connected clients

**Implementation:**
- SSE (Server-Sent Events) for push notifications
- Clients subscribe to flag changes
- Server broadcasts on flag updates

**Benefits:**
- Real-time feature flag updates
- Decoupled flag evaluation from notification
- Scalable to many subscribers

---

### 7. Factory Pattern

**Location:** Tool registration, LLM client creation

**Purpose:** Create objects based on configuration

**Implementation:**
```rust
pub struct ToolRegistry {
    tools: HashMap<String, Box<dyn Tool>>,
}

impl ToolRegistry {
    pub fn new() -> Self {
        let mut registry = Self {
            tools: HashMap::new(),
        };
        registry.register(Box::new(EditFileTool::new()));
        registry.register(Box::new(ReadFileTool::new()));
        registry.register(Box::new(BashTool::new()));
        // ...
        registry
    }
}
```

**Benefits:**
- Centralized object creation
- Easy to add new tools
- Type-safe registration

---

### 8. Adapter Pattern

**Location:** `loom-server-llm-proxy/src/lib.rs`

**Purpose:** Adapt different LLM APIs to common interface

**Implementation:**
- Converts between internal `LlmRequest` and provider-specific formats
- Normalizes streaming responses to `LlmEvent` stream
- Handles provider-specific quirks

**Benefits:**
- Uniform interface for different providers
- Provider isolation
- Easy to add new providers

---

## Code Organization Philosophy

### 1. Crate Responsibility

**Single Responsibility Principle:** Each crate has one clear purpose:

| Crate | Single Responsibility |
|-------|---------------------|
| `loom-cli` | CLI entry point and REPL loop |
| `loom-server` | HTTP server and route composition |
| `loom-common-core` | Core type definitions (Message, Tool, LLM) |
| `loom-server-auth` | Authentication abstraction |
| `loom-server-auth-github` | GitHub OAuth implementation |
| `loom-server-db` | Database repositories and migrations |
| `loom-cli-tools` | Tool implementations |
| `loom-tui-app` | TUI application composition |
| `loom-flags-core` | Feature flag evaluation logic |

**No God Objects:** Each crate is focused and composable.

---

### 2. Trait-Based Abstraction

**Dependency Inversion:** High-level modules depend on traits, not implementations:

```rust
// High-level module (agent) depends on abstraction
pub struct Agent {
    llm: Arc<dyn LlmClient>,  // Trait object
    tools: Vec<ToolDefinition>,
}

// Low-level modules implement trait
impl LlmClient for AnthropicClient { ... }
impl LlmClient for ProxyLlmClient { ... }
```

**Benefits:**
- Testable with mocks
- Runtime implementation selection
- Loose coupling

---

### 3. Module Visibility

**Public vs Private API:**
- `pub` for types used across crates
- Private (no `pub`) for internal implementation
- Re-exports for convenience (`pub use`)

**Example:**
```rust
// lib.rs - public API
pub use self::message::{Message, Role, ToolCall};
pub use self::llm::{LlmClient, LlmRequest};

// Internal modules
mod message;
mod llm;
mod state;  // Private to crate
```

---

### 4. Error Handling Strategy

**thiserror for Domain Errors:**
```rust
#[derive(Debug, thiserror::Error)]
pub enum AgentError {
    #[error("LLM error: {0}")]
    Llm(#[from] LlmError),
    #[error("Tool error: {0}")]
    Tool(#[from] ToolError),
    #[error("IO error: {0}")]
    Io(#[from] std::io::Error),
}
```

**anyhow for Propagation:**
```rust
pub async fn handle_request() -> anyhow::Result<()> {
    let result = do_work().await?;
    Ok(())
}
```

**Consistency:**
- Domain errors: `thiserror` (typed)
- Application errors: `anyhow` (erased)
- Context: `.context()` for additional info

---

## Separation of Concerns Strategy

### 1. Horizontal Concerns

**HTTP Layer (`loom-server-api`):**
- Request/response types only
- No business logic
- Serialization/deserialization

**Business Logic Layer (`loom-server-*`):**
- Domain logic and rules
- No HTTP details
- Use domain types, not HTTP types

**Data Layer (`loom-server-db`):**
- Database operations only
- No business rules
- Returns domain types

---

### 2. Cross-Cutting Concerns

**Handled via Middleware:**
- Authentication: `auth_middleware`
- Authorization: `RequireRole` (ABAC)
- Audit logging: `audit_middleware`
- Tracing: `query_tracing`
- Metrics: `query_metrics`

**Benefits:**
- Don't Repeat Yourself (DRY)
- Centralized logic
- Easy to add/remove

---

### 3. Configuration Management

**Layered Configuration (Precedence):**
1. Defaults (built-in)
2. Config file (`/etc/loom/server.toml`)
3. Environment variables (`LOOM_SERVER_*`)
4. CLI flags (override all)

**Benefits:**
- Flexible deployment
- Development-friendly
- 12-factor app compatible

---

## Technology Stack Rationale

### Rust

**Why Rust?**
- **Performance:** Zero-cost abstractions, no GC
- **Safety:** Memory safety, thread safety at compile time
- **Concurrency:** Async/await with tokio runtime
- **Type System:** Expressive types prevent bugs
- **Ecosystem:** Excellent crates (tokio, serde, sqlx, etc.)

**Trade-offs:**
- Compile time (mitigated by cargo2nix caching)
- Learning curve (powerful once mastered)

---

### SQLite

**Why SQLite?**
- **Embedded:** No separate database server needed
- **Reliability:** ACID transactions, crash recovery
- **Portability:** Single file database
- **Performance:** Fast for typical Loom workloads
- **FTS5:** Built-in full-text search

**Scaling Strategy:**
- Single-tenant architecture (one DB per deployment)
- Horizontal scaling at application layer
- Future: PostgreSQL option for multi-tenant

---

### Axum (HTTP Framework)

**Why Axum?**
- **Type-safe Routing:** `TypedRouter` prevents mistakes
- **Extractor Pattern:** Easy state extraction
- **Middleware:** Tower ecosystem compatibility
- **Async:** First-class async support

**Alternative Considered:** Actix-web (chose Axum for modern design)

---

### Tokio (Async Runtime)

**Why Tokio?**
- **De facto Standard:** Most compatible crate
- **Performance:** Highly optimized I/O
- **Ecosystem:** Works with all major crates
- **Utilities:** Time, syncing, channels

---

### Kubernetes (Weaver Execution)

**Why Kubernetes?**
- **Container Orchestration:** Manages pod lifecycle
- **Scaling:** Auto-scaling based on load
- **Isolation:** Separate namespaces per org
- **Monitoring:** Built-in health checks

---

### SvelteKit (Web Frontend)

**Why SvelteKit?**
- **Performance:** Compile-time framework (no runtime)
- **TypeScript:** Full type safety
- **SSR:** Server-side rendering for SEO
- **Svelte 5 Runes:** Reactive by default

---

## Security Patterns

### 1. Secret Redaction

**Location:** `loom-common-secret/src/lib.rs`

**Implementation:**
```rust
pub struct Secret<T> {
    inner: T,
}

impl<T: Debug> Debug for Secret<T> {
    fn fmt(&self, f: &mut Formatter<'_>) -> fmt::Result {
        write!(f, "REDACTED")
    }
}
```

**Protection:**
- Auto-redaction in `Debug`, `Display`, `Serialize`
- Tracing integration (never logs secrets)
- Zeroization on drop (secure memory clearing)

---

### 2. Envelope Encryption

**Location:** `loom-server-secrets/src/lib.rs`

**Purpose:** Encrypt secrets at rest with key rotation

**Algorithm:**
1. Generate random data encryption key (DEK)
2. Encrypt secret with DEK (AES-256-GCM)
3. Encrypt DEK with master key (AWS KMS or software key)
4. Store encrypted DEK + encrypted secret

**Benefits:**
- Key rotation without re-encrypting all secrets
- Master key compromise: Rotate DEKs
- Compartmentalization

---

### 3. Path Traversal Prevention

**Location:** Tool implementations (`edit_file`, `bash`)

**Implementation:**
```rust
fn validate_path(path: &Path, workspace_root: &Path) -> Result<Path> {
    let canonical = path.canonicalize()?;
    let workspace_canonical = workspace_root.canonicalize()?;

    if !canonical.starts_with(&workspace_canonical) {
        return Err(PathOutsideWorkspace(canonical));
    }
    Ok(canonical)
}
```

**Protection:**
- Canonical path resolution (prevents `../` attacks)
- Workspace boundary enforcement
- User cannot escape workspace

---

### 4. Authentication & Authorization

**Authentication (AuthN):**
- Session-based tokens
- Secure random token generation
- Token hashing in database
- HTTP-only cookies (prevent XSS)

**Authorization (AuthZ):**
- Attribute-Based Access Control (ABAC)
- Role-based permissions (system_admin, support, auditor)
- Resource-level ownership checks
- Organization membership validation

**Middleware Stack:**
```
request → auth_middleware → abac_middleware → audit_middleware → handler
```

---

### 5. Audit Logging

**Location:** `loom-server-audit/src/lib.rs`

**Purpose:** Immutable audit trail for security events

**Implementation:**
- Async queue (buffer before database)
- Filtered by configured rules
- Enrichment (geoip, user lookup)
- Overflow policy (drop newest vs block)

**Audited Events:**
- Authentication (login, logout)
- Authorization failures
- Thread access
- Secret access
- Admin actions
- Configuration changes

---

### 6. Content Security Policy

**WebSocket Token:**
- Short-lived (30 seconds)
- Single-use
- Separate from session token
- Prevents token theft from XSS

---

## Performance Optimizations

### 1. Database Query Optimization

**Indexes:**
```sql
CREATE INDEX idx_threads_workspace_activity
  ON threads (workspace_root, last_activity_at DESC)
  WHERE deleted_at IS NULL;

CREATE INDEX idx_threads_pinned
  ON threads (is_pinned, last_activity_at DESC)
  WHERE deleted_at IS NULL;
```

**Partial Indexes:** Only index active threads
**Covering Indexes:** Include frequently accessed columns

---

### 2. Connection Pooling

**Location:** `loom-server-db/src/lib.rs`

**Implementation:**
```rust
SqlitePoolOptions::new()
    .max_connections(10)
    .min_connections(1)
    .connect_timeout(Duration::from_secs(30))
    .create_pool(pool)
```

**Benefits:**
- Reuse connections across requests
- Limit database load
- Faster request handling

---

### 3. Full-Text Search (FTS5)

**Location:** Thread search

**Implementation:**
- Separate virtual table `threads_fts`
- BM25 ranking algorithm
- Incremental updates (triggers)

**Performance:**
- Sub-second search for 100K+ threads
- Relevance ranking
- Prefix search support

---

### 4. Async Processing

**Analytics Events:**
- Immediate `202 Accepted` response
- Queue for async processing
- Batch inserts (100 events)

**Email Sending:**
- Background job queue
- Retry with exponential backoff
- Rate limiting

---

### 5. Streaming Responses

**LLM Streaming:**
- Server-Sent Events (SSE)
- Real-time text deltas
- No buffering entire response

**Benefits:**
- Lower time-to-first-token
- Better UX (progressive rendering)
- Lower memory usage

---

### 6. Caching Strategy

**Thread List:**
- In-memory cache of recent threads
- TTL-based invalidation
- Cache-aside pattern

**Feature Flags:**
- In-memory flag cache
- SSE invalidation
- Low-latency evaluation

---

---

## Scalability Approaches

### 1. Horizontal Scaling Strategy

**Application Layer:**
- **Stateless Design:** All application state is stored in the database
- **Load Balancer:** Multiple `loom-server` instances behind a load balancer
- **Session Affinity:** Not required (session tokens stored in database)
- **Graceful Shutdown:** Signal handling drains connections before exit

**Implementation:**
```rust
// Server configuration for horizontal scaling
pub struct ServerConfig {
    pub bind_address: SocketAddr,
    pub db_pool_size: u32,           // Per-instance pool
    pub max_concurrent_streams: usize, // Stream limits
}
```

**Scaling Benefits:**
- Add/remove instances without data migration
- Blue-green deployments
- Zero-downtime updates

---

### 2. Database Scaling

**Current Architecture: SQLite**
- **Single-Database-per-Deployment:** Each tenant has their own SQLite file
- **Connection Pooling:** 10 connections per instance
- **Optimistic Concurrency:** Version-based conflict detection

**Scaling Path:**

**Phase 1: Vertical Scaling (Current)**
- Larger server instances
- More memory for cache
- Faster storage (NVMe SSDs)

**Phase 2: Read Replicas (Planned)**
- Primary instance handles writes
- Read replicas handle SELECT queries
- Leader election for failover

**Phase 3: Sharding (Future)**
- Shard by `org_id` (organization)
- Each shard has its own SQLite database
- Router determines shard based on user's org

**Phase 4: PostgreSQL Migration (Future)**
- For write-heavy workloads
- Native multi-master replication
- Connection pooling via PgBouncer

**Trade-offs:**
- SQLite: Simpler operations, no replication, single-writer limitation
- PostgreSQL: Complex ops, built-in replication, multiple writers

---

### 3. Caching Layers

**In-Memory Caching:**

**Thread List Cache:**
```rust
pub struct ThreadListCache {
    cache: Arc<RwLock<HashMap<WorkspaceKey, CachedThreadList>>>,
    ttl: Duration,
}

impl ThreadListCache {
    pub async fn get(&self, workspace: &Path) -> Option<Vec<Thread>> {
        let cache = self.cache.read().await;
        cache.get(workspace).filter(|c| c.is_fresh()).map(|c| c.threads.clone())
    }
}
```

**Feature Flag Cache:**
- In-memory copy of all flags
- SSE-based invalidation
- Per-instance cache (no distributed cache needed)

**Cache Invalidation Strategies:**
1. **TTL-based:** Thread lists expire after 60 seconds
2. **Event-based:** Feature flags invalidate via SSE
3. **Write-through:** Database writes immediately update cache

**Future: Distributed Cache (Redis)**
- Session store for multi-instance deployments
- Shared rate limiting state
- Cross-instance cache invalidation

---

### 4. Content Delivery Network (CDN)

**Static Assets:**
- Web frontend assets served via CDN
- JavaScript bundles cached at edge
- Images and CSS cached long-term

**CDN Strategy:**
```toml
[cdn]
# Cache static assets for 1 year
cache_control = "public, max-age=31536000, immutable"

# Cache HTML for 1 hour
html_cache_control = "public, max-age=3600"
```

**API Caching:**
- Public endpoints (health, docs) cacheable
- Private endpoints never cached (privacy)
- ETags for conditional requests

---

### 5. Background Job Processing

**Current Approach: Async Tasks**
```rust
// Analytics event processing
pub async fn process_events_batch(events: Vec<AnalyticsEvent>) {
    tokio::spawn(async move {
        insert_events_batch(events).await;
        update_aggregations().await;
    });
}
```

**Future: Dedicated Job Queue**
- **Job Queue:** PostgreSQL-backed or Redis
- **Worker Pool:** Separate worker processes
- **Job Types:**
  - Analytics event aggregation
  - Email sending
  - Weaver cleanup (expired pods)
  - Report generation

**Benefits:**
- Isolate long-running tasks from HTTP handlers
- Retry with exponential backoff
- Dead letter queue for failed jobs

---

### 6. Multi-Tenancy Considerations

**Tenant Isolation:**

**Data Isolation:**
- **Organization-based:** All data scoped to `org_id`
- **User-based:** Personal data scoped to `user_id`
- **Workspace-based:** Threads scoped to `workspace_root`

**Execution Isolation:**
- **Weaver Namespaces:** Each org gets its own K8s namespace
- **Resource Quotas:** CPU/memory limits per org
- **Network Policies:** Prevent cross-org communication

**Configuration Isolation:**
- **Per-Org Feature Flags:** Org-specific flag overrides
- **Rate Limits:** Per-org rate limit tiers
- **Secrets:** Org-scoped secret storage

**Multi-Tenant Architecture:**
```
┌───────────────────────────────────────────────┐
│              Load Balancer                    │
└───────────────────────────────────────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Loom Server 1 │ │ Loom Server 2 │ │ Loom Server N │
└───────────────┘ └───────────────┘ └───────────────┘
        │              │              │
        └──────────────┼──────────────┘
                       │
        ┌──────────────┼──────────────┐
        │              │              │
┌───────────────┐ ┌───────────────┐ ┌───────────────┐
│ Org A DB      │ │ Org B DB      │ │ Org C DB      │
│ (SQLite)      │ │ (SQLite)      │ │ (SQLite)      │
└───────────────┘ └───────────────┘ └───────────────┘
```

---

### 7. Performance Optimization Techniques

**Database Query Optimization:**

**Partial Indexes:**
```sql
-- Only index active threads
CREATE INDEX idx_threads_active
  ON threads (workspace_root, last_activity_at DESC)
  WHERE deleted_at IS NULL;
```

**Covering Indexes:**
```sql
-- Include frequently accessed columns
CREATE INDEX idx_threads_search
  ON threads (workspace_root, title, id, last_activity_at)
  WHERE deleted_at IS NULL;
```

**Full-Text Search:**
- FTS5 virtual table for thread search
- BM25 ranking algorithm
- Sub-second search for 100K+ threads

**Application-Level Optimizations:**

**Connection Pooling:**
```rust
SqlitePoolOptions::new()
    .max_connections(10)
    .min_connections(1)
    .connect_timeout(Duration::from_secs(30))
```

**Streaming Responses:**
- LLM responses streamed via SSE
- No buffering entire response
- Lower time-to-first-token

**Async Operations:**
- Concurrent tool execution
- Parallel database queries
- Background event processing

---

### 8. Monitoring and Observability

**Metrics Collection:**

**Application Metrics:**
- Request rate and latency
- Error rate by endpoint
- Active connections
- Cache hit/miss ratio

**Business Metrics:**
- Active threads per workspace
- LLM token usage
- Tool execution success rate
- User engagement

**Logging Strategy:**
- Structured logging with `tracing`
- Correlation IDs for request tracing
- Log levels: ERROR, WARN, INFO, DEBUG, TRACE
- Sampling for high-volume logs

**Distributed Tracing:**
```rust
#[instrument(skip(self, secrets), fields(user_id = %user_id))]
pub async fn handle_request(&self, user_id: UserId) -> Result<()> {
    // Automatic trace span creation
}
```

---

### 9. Disaster Recovery

**Backup Strategy:**

**Database Backups:**
- **SQLite:** File-level snapshots to S3
- **Frequency:** Hourly incremental, daily full
- **Retention:** 30 days

**Configuration Backups:**
- Server configuration (TOML)
- Kubernetes manifests
- NixOS system config

**Recovery Procedures:**
1. **Restore Database:** Copy SQLite file from backup
2. **Restart Server:** Automatic migration on startup
3. **Verify:** Health check endpoint
4. **Monitor:** Check logs for errors

**High Availability:**
- **Load Balancer:** Health checks remove unhealthy instances
- **Graceful Shutdown:** Drain connections before exit
- **Rolling Updates:** Zero-downtime deployments

---

**Phase 6 Status:** COMPLETE

**Documented:**
- ✓ Overall architecture (Layered modular monolith)
- ✓ Design patterns (8 patterns documented)
- ✓ Code organization philosophy
- ✓ Separation of concerns strategy
- ✓ Technology stack rationale
- ✓ Security patterns (6 patterns documented)
- ✓ Performance optimizations (6 optimizations documented)
- ✓ Scalability approaches (9 approaches documented)

**Next:** Create architecture-diagram.mermaid
