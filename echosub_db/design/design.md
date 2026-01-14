# EchoSub Database Module - High-Level Design

## Overview

The EchoSub database module (`github.com/prethora/echosub_db`) is a Go package providing type-safe PostgreSQL operations for the EchoSub subtitle platform. The module manages video fingerprints, transcriptions, translations, and job state for the processing pipeline.

**Key Design Principle**: Leverage specialized tools for schema management, code generation, and database connectivity rather than building custom abstractions. This creates a maintainable, predictable system with minimal runtime magic.

## High-Level Architecture

The module is organized around four conceptual layers:

### 1. Schema Layer (Atlas)
**Purpose**: Declarative schema definition and migration management

Atlas defines the database structure in `schema.hcl` using HCL (HashiCorp Configuration Language). This declarative approach treats the schema as the source of truth, with migrations derived from schema changes rather than hand-written.

**Workflow Integration**:
- Developers modify `schema.hcl` to define or change tables, indexes, constraints
- `atlas migrate diff` compares the schema against the current database state and generates migration files
- `atlas migrate apply` executes migrations against target environments (local dev, test, production)
- Generated migrations are versioned SQL files stored in `migrations/` directory

**Rationale**: Declarative schemas eliminate drift between environments and make schema evolution explicit. Migration generation is deterministic and repeatable.

### 2. Query Layer (sqlc)
**Purpose**: Type-safe query execution through code generation

sqlc generates Go code from SQL queries, providing compile-time type safety without the runtime overhead and complexity of ORMs.

**Workflow Integration**:
- Developers write SQL queries in `queries/*.sql` files (organized by domain: videos.sql, transcriptions.sql, etc.)
- `sqlc generate` reads these queries and the database schema to produce Go code
- Generated code lives in `generated/` directory (excluded from version control, regenerated on build)
- Generated code includes:
  - `models.go`: Struct types matching database tables
  - `db.go`: Base Queries struct with database connection
  - `*.sql.go`: Methods for each query with proper parameter and return types

**Rationale**: Hand-written SQL with generated type-safe Go bindings provides predictability (you see the exact queries) with safety (compile errors for schema mismatches). No hidden query generation or N+1 problems.

### 3. Connection Layer (pgx)
**Purpose**: PostgreSQL driver and connection pooling

pgx is the underlying PostgreSQL driver that handles connections, query execution, and result parsing. It provides a connection pool (`pgxpool.Pool`) that manages database connections efficiently.

**Integration**:
- The module's public `DB` struct wraps a `*pgxpool.Pool` instance
- Pool is initialized from `DATABASE_URL` environment variable
- Pool configuration is handled by pgx defaults (in production, Supabase's serverless URL includes connection pooling parameters)
- All database operations go through this pool

**Rationale**: pgx is the de facto standard PostgreSQL driver for Go with excellent performance and PostgreSQL-specific feature support. Connection pooling is essential for Lambda functions with concurrent requests, but the module relies on provider infrastructure (Supabase) rather than implementing custom pooling logic.

### 4. Public API Layer (Hand-Written)
**Purpose**: Clean, consumer-facing API that hides implementation details

The `db.go` file provides the public API surface:
- `DB` struct (wraps pgxpool.Pool and sqlc's generated Queries)
- `New(ctx, databaseURL)` constructor
- Domain methods that delegate to sqlc-generated query methods
- Error mapping (translating pgx errors to module sentinel errors)

**Integration with Generated Code**:
```
┌─────────────────────────────────────┐
│  Public API (db.go)                 │
│  - DB struct                        │
│  - Domain methods                   │
│  - Error mapping                    │
└──────────┬──────────────────────────┘
           │ delegates to
           ▼
┌─────────────────────────────────────┐
│  Generated Code (generated/)        │
│  - Queries struct (sqlc)            │
│  - Query methods (sqlc)             │
│  - Model types (sqlc)               │
└──────────┬──────────────────────────┘
           │ executes via
           ▼
┌─────────────────────────────────────┐
│  Connection Pool (pgxpool)          │
│  - Manages PostgreSQL connections   │
└─────────────────────────────────────┘
```

**Rationale**: Consumers import the package and work with a simple `DB` instance. They don't need to know about sqlc, connection pools, or internal details. The public API can evolve (adding caching, metrics, etc.) without breaking consumers.

## Tool Integration Summary

| Tool | Input | Output | Role |
|------|-------|--------|------|
| **Atlas** | `schema.hcl` | SQL migration files in `migrations/` | Schema source of truth, migration generation |
| **sqlc** | `queries/*.sql` + database schema | Go code in `generated/` | Type-safe query execution |
| **pgx** | `DATABASE_URL` | Connection pool | PostgreSQL driver and connection management |

These tools work together in a build-time → runtime flow:
1. **Design Time**: Developer defines schema in Atlas, writes queries for sqlc
2. **Build Time**: Atlas generates migrations, sqlc generates Go code
3. **Runtime**: pgx connects to database, generated code executes queries

The module presents a unified API that abstracts away this toolchain complexity from consumers.

## Repository Structure

```
prethora/echosub_db/
├── schema.hcl                # Atlas schema definition (committed)
├── atlas.hcl                 # Atlas configuration (committed)
├── migrations/               # Generated migrations (committed)
│   └── *.sql
├── queries/                  # SQL queries for sqlc (committed)
│   ├── videos.sql
│   ├── transcriptions.sql
│   ├── translations.sql
│   └── jobs.sql
├── sqlc.yaml                 # sqlc configuration (committed)
├── db.go                     # Public API, hand-written (committed)
├── generated/                # sqlc output (NOT committed, regenerated)
│   ├── models.go
│   ├── db.go
│   └── *.sql.go
├── docker/                   # Local dev environment (committed)
│   └── psql/
├── Makefile                  # Development commands (committed)
└── README.md                 # Documentation (committed)
```

**Version Control Strategy**:
- **Committed**: Schema definitions, queries, migrations, hand-written code, configuration
- **Ignored**: Generated code (rebuilt from source during development and CI/CD)

This separation ensures that the source of truth (schema and queries) is versioned, while generated artifacts can be reproduced deterministically.

## External Dependencies

### Runtime Dependencies

The module has minimal runtime dependencies, following Go best practices of depending on stable, well-maintained libraries.

| Dependency | Version | Import Path | Rationale |
|------------|---------|-------------|-----------|
| **pgx** | v5.5.0+ | `github.com/jackc/pgx/v5` | De facto standard PostgreSQL driver for Go. Excellent performance, native PostgreSQL protocol implementation, comprehensive type support including JSONB. The v5 API is stable and widely adopted. |
| **pgxpool** | v5.5.0+ | `github.com/jackc/pgx/v5/pgxpool` | Connection pool implementation from the pgx ecosystem. Provides concurrent-safe connection management with configurable pool sizing. |
| **google/uuid** | v1.5.0+ | `github.com/google/uuid` | Industry-standard UUID library. Used for generating UUIDs for all database primary keys. Simple, well-tested, no external dependencies. |

**Version Constraints**:
- Go: `1.21+` (required for pgx v5 and modern Go tooling)
- PostgreSQL: `16` (the module uses PostgreSQL 16-specific features like `bit_count()` for Hamming distance calculations)

### Development-Time Dependencies

These tools are required for the development workflow but are not Go package dependencies:

| Tool | Purpose | Installation |
|------|---------|--------------|
| **Atlas CLI** | Schema management and migration generation | Installed via go install or package manager |
| **sqlc CLI** | SQL to Go code generation | Installed via go install or package manager |
| **Docker** | Local PostgreSQL instance for development and testing | Standard Docker installation |

**Rationale for Tool Choices**:
- **Atlas over manual migrations**: Declarative schemas eliminate human error in migration writing and ensure schema-migration consistency
- **sqlc over ORMs**: Type-safe queries without runtime reflection overhead or hidden query generation
- **pgx over database/sql**: Direct PostgreSQL support, better performance, native handling of PostgreSQL types

## Connection Lifecycle

### Initialization

The `New(ctx context.Context, databaseURL string) (*DB, error)` constructor performs connection pool initialization:

1. Parses the `databaseURL` using pgxpool configuration parser
2. Creates a connection pool with pgx defaults:
   - Default max connections: determined by pgxpool based on system resources
   - Default connection timeout: 30 seconds
   - Default idle timeout: 30 minutes
3. Performs initial connectivity check (fails fast if database is unreachable)
4. Wraps the pool in the module's `DB` struct along with sqlc's generated `Queries` struct
5. Returns ready-to-use `DB` instance or connection error

**Fail-Fast Behavior**: If the database is unreachable or `DATABASE_URL` is malformed, `New()` returns an error immediately. The module does not implement retry logic during initialization—this is the caller's responsibility. In AWS Lambda, the function will fail and AWS will retry the invocation.

### Connection Pooling Strategy

The module **relies on external pooling infrastructure** rather than implementing custom pooling:

- **Development/Test**: pgxpool handles local connection management with default settings (sufficient for single-developer usage)
- **Production (Supabase)**: The `DATABASE_URL` points to Supabase's serverless connection URL, which includes Supabase's own connection pooling infrastructure. The module's pgxpool works in concert with Supabase's pooler.

**No Custom Pool Configuration**: The module uses pgxpool defaults. This keeps the implementation simple and relies on battle-tested defaults. If custom tuning becomes necessary (e.g., limiting Lambda concurrency), configuration can be exposed through constructor options in a future iteration.

### Thread Safety

The `DB` struct is **safe for concurrent use** across multiple goroutines:
- The underlying `pgxpool.Pool` is explicitly designed for concurrent access
- No mutable state exists in the `DB` struct beyond the pool
- All query methods accept `context.Context` for per-operation lifecycle management
- Multiple Lambda invocations can safely share a single `DB` instance (if Lambda reuses containers)

**Best Practice**: Create one `DB` instance per Lambda container (initialized outside the handler), reuse it across invocations.

### Cleanup

The `Close()` method gracefully shuts down the connection pool:
- Waits for in-flight queries to complete (respects context deadlines)
- Closes idle connections
- Releases resources

**Guarantees**:
- Calling `Close()` while queries are running is safe—those queries will complete
- After `Close()`, any new query attempts will return errors
- `Close()` is idempotent—calling it multiple times is safe

**Lambda Context**: In Lambda functions, explicit `Close()` calls are often unnecessary (the container shuts down when evicted). However, including `defer db.Close()` is good practice for test code and non-Lambda usage.

## Error Handling Strategy

### Error Categories

The module distinguishes between four error categories:

| Category | Cause | Recoverability | Examples |
|----------|-------|----------------|----------|
| **Connection Errors** | Cannot reach database, auth failure, network issues | Possibly transient (retry at caller level) | Invalid DATABASE_URL, database down, network partition |
| **Not Found Errors** | Query for specific record returns zero rows | Expected condition, not an error in many cases | GetVideo() for non-existent UUID, no transcription exists |
| **Constraint Violations** | Unique constraint, foreign key violation | Permanent (requires different input) | Duplicate video fingerprint, orphaned translation reference |
| **Query Errors** | Malformed query, type mismatch, invalid SQL | Permanent (indicates bug in module) | Should not occur with sqlc-generated code |

### Sentinel Errors

The module exports sentinel errors for programmatic error handling:

```go
var (
    // ErrNotFound indicates the requested record does not exist
    ErrNotFound = errors.New("echosubdb: not found")

    // ErrDuplicateKey indicates a unique constraint violation
    ErrDuplicateKey = errors.New("echosubdb: duplicate key")

    // ErrConnection indicates database connectivity failure
    ErrConnection = errors.New("echosubdb: connection failed")
)
```

**Usage**: Consumers use `errors.Is(err, echosubdb.ErrNotFound)` to check error types.

### Error Mapping

The public API methods translate pgx-specific errors to module sentinel errors:

**pgx Error → Sentinel Error Mapping**:
- `pgx.ErrNoRows` → `ErrNotFound`
- `pgconn.PgError` with code `23505` (unique violation) → `ErrDuplicateKey`
- `pgconn.PgError` with code `23503` (foreign key violation) → `ErrDuplicateKey`
- Connection errors (auth failure, host unreachable) → `ErrConnection`
- Other pgx errors → wrapped error with context (preserves stack trace)

**Rationale**: Consumers shouldn't need to import pgx or understand PostgreSQL error codes. Sentinel errors provide a stable API that won't break if the underlying driver changes.

### Connection Failure Behavior

**During Initialization (`New()`)**: Fails immediately with `ErrConnection`. No retries.

**During Query Execution**: If a connection is lost mid-query (e.g., database restarts), pgxpool will:
1. Return an error for the current query
2. Mark the connection as unhealthy
3. Attempt to establish a new connection for subsequent queries

The module does not implement application-level retry logic. Retries are the caller's responsibility (AWS Lambda retries automatically on failure).

**Design Decision**: Keep the module stateless and focused on database operations. Retry policies vary by use case (immediate retry vs exponential backoff) and should be implemented at the service layer.

## Data Storage and Query Strategies

### JSONB Segment Storage

Both `transcriptions` and `translations` tables store subtitle segments in a JSONB column rather than a normalized relational structure.

#### Storage Format

Segments are stored as a JSON array of objects:

```json
[
  {"start": 0.0, "end": 2.5, "text": "Hello, world."},
  {"start": 2.5, "end": 5.0, "text": "This is a subtitle."},
  {"start": 5.0, "end": 8.2, "text": "Another segment."}
]
```

Each segment object has three fields:
- `start`: Start time in seconds (float64)
- `end`: End time in seconds (float64)
- `text`: Subtitle text (string)

#### Marshaling/Unmarshaling Strategy

**sqlc handles JSONB automatically** when the Go type is a slice or struct with JSON tags:

- Query parameter: Pass `[]Segment` to sqlc-generated methods
- Return value: sqlc unmarshals JSONB → `[]Segment`
- No custom marshal/unmarshal logic needed in hand-written code

The `Segment` type defined in the public API:
```go
type Segment struct {
    Start float64 `json:"start"`
    End   float64 `json:"end"`
    Text  string  `json:"text"`
}
```

sqlc's generated code uses `encoding/json` to handle the JSONB ↔ Go conversion transparently.

#### Design Rationale: JSONB vs. Relational Table

**Why JSONB?**

1. **Atomic Write Semantics**: Transcriptions and translations are created as complete units. A transcription is never partially written—all segments arrive together from Whisper. JSONB ensures atomicity without transaction complexity.

2. **Read Access Pattern**: Consumers always fetch the entire segment list, never individual segments. There's no use case for "get segment 5 of transcription X." JSONB avoids JOIN overhead.

3. **Immutability**: Once created, transcriptions/translations are never modified segment-by-segment. They're created, read, and eventually deleted (when video is deleted). No UPDATE operations on individual segments.

4. **Simplicity**: A single column is simpler than a one-to-many relationship with a separate `segments` table, foreign keys, and ordering constraints.

**When to Use JSONB vs. Relational Tables**:
- **JSONB**: Data is accessed as a complete unit, immutable or rarely updated, no need for complex queries within the JSON structure
- **Relational**: Need to query/filter/sort by individual elements, frequent partial updates, referential integrity across JSON elements

For EchoSub's subtitle segments, JSONB is the right choice.

#### Performance Implications

- **Write**: Single INSERT instead of N INSERTs for N segments (faster, less lock contention)
- **Read**: Single column fetch instead of JOIN + ORDER BY (faster, simpler query plan)
- **Storage**: Comparable to relational (JSONB is binary format, not text JSON)
- **Indexing**: Not needed—we never query within the segments array (we retrieve the whole array)

### Fingerprint Matching Implementation

Video identification uses perceptual hashing with Hamming distance similarity matching.

#### Hamming Distance Calculation

PostgreSQL 16 provides `bit_count()` function for popcount operations. A custom SQL function wraps this:

```sql
CREATE FUNCTION hamming_distance(a BIGINT, b BIGINT) RETURNS INTEGER AS $$
    SELECT bit_count(a # b)::INTEGER;
$$ LANGUAGE SQL IMMUTABLE PARALLEL SAFE;
```

- `#` is XOR operator (differences between bit patterns)
- `bit_count()` counts set bits (number of differing bits)
- `IMMUTABLE` allows PostgreSQL to optimize and parallelize
- `PARALLEL SAFE` enables parallel query execution

This function is defined in the Atlas schema and included in migrations.

#### Fingerprint Matching Algorithm

The `FindVideosByFingerprint()` query executes a single SQL query with this logic:

1. **Duration Filtering**: `WHERE duration_ms BETWEEN (target - 100) AND (target + 100)`
   - Narrow candidates to videos within ±100ms of target duration
   - Uses index on `duration_ms` for fast filtering

2. **Hamming Distance Calculation**: For each of the 5 hash positions (10%, 25%, 50%, 75%, 90%):
   - Compute `hamming_distance(videos.hash_10, target.hash_10)` for each position
   - Returns integer distance (0-64 bits for BIGINT)

3. **Average Distance**: `(dist_10 + dist_25 + dist_50 + dist_75 + dist_90) / 5.0`
   - Average across all 5 positions
   - Smooths out single-position anomalies (e.g., scene change at 50% position)

4. **Threshold Filtering**: `WHERE avg_distance <= maxAvgDistance`
   - Typically `maxAvgDistance = 6.0` (caller-specified)
   - Allows ~10% bit error across all positions

5. **Ordering**: `ORDER BY avg_distance ASC`
   - Best matches first
   - Caller can take top N results

**Single Query Implementation**: All steps execute in one SQL query. PostgreSQL computes distances in-memory for the filtered candidate set (duration filter reduces candidates to manageable size before expensive Hamming distance calculations).

#### Design Rationale

**Why 5 hash positions?**
- Temporal distribution: Captures visual characteristics across the video timeline
- Robustness: Scene changes at one position don't invalidate the entire fingerprint
- Balance: More positions = more precision but more storage and computation

**Why Hamming distance over other similarity metrics?**
- Perceptual hash-compatible: Hamming distance is the standard metric for perceptual hashes (similar images have low Hamming distance)
- PostgreSQL-native: `bit_count()` is fast (hardware popcount instruction)
- Predictable: Distance is bounded (0-64 for BIGINT), easy to reason about thresholds

**Why average distance instead of minimum or maximum?**
- Average is robust to outliers (one bad hash position doesn't disqualify an otherwise-good match)
- Matches common practice in perceptual hashing literature

**Why BIGINT storage instead of BIT(64)?**
- BIGINT has better tooling support (easier to work with in Go and PostgreSQL)
- XOR and bit_count operations are native for integers
- Performance is equivalent

### Database Schema Design Decisions

#### UUID Generation Strategy

**UUIDs are generated application-side**, not by the database:

- Go code uses `google/uuid` to generate UUIDv4 values before INSERT
- Database columns are `UUID` type but have no `DEFAULT` value
- sqlc-generated code expects UUID strings as parameters

**Rationale**:
- **Consistency**: All record IDs are known immediately after creation in Go code (useful for logging, returning to caller without SELECT)
- **Testability**: Tests can provide deterministic UUIDs for reproducibility
- **Flexibility**: Easy to switch UUID versions (v4, v7, etc.) without schema changes
- **No Database Round-Trip**: Client doesn't need to query `RETURNING id` to learn the generated UUID (though we use RETURNING for consistency anyway)

**Trade-off**: Application must remember to generate UUIDs. sqlc enforces this at compile time (required parameter).

#### CASCADE Delete Behavior

Foreign keys enforce referential integrity with CASCADE deletes:

```
videos (id)
  ↓ ON DELETE CASCADE
transcriptions (video_id)
  ↓ ON DELETE CASCADE
translations (video_id, transcription_id)

jobs (video_id)
  ↓ ON DELETE SET NULL
```

**Semantics**:
- Deleting a video automatically deletes its transcription and all translations
- Deleting a transcription automatically deletes all translations derived from it
- Deleting a video sets `jobs.video_id` to NULL (job record remains for historical tracking)

**Rationale**:
- **Data Integrity**: Prevents orphaned records (translation without video)
- **Convenience**: Application doesn't need to manually delete dependent records
- **Performance**: Database handles cascade in a single transaction (more efficient than application-side loops)

**When NOT to CASCADE**: Jobs table uses SET NULL instead of CASCADE because jobs track processing history. A deleted video shouldn't erase the fact that a job was attempted.

#### Nullable Field Strategy

Optional fields use **nullable database columns** with **pointer types in Go**:

| Table | Nullable Columns | Go Type |
|-------|------------------|---------|
| videos | `youtube_id`, `title` | `*string` |
| jobs | `video_id`, `error_message`, `audio_key`, `audio_duration_ms`, `started_at`, `completed_at` | `*string`, `*int32`, `*time.Time` |

**Marshaling**: sqlc automatically generates pointer types for nullable columns. Consumers check `if title != nil` before dereferencing.

**Rationale**:
- **Semantic Clarity**: `nil` distinctly represents "absent" vs. `""` (empty string) or `0` (zero value)
- **Database Fidelity**: Go types match database nullability exactly (no confusion about zero values)
- **API Ergonomics**: Callers explicitly handle optional fields (compile-time safety via type system)

**Example**:
```go
video, _ := db.GetVideo(ctx, id)
if video.YoutubeID != nil {
    fmt.Println("YouTube ID:", *video.YoutubeID)
} else {
    fmt.Println("No YouTube ID (fingerprint-only match)")
}
```

#### Database Constraints vs. Application Validation

**Enforced at Database Level** (constraints in schema):
- **NOT NULL**: Required fields (video duration, transcription language, job status)
- **UNIQUE**: `videos.youtube_id` (where not null), `(video_id, language)` for translations
- **FOREIGN KEY**: Referential integrity (video_id references videos)
- **CHECK**: Enum values (job status must be one of predefined values—though PostgreSQL doesn't enforce this natively, it's documented)

**Enforced at Application Level** (validation before INSERT/UPDATE):
- **Business Rules**: "Language code must be ISO 639-1 format" (2-letter codes)
- **Semantic Validation**: "Segments must be in chronological order" (non-overlapping timestamps)
- **External Constraints**: "Transcription exists before creating translation" (checked via query)

**Rationale**:
- **Database constraints** protect data integrity even if buggy code or manual SQL bypasses the application
- **Application validation** provides better error messages and can enforce complex rules (e.g., timestamp ordering) that SQL constraints can't express cleanly
- **Separation of Concerns**: Database ensures correctness, application ensures usability

**Design Philosophy**: Use database constraints for structural integrity (types, nullability, uniqueness, referential integrity). Use application code for business logic and usability (meaningful errors, complex validation).

## Testing Strategy

### Testing Approach Overview

The module's testing strategy is **integration-test focused** rather than unit-test heavy.

**Why Integration Tests Are Primary**:

This module is a thin wrapper around database operations. The value—and risk—lies in:
- Correct SQL queries (schema compatibility, proper JOINs, correct WHERE clauses)
- Proper marshaling/unmarshaling (JSONB ↔ Go types, UUID handling)
- Connection handling (pool lifecycle, error propagation)

Unit tests would require mocking the database layer, which duplicates the module's implementation logic without actually verifying correctness against PostgreSQL. Integration tests directly verify that queries execute correctly against a real database.

**Test Type Distribution**:
- **90% Integration Tests**: All CRUD operations, fingerprint matching, error handling against real PostgreSQL
- **10% Unit Tests**: Pure logic (error mapping functions, validation helpers) that don't require database access

### Integration Testing Approach

Integration tests run against a real PostgreSQL 16 instance in Docker.

#### Test Database Setup

**Environment**:
- PostgreSQL 16 in Docker (same container used for development)
- Separate `test` database (isolated from `development` database)
- Connection via `DATABASE_URL` environment variable:
  ```
  DATABASE_URL="postgres://postgres:123456@localhost:5432/test?sslmode=disable"
  ```

**Schema Initialization**:
Before running tests, migrations must be applied to the test database:
```bash
atlas migrate apply --env test
```

This ensures the test database schema matches what sqlc expects. The `Makefile` includes a `test` target that handles this automatically.

#### Test Structure

Each integration test follows this pattern:

```go
func TestCreateVideo(t *testing.T) {
    // Setup: Connect to test database
    ctx := context.Background()
    db, err := echosubdb.New(ctx, os.Getenv("DATABASE_URL"))
    require.NoError(t, err)
    defer db.Close()

    // Test: Execute operation
    video := echosubdb.Video{
        ID: uuid.New().String(),
        DurationMs: 60000,
        Hash10: 0x1234567890abcdef,
        // ... other fields
    }
    created, err := db.CreateVideo(ctx, video)

    // Assert: Verify results
    require.NoError(t, err)
    assert.Equal(t, video.ID, created.ID)
    assert.Equal(t, video.DurationMs, created.DurationMs)
}
```

**Key Testing Libraries**:
- `testing` (standard library): Test framework
- `github.com/stretchr/testify/assert` and `require`: Assertions (require fails fast, assert continues)

### Test Isolation and Fixtures

#### Isolation Strategy

Tests achieve isolation through **table truncation** after each test:

```go
func TestMain(m *testing.M) {
    // Run tests
    code := m.Run()

    // Cleanup: Truncate all tables
    cleanupTestDatabase()

    os.Exit(code)
}

func cleanupTestDatabase() {
    // TRUNCATE videos CASCADE (cascades to transcriptions, translations)
    // TRUNCATE jobs
}
```

**Why Truncation Over Transactions**:
- Simplicity: No need to wrap every test in BEGIN/ROLLBACK
- Matches Production Behavior: Tests see committed data, same as production
- Migration Testing: Can verify CASCADE delete behavior directly

**Trade-off**: Tests cannot run in parallel (truncation would interfere). Tests must run sequentially with `go test -p 1`.

#### Fixture Management

Test data is created using **the module's own public API methods**:

```go
// Create a video fixture
video := createVideoFixture(t, db)

// Create a transcription fixture for that video
transcription := createTranscriptionFixture(t, db, video.ID)

// Now test translation creation
translation, err := db.CreateTranslation(ctx, echosubdb.Translation{
    VideoID: video.ID,
    TranscriptionID: transcription.ID,
    Language: "es",
    Segments: []echosubdb.Segment{{Start: 0, End: 1, Text: "Hola"}},
})
```

**Rationale**:
- **Dogfooding**: Tests exercise the same API consumers use (finds API ergonomics issues)
- **DRY**: No duplication of INSERT logic in test helpers vs production code
- **Refactoring Safety**: If API changes, tests break (alerting to breaking changes)

**When to Use Direct SQL**: For negative tests (inserting malformed data to verify error handling), use direct SQL to bypass API validation.

#### Parallel Execution

**Tests run sequentially** (`-p 1` flag):
- Table truncation makes parallel execution unsafe
- Single database instance would serialize anyway (lock contention)
- Test suite is small enough that sequential execution is fast (<10 seconds)

**Future Optimization**: If test suite grows large, use per-test database snapshots or isolated containers (e.g., testcontainers-go).

### Testing Code Generation (sqlc)

#### Approach

**sqlc-generated code is not directly unit tested**. Instead:

1. **Implicit Testing Through Integration Tests**: When integration tests call `db.CreateVideo()`, they're testing sqlc's generated `CreateVideo` query method. If sqlc generates incorrect SQL, the integration test fails.

2. **Schema-Query Compatibility**: sqlc validates queries against the database schema during `sqlc generate`. If a query references a non-existent column or uses incompatible types, sqlc fails at build time (before tests even run).

3. **CI Verification**: The build process runs:
   ```bash
   sqlc generate  # Regenerate code
   git diff --exit-code generated/  # Verify no unexpected changes
   ```
   This catches cases where schema changes invalidate existing queries.

**Confidence Level**: High. The combination of sqlc's compile-time validation + integration tests against real PostgreSQL provides strong guarantees that generated code is correct.

**When Generated Code Might Be Wrong**: If sqlc has a bug (rare, but possible), integration tests will catch it (e.g., generated query returns wrong columns, fails to unmarshal JSONB correctly).

### Testing Atlas Migrations

#### Migration Up Testing

**Automated via Integration Tests**:
- Test setup runs `atlas migrate apply --env test` before tests
- If migration fails, tests cannot run (immediate failure)
- Integration tests indirectly verify schema correctness (queries succeed only if schema is correct)

**Manual Verification During Development**:
When creating a new migration with `atlas migrate diff`:
1. Inspect generated SQL in `migrations/` directory (sanity check)
2. Apply to development database: `atlas migrate apply --env local`
3. Verify application still works (run a service locally)
4. Apply to test database and run test suite

#### Migration Down Testing

**Atlas migrations are append-only** (no rollback in production):
- Production migrations only move forward (new migrations are additive)
- Rollback is handled by deploying a new forward migration that undoes changes

**Why No Rollback**:
- Rollback is risky in production (data loss, especially with CASCADE deletes)
- Forward-only migrations are safer (explicit, reviewable, testable)
- If a migration breaks production, fix-forward with a new migration

**Development Rollback**: For local development, `atlas migrate down` can undo migrations, but this is not tested or used in production workflows.

#### Idempotency Verification

**Atlas ensures idempotency automatically**:
- `atlas migrate apply` tracks applied migrations in a `atlas_schema_revisions` table
- Re-running `apply` skips already-applied migrations (no-op)
- Safe to run `atlas migrate apply` multiple times (e.g., in CI/CD pipelines)

**Testing**: Not explicitly tested (Atlas's responsibility). Developers can manually verify by running `atlas migrate apply` twice and observing no changes.

### Test Coverage Goals

**Target Coverage**:
- **100% of public API methods**: Every exported function in `db.go` has at least one integration test
- **Happy Path**: All CRUD operations work correctly
- **Error Cases**: Not found (query non-existent ID), duplicate key (insert duplicate), connection errors (invalid DATABASE_URL)
- **Edge Cases**: Fingerprint matching with no matches, transcription exists check, nullable fields

**Not Covered**:
- Database crashes mid-query (pgxpool handles this, no need to test)
- Network partitions (infrastructure-level concern)
- Concurrency stress tests (pgxpool is already battle-tested)

**Confidence Level**: High coverage of business logic, relying on pgx and PostgreSQL's own testing for infrastructure concerns.

## Local Development Setup

### Docker PostgreSQL Configuration

The module includes Docker configuration for running PostgreSQL 16 locally with multi-database support.

#### Dockerfile (`docker/psql/Dockerfile`)

```dockerfile
FROM postgres:16
COPY init-multiple-databases.sh /docker-entrypoint-initdb.d/
RUN chmod +x /docker-entrypoint-initdb.d/init-multiple-databases.sh
```

**Base Image**: Official PostgreSQL 16 image from Docker Hub (maintained by Postgres community)

**Initialization Script**: Copies a script that creates multiple databases on container startup.

#### Multi-Database Init Script (`docker/psql/init-multiple-databases.sh`)

```bash
#!/bin/bash
set -e
set -u

function create_database() {
  local database=$1
  echo "Creating database '$database'"
  psql -v ON_ERROR_STOP=1 --username "$POSTGRES_USER" --dbname "$POSTGRES_DB" <<-EOSQL
CREATE DATABASE $database;
EOSQL
}

if [ -n "$POSTGRES_MULTIPLE_DATABASES" ]; then
  echo "Multiple database creation requested: $POSTGRES_MULTIPLE_DATABASES"
  for db in $(echo $POSTGRES_MULTIPLE_DATABASES | tr ',' ' '); do
    create_database $db
  done
  echo "Multiple databases created"
fi
```

**Purpose**: PostgreSQL container normally creates only one database. This script reads a comma-separated list from `POSTGRES_MULTIPLE_DATABASES` environment variable and creates each database at initialization.

**Databases Created**: `development` and `test` (isolated environments).

#### Container Lifecycle

**Build Image**:
```bash
make docker-build
# Builds the custom PostgreSQL image with the init script
```

**Create Container**:
```bash
make docker-create
# Creates a container named 'echosub-psql-container'
# Environment: POSTGRES_PASSWORD=123456, POSTGRES_MULTIPLE_DATABASES=development,test
# Volume: ./docker/psql/data (persists data across container restarts)
# Port: 5432 (host) → 5432 (container)
```

**Start/Stop**:
```bash
make docker-start  # Start existing container
make docker-stop   # Stop running container
```

**Data Persistence**: Database data is stored in `docker/psql/data/` on the host filesystem. Removing the container does not delete data. To reset, delete this directory.

### Makefile Targets

The `Makefile` provides convenient commands for all development workflows.

#### Docker Commands

| Target | Command | Description |
|--------|---------|-------------|
| `docker-build` | `docker build -t echosub-psql ./docker/psql` | Build PostgreSQL image with multi-database support |
| `docker-create` | `docker create --name echosub-psql-container ...` | Create container (one-time setup) |
| `docker-start` | `docker start echosub-psql-container` | Start existing container |
| `docker-stop` | `docker stop echosub-psql-container` | Stop running container |

#### Schema and Migration Commands

| Target | Command | Description |
|--------|---------|-------------|
| `migrate-diff` | `atlas migrate diff --env local` | Generate new migration from schema.hcl changes |
| `migrate-apply` | `atlas migrate apply --env local` | Apply migrations to development database |
| `migrate-apply-test` | `atlas migrate apply --env test` | Apply migrations to test database |

**Atlas Environments** (defined in `atlas.hcl`):
- `local`: Points to `postgres://postgres:123456@localhost:5432/development?sslmode=disable`
- `test`: Points to `postgres://postgres:123456@localhost:5432/test?sslmode=disable`

#### Code Generation

| Target | Command | Description |
|--------|---------|-------------|
| `generate` | `sqlc generate` | Generate Go code from SQL queries in queries/ directory |

**sqlc Configuration** (`sqlc.yaml`): Specifies database connection, query file locations, and output directory (`generated/`).

#### Development Setup

| Target | Command | Description |
|--------|---------|-------------|
| `dev-setup` | `docker-build && docker-create && docker-start && migrate-apply` | Complete first-time setup: build Docker image, create container, start it, apply schema |

**Usage**: Run `make dev-setup` once when first cloning the repository. This sets up the entire local environment.

#### Testing

| Target | Command | Description |
|--------|---------|-------------|
| `test` | `DATABASE_URL="postgres://postgres:123456@localhost:5432/test?sslmode=disable" go test ./...` | Run all tests against the test database |

**Prerequisite**: Test database must have schema applied (`make migrate-apply-test`).

### Build Process

#### Prerequisites

Developers must install these tools before building:

| Tool | Installation | Verification |
|------|--------------|--------------|
| **Atlas CLI** | `brew install ariga/tap/atlas` (macOS) or [download binary](https://atlasgo.io/getting-started#installation) | `atlas version` |
| **sqlc CLI** | `brew install sqlc` (macOS) or [download binary](https://docs.sqlc.dev/en/latest/overview/install.html) | `sqlc version` |
| **Docker** | Install Docker Desktop | `docker --version` |
| **Go** | Install Go 1.21+ | `go version` |

#### Build Steps

1. **Start PostgreSQL** (if not running):
   ```bash
   make docker-start
   ```

2. **Apply Migrations** (ensure schema is current):
   ```bash
   make migrate-apply
   ```

3. **Generate Code**:
   ```bash
   make generate
   ```
   Produces `generated/*.go` files from `queries/*.sql`.

4. **Build Module**:
   ```bash
   go build ./...
   ```

5. **Run Tests**:
   ```bash
   make migrate-apply-test  # Ensure test database has schema
   make test
   ```

#### CI/CD Implications

Build servers (GitHub Actions, GitLab CI, etc.) must:
1. Install Atlas CLI and sqlc CLI (via download or package manager)
2. Start PostgreSQL (use Docker or hosted PostgreSQL service)
3. Run `atlas migrate apply --env test` to initialize schema
4. Run `sqlc generate` to produce generated code
5. Run `go test ./...` to execute tests
6. Optionally: Verify `git diff --exit-code generated/` to ensure generated code is up-to-date with committed queries

**Generated Code in Version Control**: The `generated/` directory is **not committed** (gitignored). CI must regenerate it during the build. This ensures queries and schema stay synchronized.

### Development Workflows

#### First-Time Setup

When a developer first clones the repository:

```bash
# 1. Clone repository
git clone github.com/prethora/echosub_db
cd echosub_db

# 2. Install prerequisites (one-time)
brew install atlas sqlc docker

# 3. Run development setup (creates Docker container, applies schema)
make dev-setup

# 4. Generate code
make generate

# 5. Verify setup by running tests
make migrate-apply-test
make test
```

**Total Time**: ~5 minutes (mostly Docker image download).

#### Schema Changes

When modifying the database schema:

```bash
# 1. Edit schema definition
vim schema.hcl  # Make changes (add table, column, index, etc.)

# 2. Generate migration
make migrate-diff
# This creates a new file in migrations/ directory
# Review the generated SQL to ensure correctness

# 3. Apply migration to development database
make migrate-apply

# 4. Regenerate Go code (sqlc needs updated schema)
make generate

# 5. Update queries if needed
vim queries/videos.sql  # Add queries for new columns/tables

# 6. Regenerate again if queries changed
make generate

# 7. Update public API in db.go to expose new functionality
vim db.go

# 8. Test changes
make migrate-apply-test
make test
```

**Migration Review**: Always inspect generated migration SQL before applying. Atlas is smart, but manual review catches unintended changes (e.g., accidental column drops).

#### Query Changes

When adding or modifying SQL queries:

```bash
# 1. Edit query files
vim queries/videos.sql  # Add new query or modify existing

# 2. Regenerate Go code
make generate

# 3. Update public API if exposing new query
vim db.go

# 4. Test
make test
```

**Query Syntax**: sqlc validates queries against the actual database schema. If a query references a non-existent column, `sqlc generate` fails immediately (fast feedback).

#### Running Tests

To run tests during development:

```bash
# Ensure test database has latest schema
make migrate-apply-test

# Run tests
make test
```

**Fast Iteration**: After making code changes in `db.go`, you can run `make test` directly (no need to regenerate or reapply migrations unless schema or queries changed).

### Code Generation Workflow (sqlc)

#### When sqlc Runs

**Build-time only**. sqlc is not a runtime dependency. It runs during development and CI/CD to generate Go code. The generated code is what actually ships in production (compiled into the binary).

#### What sqlc Consumes

1. **SQL Query Files** (`queries/*.sql`):
   ```sql
   -- name: GetVideo :one
   SELECT * FROM videos WHERE id = $1;

   -- name: CreateVideo :one
   INSERT INTO videos (...) VALUES (...) RETURNING *;
   ```

2. **Database Schema** (via live connection):
   - sqlc connects to the database specified in `sqlc.yaml`
   - Introspects table definitions to infer Go types for columns
   - Validates queries against actual schema (catches typos, missing columns)

**Configuration** (`sqlc.yaml`):
```yaml
version: "2"
sql:
  - schema: "schema.hcl"  # Optional: schema file reference
    queries: "queries"
    engine: "postgresql"
    database:
      uri: "postgres://postgres:123456@localhost:5432/development?sslmode=disable"
    gen:
      go:
        package: "generated"
        out: "generated"
        emit_json_tags: true
        emit_pointers_for_null_types: true
```

#### What sqlc Produces

**Generated Files** (in `generated/` directory):

- **`models.go`**: Go structs for each database table, matching column types exactly
- **`db.go`**: `Queries` struct with a `*pgxpool.Pool` field and initialization method
- **`*.sql.go`**: One file per SQL file in `queries/`. Contains methods for each query with typed parameters and return values.

**Example Generated Method**:
```go
// Generated from: -- name: GetVideo :one
func (q *Queries) GetVideo(ctx context.Context, id uuid.UUID) (Video, error) {
    // ... generated SQL execution code
}
```

**Type Safety**: If `schema.hcl` changes (e.g., column renamed), and queries aren't updated, `sqlc generate` fails. If queries are updated but Go code calling them isn't, `go build` fails. Compile-time safety throughout.

#### How sqlc Knows the Schema

sqlc **connects to a live database** to introspect the schema. The database URL in `sqlc.yaml` must point to a database that has had migrations applied.

**Workflow Dependency**:
```
schema.hcl → atlas migrate apply → PostgreSQL has schema → sqlc introspects → generates Go code
```

This is why `make dev-setup` applies migrations before developers run `sqlc generate`.

**Alternative**: sqlc can also parse schema files directly (without database connection), but live database introspection is more reliable and catches drift between schema files and actual database state.

## Environment Differences

The module operates across three distinct environments, each with different infrastructure and operational characteristics.

### Environment Comparison

| Aspect | Development | Test | Production |
|--------|-------------|------|------------|
| **Database Hosting** | Local Docker container | Local Docker container (separate database) | Supabase (managed PostgreSQL) |
| **Connection URL** | `postgres://postgres:123456@localhost:5432/development?sslmode=disable` | `postgres://postgres:123456@localhost:5432/test?sslmode=disable` | Supabase serverless URL (SSL required, pooling parameters included) |
| **Connection Pooling** | pgxpool defaults (local pooling only) | pgxpool defaults (local pooling only) | Supabase connection pooler + pgxpool (layered pooling) |
| **Schema Management** | `make migrate-apply` (applies to local dev DB) | `make migrate-apply-test` (applies to local test DB) | Automated deployment pipeline applies migrations |
| **Runtime Environment** | Developer workstation | CI/CD build server | AWS Lambda (Docker containers) |
| **Data Persistence** | Local filesystem (`docker/psql/data/`) | Ephemeral (truncated after tests) | Supabase storage (persistent, backed up) |

### Development Environment

**Purpose**: Local development and manual testing.

**Infrastructure**:
- PostgreSQL 16 in Docker container
- Single container hosts both `development` and `test` databases
- Port 5432 exposed to host
- Data persists in `docker/psql/data/`

**Connection**: Direct connection to localhost (no SSL).

**Workflow**: Developer makes changes → applies migrations locally → regenerates code → tests manually or with `make test`.

### Test Environment

**Purpose**: Automated integration testing in CI/CD.

**Infrastructure**:
- Same Docker container as development
- Separate `test` database (isolated from development data)
- Schema applied via `make migrate-apply-test` before tests run
- Data truncated after test runs (clean slate for each test execution)

**Connection**: Same Docker container, different database name.

**Workflow**: CI build starts → spins up PostgreSQL in Docker → applies migrations → generates code → runs tests → reports results.

### Production Environment (Supabase)

**Purpose**: Production workloads in AWS Lambda.

**Infrastructure**:
- **Supabase-hosted PostgreSQL 16**: Managed database with automatic backups, replication, and monitoring
- **Serverless Connection URL**: Supabase provides a specialized URL optimized for serverless functions:
  ```
  postgres://[project-ref].[region].pooler.supabase.com:6543/postgres
  ```
  - Port 6543: Supabase's connection pooler (not standard PostgreSQL 5432)
  - SSL required: `?sslmode=require` (Supabase enforces encrypted connections)
  - Pooling built-in: The URL points to Supabase's PgBouncer instance, which handles connection pooling across all clients

**Connection Pooling Strategy**:
- **Two-Layer Pooling**:
  1. **Supabase PgBouncer**: Serverless-optimized pooler that manages connections to actual PostgreSQL instances. Handles connection reuse across Lambda invocations from different containers.
  2. **pgxpool (in module)**: Local pool within each Lambda container. Maintains a small pool of connections to the Supabase pooler.

- **Why Both**: pgxpool reduces the overhead of creating new connections on every query. Supabase's pooler handles global connection limits (PostgreSQL max_connections).

**AWS Lambda Integration**:
- Lambda containers run Docker images containing the compiled Go binary with this module
- `DATABASE_URL` environment variable points to Supabase serverless URL
- One `DB` instance created per Lambda container (outside handler, in global scope):
  ```go
  var db *echosubdb.DB

  func init() {
      db, err = echosubdb.New(context.Background(), os.Getenv("DATABASE_URL"))
      // Handle error
  }

  func handler(ctx context.Context, event Event) (Response, error) {
      // Use db instance (reused across invocations)
      video, err := db.GetVideo(ctx, videoID)
      // ...
  }
  ```
- Container reuse: AWS Lambda reuses containers across invocations. The `DB` instance persists, avoiding repeated connection pool initialization.

**Schema Management in Production**:
- Migrations are applied via deployment pipeline (GitHub Actions, etc.)
- Process:
  1. Build deploys new migration files to migration runner
  2. Runner executes `atlas migrate apply --env production` (with production DATABASE_URL)
  3. Migrations apply before new Lambda code is deployed (ensures schema is ready)
- **No manual schema changes**: All changes go through version-controlled migrations

**Connection URL Differences**:
| Component | Local | Supabase |
|-----------|-------|----------|
| **Host** | `localhost` | `[project-ref].[region].pooler.supabase.com` |
| **Port** | `5432` (PostgreSQL default) | `6543` (Supabase pooler) |
| **Database** | `development` or `test` | `postgres` (default Supabase DB name) |
| **SSL** | `sslmode=disable` (local trust) | `sslmode=require` (enforced encryption) |
| **Pooling** | None in URL (pgxpool handles locally) | Implicit (URL points to pooler) |

## Observability and Debugging

### Logging Strategy

The module itself **does not log** by default. It follows the principle that libraries should not make logging decisions for applications.

**Rationale**:
- **No Logging Dependencies**: Avoids coupling to specific logging frameworks (zap, logrus, slog, etc.)
- **Caller Control**: Consuming applications decide what to log, at what level, and to which destination
- **Error Propagation**: All errors are returned to callers, who can log them with appropriate context

**How Callers Log**:
```go
video, err := db.GetVideo(ctx, videoID)
if err != nil {
    // Caller logs with their chosen framework
    logger.Error("failed to get video", zap.String("video_id", videoID), zap.Error(err))
    return err
}
```

**Future Extension**: If logging becomes necessary, the `DB` struct could accept an optional logger interface:
```go
type Logger interface {
    Info(msg string, fields ...any)
    Error(msg string, fields ...any)
}

func NewWithLogger(ctx context.Context, url string, logger Logger) (*DB, error)
```
This allows opt-in logging without forcing a specific framework.

### Error Context

Errors returned by the module include sufficient context for debugging.

**Error Wrapping**:
```go
// In db.go
func (db *DB) GetVideo(ctx context.Context, id string) (Video, error) {
    video, err := db.queries.GetVideo(ctx, id)
    if err != nil {
        if errors.Is(err, pgx.ErrNoRows) {
            return Video{}, ErrNotFound
        }
        // Wrap with context
        return Video{}, fmt.Errorf("GetVideo(%s): %w", id, err)
    }
    return video, nil
}
```

**Error Message Example**:
```
GetVideo(123e4567-e89b-12d3-a456-426614174000): connection refused
```

This includes:
- Operation name (`GetVideo`)
- Input parameter (video ID)
- Root cause (connection refused)

**Stack Traces**: Go's error wrapping (`%w`) preserves error chains. Tools like `fmt.Printf("%+v", err)` (with libraries like github.com/pkg/errors) can print stack traces for debugging.

### Query Tracing

pgx supports query logging and tracing via tracers.

**Built-in pgx Tracer**:
```go
import "github.com/jackc/pgx/v5/tracelog"

// In New() function, configure pool with tracer
config, _ := pgxpool.ParseConfig(databaseURL)
config.ConnConfig.Tracer = &tracelog.TraceLog{
    Logger: pgxLogger,  // Implements pgx.Logger interface
    LogLevel: tracelog.LogLevelDebug,
}
pool, _ := pgxpool.NewWithConfig(ctx, config)
```

**What Gets Logged**:
- SQL query text
- Query parameters
- Execution time
- Rows affected/returned
- Errors

**Production Consideration**: Enable query tracing only in development or when debugging specific issues. In production, it generates high log volume and may expose sensitive data in query parameters.

**Current Implementation**: The module does not enable tracing by default (keeps it simple). Callers can wrap the pool with tracing if needed, or the module could expose a configuration option.

### Production Debugging

When issues occur in production:

**1. Application Logs (Lambda CloudWatch)**:
- Lambda functions log to CloudWatch Logs
- Application code logs errors returned by the module (see Error Context above)
- Logs include trace IDs (if using AWS X-Ray or similar)

**2. Database Logs (Supabase Dashboard)**:
- Supabase provides query logs showing slow queries, errors, and connection metrics
- Access via Supabase dashboard → Database → Logs
- Useful for diagnosing query performance issues or connection pool exhaustion

**3. Error Messages**:
- Module errors include operation name and parameters (sanitized, no sensitive data)
- Stack traces available if application uses error libraries that preserve them

**4. Metrics (Future)**:
- Connection pool metrics (active connections, idle connections, wait time)
- Query latency histograms
- Error rate by operation
- Can be exposed via Prometheus metrics or CloudWatch custom metrics

**Debugging Workflow**:
```
Production error occurs
→ Check Lambda logs for error message (includes operation and parameters)
→ Check Supabase logs for corresponding query execution
→ Reproduce locally with same parameters against test database
→ Fix issue, deploy new version
```

## Context Propagation

The module uses `context.Context` throughout for lifecycle management and cancellation.

### Context Requirements

**All database operations require context**:
```go
func (db *DB) GetVideo(ctx context.Context, id string) (Video, error)
func (db *DB) CreateVideo(ctx context.Context, video Video) (Video, error)
// ... all methods
```

**Why Context Everywhere**:
- **Cancellation**: Allows callers to cancel long-running queries
- **Timeouts**: Callers can enforce query deadlines via `context.WithTimeout`
- **Trace Propagation**: Context can carry trace IDs for distributed tracing (AWS X-Ray, OpenTelemetry)
- **Request Scoping**: In web servers, context ties queries to the originating HTTP request

### Context Cancellation Behavior

When context is cancelled (timeout or explicit cancellation), pgx aborts the query:

**What Happens**:
1. pgx sends a cancellation request to PostgreSQL
2. PostgreSQL terminates the query execution
3. pgx returns `context.Canceled` or `context.DeadlineExceeded` error
4. Connection remains valid (can be reused for subsequent queries)

**Example**:
```go
ctx, cancel := context.WithTimeout(context.Background(), 2*time.Second)
defer cancel()

video, err := db.GetVideo(ctx, videoID)
if errors.Is(err, context.DeadlineExceeded) {
    // Query took longer than 2 seconds
    logger.Warn("query timeout", zap.String("video_id", videoID))
}
```

**Transaction Cancellation**: If context is cancelled during a transaction, PostgreSQL rolls back the transaction automatically. The module currently doesn't expose transactions, but when it does, this behavior will be critical.

### Timeout Handling

**Caller-Controlled Timeouts**:
```go
// Per-query timeout
ctx, cancel := context.WithTimeout(parentCtx, 500*time.Millisecond)
defer cancel()
video, err := db.GetVideo(ctx, videoID)
```

**Lambda Request Timeout**:
```go
func handler(ctx context.Context, event Event) (Response, error) {
    // ctx has deadline = Lambda timeout (e.g., 15 seconds)
    // All db operations inherit this deadline
    video, err := db.GetVideo(ctx, event.VideoID)
    // If Lambda is about to timeout, ctx is cancelled, query aborts
}
```

**Default Timeout**: pgxpool has a default connection timeout (30 seconds) and query timeout (none by default). The module relies on caller-provided context for query-level timeouts.

### Trace IDs and Distributed Tracing

Context can carry trace IDs for correlating logs across services:

**AWS X-Ray**:
```go
import "github.com/aws/aws-xray-sdk-go/xray"

func handler(ctx context.Context, event Event) (Response, error) {
    // X-Ray injects trace ID into ctx
    _, seg := xray.BeginSubsegment(ctx, "GetVideo")
    defer seg.Close(nil)

    video, err := db.GetVideo(ctx, videoID)
    // pgx queries automatically include X-Ray trace context
}
```

**OpenTelemetry**:
Similar integration possible with OpenTelemetry Go SDK and pgx OpenTelemetry tracer.

**Trace Propagation**: pgx can be configured with tracers that extract trace IDs from context and include them in query logs or send them to tracing backends (Jaeger, Honeycomb, etc.).

## Success Criteria

The following criteria define when the module is considered complete and production-ready.

### 1. Schema Management

**Atlas schema applies cleanly**:
- Running `atlas migrate apply` against a fresh PostgreSQL 16 database succeeds without errors
- All tables, indexes, constraints, and functions are created as specified
- Schema matches the declarative definition in `schema.hcl`

**Migration generation works correctly**:
- `atlas migrate diff` generates valid SQL migrations from `schema.hcl` changes
- Generated migrations can be applied successfully
- Migration SQL is reviewable and matches intended schema changes

**Idempotency**:
- Running `atlas migrate apply` multiple times on the same database has no effect (no duplicate migrations applied)
- Atlas correctly tracks applied migrations in `atlas_schema_revisions` table

### 2. Code Generation

**sqlc generates type-safe code**:
- `sqlc generate` produces Go code in `generated/` directory without errors
- Generated code compiles with `go build ./...`
- All queries in `queries/*.sql` are translated to Go methods

**Type accuracy**:
- Generated Go types match database column types exactly:
  - UUID columns → `uuid.UUID` or `string`
  - JSONB columns → `[]Segment` (with proper marshaling)
  - Nullable columns → pointer types (`*string`, `*int32`, `*time.Time`)
  - BIGINT columns → `int64`
- No type mismatches that would cause runtime errors

**Schema-Query Synchronization**:
- If schema changes and queries are not updated, `sqlc generate` fails (compile-time safety)
- If queries reference non-existent columns, `sqlc generate` reports errors

### 3. Connection Handling

**Successful connection**:
- `New(ctx, databaseURL)` succeeds when connecting to:
  - Local Docker PostgreSQL (`postgres://postgres:123456@localhost:5432/development?sslmode=disable`)
  - Supabase production database (serverless URL with SSL)
- Initial connectivity check passes (database is reachable)

**Connection pooling**:
- pgxpool creates a connection pool with appropriate defaults
- Pool can handle concurrent queries from multiple goroutines
- Pool stats show healthy connection reuse (low rate of new connection creation)

**Failure handling**:
- Invalid `DATABASE_URL` returns `ErrConnection` immediately
- Unreachable database returns `ErrConnection` (fails fast, no hanging)

### 4. CRUD Operations

**All public API methods execute correctly**:
- **Videos**: `CreateVideo`, `GetVideo`, `GetVideoByYouTubeID`, `FindVideosByFingerprint`
- **Transcriptions**: `CreateTranscription`, `GetTranscription`, `GetTranscriptionForVideo`, `TranscriptionExistsForVideo`
- **Translations**: `CreateTranslation`, `GetTranslation`, `GetTranslationForVideo`, `ListTranslationsForVideo`, `TranslationExistsForVideo`
- **Jobs**: `CreateJob`, `GetJob`, `UpdateJobStatus`, `SetJobVideoID`, `ListJobsByStatus`

**Data integrity**:
- Foreign key constraints prevent orphaned records (can't create translation without video)
- Unique constraints prevent duplicate records (duplicate `(video_id, language)` for translations fails)
- CASCADE deletes work: deleting a video deletes its transcriptions and translations
- SET NULL works: deleting a video sets `jobs.video_id` to NULL (job record remains)

**JSONB handling**:
- Segments are correctly marshaled to JSONB on INSERT
- Segments are correctly unmarshaled from JSONB on SELECT
- Segment arrays with 100+ segments are handled correctly

### 5. Fingerprint Matching

**Correct matching**:
- `FindVideosByFingerprint` returns videos with average Hamming distance ≤ threshold
- Exact matches (distance = 0) are always returned
- Near matches (distance ≤ 6) are returned when threshold allows
- No false negatives (videos within threshold are not missed)

**Performance**:
- Duration filtering reduces candidate set before expensive Hamming distance calculations
- Query completes in < 100ms for databases with up to 100,000 videos (typical EchoSub scale)
- Index on `duration_ms` is used (verify with `EXPLAIN ANALYZE`)

**Edge cases**:
- Zero matching videos returns empty array (no error)
- Multiple matches are sorted by distance (best match first)
- Videos with duration outside ±100ms tolerance are excluded

### 6. Error Handling

**Sentinel errors**:
- `ErrNotFound` returned when querying non-existent records (e.g., `GetVideo` with invalid UUID)
- `ErrDuplicateKey` returned on unique constraint violations (e.g., duplicate YouTube ID)
- `ErrConnection` returned on connection failures (invalid URL, database unreachable)

**Error context**:
- Errors include operation name (e.g., `GetVideo(...)`)
- Errors include input parameters (sanitized, no sensitive data)
- Errors wrap underlying causes (pgx errors preserved with `%w`)

**Error propagation**:
- Application code can use `errors.Is(err, echosubdb.ErrNotFound)` for error type checking
- Stack traces are preserved for debugging (via error wrapping)

### 7. Testing

**Comprehensive coverage**:
- All public API methods have at least one integration test
- Happy path tests (create, read, update operations succeed)
- Error case tests (not found, duplicate key, connection errors)
- Edge case tests (null fields, empty segments, fingerprint matching with no results)

**Test reliability**:
- Test suite passes consistently (no flakiness)
- Tests can run locally via `make test`
- Tests can run in CI/CD (GitHub Actions, GitLab CI) without manual intervention

**CI/CD integration**:
- Build pipeline installs Atlas and sqlc
- Pipeline applies migrations to test database
- Pipeline runs `sqlc generate` and `go test ./...`
- All tests pass before allowing deployment

### 8. Development Experience

**First-time setup**:
- Developer can run `make dev-setup` and have a working environment in < 10 minutes
- Setup includes: Docker build, container creation, schema application
- No manual steps required beyond installing prerequisite tools (Docker, Atlas, sqlc, Go)

**Schema change workflow**:
- Developer modifies `schema.hcl`
- `make migrate-diff` generates migration
- Developer reviews migration SQL
- `make migrate-apply` applies migration to dev database
- `make generate` regenerates Go code
- Workflow completes in < 1 minute

**Documentation**:
- README provides clear setup instructions for new developers
- README documents common workflows (schema changes, query changes, running tests)
- README links to Atlas and sqlc documentation for deeper learning

**Error messages**:
- Clear error messages when prerequisites are missing (e.g., "atlas not found, install with...")
- Clear error messages when steps are run out of order (e.g., "apply migrations before generating code")

### Acceptance Checklist

The module is ready for production when:

- [ ] All success criteria above are met
- [ ] Integration test suite passes locally and in CI
- [ ] Schema applies cleanly to fresh PostgreSQL 16 database
- [ ] Code generation produces compilable, type-safe Go code
- [ ] All CRUD operations work against both local and Supabase databases
- [ ] Fingerprint matching returns accurate results within performance targets
- [ ] Error handling provides actionable error messages
- [ ] Documentation (README) is complete and accurate
- [ ] At least one EchoSub service (Lambda function) uses the module successfully in staging environment
- [ ] No outstanding critical bugs or security issues

This checklist gates the transition from development to production deployment.
