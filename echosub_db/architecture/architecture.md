# Architecture: EchoSub Database Module

## 1. Package Overview

```
github.com/prethora/echosub_db/
├── schema.hcl                # Atlas schema definition (committed)
├── atlas.hcl                 # Atlas configuration (committed)
├── migrations/               # Atlas-generated migrations (committed)
│   └── *.sql
├── queries/                  # SQL queries for sqlc (committed)
│   ├── videos.sql
│   ├── transcriptions.sql
│   ├── translations.sql
│   └── jobs.sql
├── sqlc.yaml                 # sqlc configuration (committed)
├── db.go                     # Public API (committed)
├── types.go                  # Type definitions (committed)
├── errors.go                 # Sentinel errors (committed)
├── generated/                # sqlc output (NOT committed, regenerated)
│   ├── models.go
│   ├── db.go
│   └── *.sql.go
├── docker/                   # Local dev environment (committed)
│   └── psql/
│       ├── Dockerfile
│       └── init-multiple-databases.sh
├── Makefile                  # Development commands (committed)
└── README.md                 # Documentation (committed)
```

**Module Path**: `github.com/prethora/echosub_db`

**Package Name**: `echosubdb` (all exported types and functions live in the root package)

**Generated Code**: The `generated/` directory contains sqlc-generated code and is excluded from version control. It must be regenerated via `sqlc generate` during development and CI/CD builds.

**Go Version**: 1.21+ (required for pgx v5 compatibility)

**PostgreSQL Version**: 16 (required for `bit_count()` function used in fingerprint matching)

## 2. Sentinel Errors

### 2.1 Error Definitions (`errors.go`)

```go
// Package echosubdb provides type-safe database operations for the EchoSub
// subtitle platform. It manages video fingerprints, transcriptions, translations,
// and job state using PostgreSQL.
package echosubdb

import "errors"

// Sentinel errors for database operations.
// Use errors.Is() to check for specific error conditions.
var (
	// ErrNotFound is returned when a database query finds no matching records.
	// This occurs when querying by ID, YouTube ID, or other unique identifiers
	// that don't exist in the database.
	ErrNotFound = errors.New("echosubdb: not found")

	// ErrDuplicateKey is returned when an insert or update violates a unique
	// constraint. This typically occurs when attempting to create a duplicate
	// video fingerprint, duplicate translation for the same language, or
	// duplicate YouTube ID.
	ErrDuplicateKey = errors.New("echosubdb: duplicate key")

	// ErrConnection is returned when the database connection fails.
	// This can occur during initialization (invalid DATABASE_URL, unreachable
	// database) or during query execution (network partition, database restart).
	ErrConnection = errors.New("echosubdb: connection failed")
)
```

## 3. Basic Types

### 3.1 Segment Type (`types.go`)

```go
package echosubdb

// Segment represents a single subtitle segment with timing and text.
// Segments are stored in JSONB columns in the database and automatically
// marshaled/unmarshaled by sqlc.
type Segment struct {
	// Start is the segment start time in seconds.
	Start float64 `json:"start"`

	// End is the segment end time in seconds.
	End float64 `json:"end"`

	// Text is the subtitle text for this segment.
	Text string `json:"text"`
}
```

### 3.2 Job Status Type (`types.go`)

```go
// JobStatus represents the current state of a processing job.
type JobStatus string

const (
	// JobStatusPending indicates the job has been created but not started.
	JobStatusPending JobStatus = "pending"

	// JobStatusUploading indicates the audio file is being uploaded to storage.
	JobStatusUploading JobStatus = "uploading"

	// JobStatusTranscribing indicates Whisper is generating the transcription.
	JobStatusTranscribing JobStatus = "transcribing"

	// JobStatusTranslating indicates the translation service is processing.
	JobStatusTranslating JobStatus = "translating"

	// JobStatusSplitting indicates subtitle segments are being split/formatted.
	JobStatusSplitting JobStatus = "splitting"

	// JobStatusComplete indicates the job finished successfully.
	JobStatusComplete JobStatus = "complete"

	// JobStatusFailed indicates the job encountered an error and terminated.
	JobStatusFailed JobStatus = "failed"
)
```

### 3.3 Fingerprint Type (`types.go`)

```go
// Fingerprint contains the perceptual hash data used to identify videos.
// A fingerprint consists of the video duration and 5 perceptual hashes
// sampled at different positions (10%, 25%, 50%, 75%, 90%) in the video.
type Fingerprint struct {
	// DurationMs is the video duration in milliseconds.
	// Used for initial filtering (±100ms tolerance).
	DurationMs int32

	// Hash10 is the perceptual hash at the 10% position.
	Hash10 int64

	// Hash25 is the perceptual hash at the 25% position.
	Hash25 int64

	// Hash50 is the perceptual hash at the 50% position.
	Hash50 int64

	// Hash75 is the perceptual hash at the 75% position.
	Hash75 int64

	// Hash90 is the perceptual hash at the 90% position.
	Hash90 int64
}
```

### 3.4 Data Model Types (`types.go`)

The following types represent database records and match the PostgreSQL table schemas. Nullable database columns are represented as pointer types in Go (`*string`, `*int32`, `*time.Time`).

```go
import "time"

// Video represents a video identified by its perceptual fingerprint.
// Videos are uniquely identified by their visual characteristics (duration + hashes).
type Video struct {
	// ID is the primary key (UUID stored as string).
	ID string

	// DurationMs is the video duration in milliseconds.
	DurationMs int32

	// Hash10 is the perceptual hash at the 10% position.
	Hash10 int64

	// Hash25 is the perceptual hash at the 25% position.
	Hash25 int64

	// Hash50 is the perceptual hash at the 50% position.
	Hash50 int64

	// Hash75 is the perceptual hash at the 75% position.
	Hash75 int64

	// Hash90 is the perceptual hash at the 90% position.
	Hash90 int64

	// YoutubeID is an optional YouTube video ID for fast-path lookups.
	// Nullable field represented as pointer (nil = not set).
	YoutubeID *string

	// Title is an optional video title for display purposes.
	// Nullable field represented as pointer (nil = not set).
	Title *string

	// CreatedAt is the timestamp when this video record was created.
	CreatedAt time.Time
}

// Transcription represents a source language transcription from Whisper.
// Each video has at most one transcription (unique constraint on video_id).
type Transcription struct {
	// ID is the primary key (UUID stored as string).
	ID string

	// VideoID is the foreign key to the videos table.
	VideoID string

	// Language is the ISO 639-1 language code (e.g., "en", "ko").
	Language string

	// Segments is the array of subtitle segments with timing and text.
	// Stored as JSONB in the database, automatically marshaled/unmarshaled by sqlc.
	Segments []Segment

	// SegmentCount is the number of segments (denormalized for convenience).
	SegmentCount int32

	// CreatedAt is the timestamp when this transcription was created.
	CreatedAt time.Time
}

// Translation represents translated subtitles in a target language.
// Each (video, language) pair has at most one translation (unique constraint).
type Translation struct {
	// ID is the primary key (UUID stored as string).
	ID string

	// VideoID is the foreign key to the videos table.
	VideoID string

	// TranscriptionID is the foreign key to the transcriptions table.
	// Translations are derived from transcriptions.
	TranscriptionID string

	// Language is the target language ISO 639-1 code (e.g., "es", "ja").
	Language string

	// Segments is the array of translated subtitle segments.
	// Stored as JSONB in the database, automatically marshaled/unmarshaled by sqlc.
	Segments []Segment

	// SegmentCount is the number of segments (denormalized for convenience).
	SegmentCount int32

	// CreatedAt is the timestamp when this translation was created.
	CreatedAt time.Time
}

// Job represents a processing job in the EchoSub pipeline.
// Jobs track the state of video processing from upload through completion.
type Job struct {
	// ID is the primary key (UUID stored as string).
	ID string

	// VideoID is the foreign key to the videos table (nullable until fingerprint is created).
	// Nullable field represented as pointer (nil = video not yet identified).
	// Uses ON DELETE SET NULL (job record persists for history even if video deleted).
	VideoID *string

	// TargetLanguage is the requested translation language (ISO 639-1 code).
	TargetLanguage string

	// Status is the current processing state (pending, uploading, transcribing, etc.).
	Status JobStatus

	// ErrorMessage contains error details if Status is JobStatusFailed.
	// Nullable field represented as pointer (nil = no error).
	ErrorMessage *string

	// AudioKey is the storage key for the uploaded audio file.
	// Nullable field represented as pointer (nil = not yet uploaded).
	AudioKey *string

	// AudioDurationMs is the duration of the uploaded audio in milliseconds.
	// Nullable field represented as pointer (nil = not yet determined).
	AudioDurationMs *int32

	// CreatedAt is the timestamp when this job was created.
	CreatedAt time.Time

	// StartedAt is the timestamp when processing started.
	// Nullable field represented as pointer (nil = not yet started).
	StartedAt *time.Time

	// CompletedAt is the timestamp when processing completed (success or failure).
	// Nullable field represented as pointer (nil = not yet completed).
	CompletedAt *time.Time
}

// VideoMatch represents a video matched by fingerprint with its similarity score.
// Returned by FindVideosByFingerprint to rank matches by quality.
type VideoMatch struct {
	// Video is the matched video record.
	Video Video

	// AvgDistance is the average Hamming distance across all 5 hash positions.
	// Lower values indicate better matches (0 = exact match, typical threshold ≤ 6).
	AvgDistance float64
}
```

**Foreign Key Relationships**:
- `Transcription.VideoID` → `Video.ID` (CASCADE delete)
- `Translation.VideoID` → `Video.ID` (CASCADE delete)
- `Translation.TranscriptionID` → `Transcription.ID` (CASCADE delete)
- `Job.VideoID` → `Video.ID` (SET NULL on delete, preserves job history)

**JSONB Marshaling**:
- `Transcription.Segments` and `Translation.Segments` are stored as JSONB in PostgreSQL
- sqlc automatically handles marshaling/unmarshaling using the `json` struct tags on the `Segment` type
- No custom marshal/unmarshal code required

## 4. Import Requirements

**Standard Library Imports**:
- `errors` — For sentinel error definitions
- `time` — For timestamp fields (time.Time)
- `context` — For context propagation, cancellation, and timeouts

**External Dependencies**:
- `github.com/jackc/pgx/v5/pgxpool` — PostgreSQL connection pooling
- `github.com/google/uuid` — UUID generation (application-side)

## 5. Public API

### 5.1 DB Struct (`db.go`)

```go
package echosubdb

import (
	"context"
	"github.com/jackc/pgx/v5/pgxpool"
)

// DB provides type-safe database operations for the EchoSub platform.
// It wraps a PostgreSQL connection pool and sqlc-generated query methods.
//
// Thread Safety: DB is safe for concurrent use across multiple goroutines.
// The underlying connection pool handles concurrent query execution safely.
//
// Lifecycle: Create one DB instance per application (or Lambda container),
// reuse it across requests, and call Close() on shutdown.
type DB struct {
	// pool is the PostgreSQL connection pool (pgxpool.Pool).
	pool *pgxpool.Pool

	// queries would reference sqlc-generated Queries struct containing
	// all query methods. Not shown here as it's generated code.
	// Example: queries *generated.Queries
}
```

### 5.2 Constructor and Lifecycle (`db.go`)

```go
// New creates a new DB instance from the provided database URL.
//
// It performs the following initialization steps:
// 1. Parses the database URL (validates format)
// 2. Creates a connection pool with pgxpool defaults
// 3. Performs an initial connectivity check (fails fast if unreachable)
// 4. Wraps the pool in a DB instance with sqlc query methods
//
// Database URL Formats:
//   - Local/Development: postgres://postgres:123456@localhost:5432/development?sslmode=disable
//   - Local/Test: postgres://postgres:123456@localhost:5432/test?sslmode=disable
//   - Production (Supabase): postgres://[project-ref].[region].pooler.supabase.com:6543/postgres?sslmode=require
//
// Connection Pooling:
//   - Uses pgxpool defaults (determined by system resources)
//   - In production, works in concert with Supabase's external connection pooler
//
// Error Conditions:
//   - Returns ErrConnection if the database URL is malformed
//   - Returns ErrConnection if the database is unreachable
//   - Returns ErrConnection if authentication fails
//
// The caller is responsible for calling Close() when done with the DB instance.
func New(ctx context.Context, databaseURL string) (*DB, error)

// Close gracefully shuts down the database connection pool.
//
// Behavior:
//   - Waits for in-flight queries to complete (respects context deadlines)
//   - Closes idle connections immediately
//   - Releases all resources held by the pool
//
// Safety:
//   - Close is idempotent (safe to call multiple times)
//   - Calling Close while queries are running is safe (those queries complete)
//   - After Close, any new query attempts return errors
//
// Lambda Context: In AWS Lambda, explicit Close() calls are often unnecessary
// (container shuts down when evicted), but including defer db.Close() is good
// practice for test code and non-Lambda usage.
func (db *DB) Close()
```

### 5.3 Video Operations (`db.go`)

```go
// CreateVideo inserts a new video record into the database.
//
// The caller must generate the video ID (UUID) before calling this method.
// All hash fields (Hash10-Hash90) and DurationMs must be provided.
// YoutubeID and Title are optional (use nil for not set).
//
// Returns:
//   - The created Video with server-generated CreatedAt timestamp
//   - ErrDuplicateKey if a video with the same YouTube ID already exists
//   - ErrConnection on database connectivity issues
func (db *DB) CreateVideo(ctx context.Context, v Video) (Video, error)

// GetVideo retrieves a video by its unique ID.
//
// Parameters:
//   - id: The video UUID as a string
//
// Returns:
//   - The Video record if found
//   - ErrNotFound if no video with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) GetVideo(ctx context.Context, id string) (Video, error)

// GetVideoByYouTubeID retrieves a video by its YouTube ID (fast path).
//
// This method provides a fast lookup path for videos with known YouTube IDs,
// avoiding the more expensive fingerprint matching algorithm.
//
// Parameters:
//   - youtubeID: The YouTube video ID (e.g., "dQw4w9WgXcQ")
//
// Returns:
//   - The Video record if found
//   - ErrNotFound if no video with the given YouTube ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) GetVideoByYouTubeID(ctx context.Context, youtubeID string) (Video, error)

// FindVideosByFingerprint finds videos matching the given perceptual fingerprint.
//
// Algorithm:
//   1. Filter by duration (candidates within ±100ms of fp.DurationMs)
//   2. Calculate Hamming distance for each of 5 hash positions (10%, 25%, 50%, 75%, 90%)
//   3. Compute average distance across all 5 positions
//   4. Return videos where average distance ≤ maxAvgDistance
//   5. Sort results by average distance (best matches first)
//
// Parameters:
//   - fp: The fingerprint to match against (duration + 5 perceptual hashes)
//   - maxAvgDistance: Maximum average Hamming distance threshold (typical: 6.0)
//
// Returns:
//   - Slice of VideoMatch records sorted by similarity (empty if no matches)
//   - ErrConnection on database connectivity issues
//
// Note: An empty result slice is not an error (no matches found is a valid outcome).
func (db *DB) FindVideosByFingerprint(ctx context.Context, fp Fingerprint, maxAvgDistance float64) ([]VideoMatch, error)
```

### 5.4 Transcription Operations (`db.go`)

```go
// CreateTranscription inserts a new transcription record.
//
// The caller must generate the transcription ID (UUID) before calling this method.
// Each video can have at most one transcription (enforced by unique constraint on video_id).
//
// Returns:
//   - The created Transcription with server-generated CreatedAt timestamp
//   - ErrDuplicateKey if a transcription for this video already exists
//   - ErrConnection on database connectivity issues
func (db *DB) CreateTranscription(ctx context.Context, t Transcription) (Transcription, error)

// GetTranscription retrieves a transcription by its unique ID.
//
// Parameters:
//   - id: The transcription UUID as a string
//
// Returns:
//   - The Transcription record if found
//   - ErrNotFound if no transcription with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) GetTranscription(ctx context.Context, id string) (Transcription, error)

// GetTranscriptionForVideo retrieves the transcription for a specific video.
//
// Since each video has at most one transcription, this returns the unique
// transcription record for the video.
//
// Parameters:
//   - videoID: The video UUID as a string
//
// Returns:
//   - The Transcription record if found
//   - ErrNotFound if no transcription exists for the video
//   - ErrConnection on database connectivity issues
func (db *DB) GetTranscriptionForVideo(ctx context.Context, videoID string) (Transcription, error)

// TranscriptionExistsForVideo checks whether a transcription exists for a video.
//
// This is an existence check, not a retrieval operation. Use this when you only
// need to know if a transcription exists, not the transcription data itself.
//
// Parameters:
//   - videoID: The video UUID as a string
//
// Returns:
//   - true if a transcription exists for the video
//   - false if no transcription exists for the video
//   - Does NOT return ErrNotFound (existence check returns boolean, not error)
//   - ErrConnection on database connectivity issues
func (db *DB) TranscriptionExistsForVideo(ctx context.Context, videoID string) (bool, error)
```

### 5.5 Translation Operations (`db.go`)

```go
// CreateTranslation inserts a new translation record.
//
// The caller must generate the translation ID (UUID) before calling this method.
// Each (video, language) pair can have at most one translation (enforced by unique constraint).
//
// Returns:
//   - The created Translation with server-generated CreatedAt timestamp
//   - ErrDuplicateKey if a translation for this (video, language) pair already exists
//   - ErrConnection on database connectivity issues
func (db *DB) CreateTranslation(ctx context.Context, t Translation) (Translation, error)

// GetTranslation retrieves a translation by its unique ID.
//
// Parameters:
//   - id: The translation UUID as a string
//
// Returns:
//   - The Translation record if found
//   - ErrNotFound if no translation with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) GetTranslation(ctx context.Context, id string) (Translation, error)

// GetTranslationForVideo retrieves a specific translation for a video and language.
//
// Parameters:
//   - videoID: The video UUID as a string
//   - language: The target language ISO 639-1 code (e.g., "es", "ja")
//
// Returns:
//   - The Translation record if found
//   - ErrNotFound if no translation exists for the (video, language) pair
//   - ErrConnection on database connectivity issues
func (db *DB) GetTranslationForVideo(ctx context.Context, videoID string, language string) (Translation, error)

// ListTranslationsForVideo retrieves all translations for a video.
//
// A video can have multiple translations (one per target language).
// This method returns all of them.
//
// Parameters:
//   - videoID: The video UUID as a string
//
// Returns:
//   - Slice of Translation records (empty if no translations exist)
//   - ErrConnection on database connectivity issues
//
// Note: An empty result slice is not an error (no translations is a valid state).
func (db *DB) ListTranslationsForVideo(ctx context.Context, videoID string) ([]Translation, error)

// TranslationExistsForVideo checks whether a translation exists for a (video, language) pair.
//
// This is an existence check, not a retrieval operation. Use this when you only
// need to know if a translation exists, not the translation data itself.
//
// Parameters:
//   - videoID: The video UUID as a string
//   - language: The target language ISO 639-1 code
//
// Returns:
//   - true if a translation exists for the (video, language) pair
//   - false if no translation exists
//   - Does NOT return ErrNotFound (existence check returns boolean, not error)
//   - ErrConnection on database connectivity issues
func (db *DB) TranslationExistsForVideo(ctx context.Context, videoID string, language string) (bool, error)
```

### 5.6 Job Operations (`db.go`)

```go
// CreateJob inserts a new job record.
//
// The caller must generate the job ID (UUID) before calling this method.
// Initial jobs typically have Status = JobStatusPending and VideoID = nil
// (video is associated later after fingerprint matching).
//
// Returns:
//   - The created Job with server-generated CreatedAt timestamp
//   - ErrConnection on database connectivity issues
func (db *DB) CreateJob(ctx context.Context, j Job) (Job, error)

// GetJob retrieves a job by its unique ID.
//
// Parameters:
//   - id: The job UUID as a string
//
// Returns:
//   - The Job record if found
//   - ErrNotFound if no job with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) GetJob(ctx context.Context, id string) (Job, error)

// UpdateJobStatus updates the status and optional error message of a job.
//
// This method is used to transition jobs through the processing pipeline states.
// When transitioning to JobStatusFailed, provide an errorMessage. For other
// status transitions, pass nil for errorMessage.
//
// Parameters:
//   - id: The job UUID as a string
//   - status: The new status (e.g., JobStatusTranscribing, JobStatusComplete)
//   - errorMessage: Error details if status is JobStatusFailed, nil otherwise
//
// Returns:
//   - ErrNotFound if no job with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) UpdateJobStatus(ctx context.Context, id string, status JobStatus, errorMessage *string) error

// SetJobVideoID associates a job with an identified video.
//
// This is called after fingerprint matching successfully identifies a video
// for the uploaded audio. It links the job to the video record.
//
// Parameters:
//   - id: The job UUID as a string
//   - videoID: The video UUID as a string
//
// Returns:
//   - ErrNotFound if no job with the given ID exists
//   - ErrConnection on database connectivity issues
func (db *DB) SetJobVideoID(ctx context.Context, id string, videoID string) error

// ListJobsByStatus retrieves all jobs with a given status.
//
// This is used by worker processes to find jobs that need processing
// (e.g., all jobs with Status = JobStatusPending).
//
// Parameters:
//   - status: The job status to filter by
//
// Returns:
//   - Slice of Job records matching the status (empty if none match)
//   - ErrConnection on database connectivity issues
//
// Note: An empty result slice is not an error (no jobs in this status is valid).
func (db *DB) ListJobsByStatus(ctx context.Context, status JobStatus) ([]Job, error)
```

## 6. Toolchain Integration

The EchoSub database module uses three specialized tools that work together to provide type-safe database operations:

### 6.1 Atlas (Schema Management)

**Purpose**: Declarative schema definition and migration generation

**Workflow**:
1. Schema defined in `schema.hcl` using HCL (HashiCorp Configuration Language)
2. Developers modify `schema.hcl` to define or change tables, indexes, constraints, functions
3. `atlas migrate diff` compares schema against current database state
4. Atlas generates migration SQL files in `migrations/` directory (versioned, committed)
5. `atlas migrate apply` executes migrations against target environment

**Configuration**: `atlas.hcl` defines multiple environments (local, test, production) with different DATABASE_URLs

**Benefits**: Declarative schemas eliminate drift, migrations are deterministic and reviewable

### 6.2 sqlc (Query Code Generation)

**Purpose**: Type-safe Go code generation from SQL queries

**Workflow**:
1. Developers write SQL queries in `queries/*.sql` files (organized by domain)
2. Queries include special annotations (e.g., `-- name: GetVideo :one`)
3. `sqlc generate` connects to database, introspects schema
4. sqlc produces Go code in `generated/` directory:
   - `models.go`: Structs matching database tables
   - `db.go`: Queries struct with connection pool
   - `*.sql.go`: Methods for each query with typed parameters and return values

**Configuration**: `sqlc.yaml` specifies database connection, query directories, output settings

**Benefits**: Compile-time type safety, no runtime reflection, hand-written SQL with generated Go bindings

### 6.3 pgx (PostgreSQL Driver)

**Purpose**: PostgreSQL connectivity and connection pooling

**Runtime Integration**:
- The `DB` struct wraps a `*pgxpool.Pool` instance
- Pool is initialized from `DATABASE_URL` environment variable
- All queries execute through the pool
- Pool handles concurrent access, connection reuse, health checks

**Benefits**: High-performance native PostgreSQL driver, excellent type support including JSONB

### 6.4 Integration Flow

```
Design Time:  schema.hcl  +  queries/*.sql
                  ↓              ↓
Build Time:   atlas migrate   sqlc generate
              apply (schema)   (Go code)
                  ↓              ↓
Runtime:      PostgreSQL  ←  pgxpool  ←  DB struct (public API)
```

**Development Workflow**:
```bash
# 1. Define schema
vim schema.hcl

# 2. Generate migration
make migrate-diff

# 3. Apply to development database
make migrate-apply

# 4. Generate Go code (sqlc introspects schema)
make generate

# 5. Build and test
go build ./...
make test
```

## 7. Development Setup

### 7.1 Docker PostgreSQL Configuration

The module includes Docker configuration for local PostgreSQL 16 with multi-database support.

**Files**:
- `docker/psql/Dockerfile`: Extends `postgres:16` image with initialization script
- `docker/psql/init-multiple-databases.sh`: Creates `development` and `test` databases on startup

**Container Lifecycle**:
```bash
make docker-build   # Build custom PostgreSQL image
make docker-create  # Create container (one-time setup)
make docker-start   # Start existing container
make docker-stop    # Stop running container
```

**Data Persistence**: Database data stored in `docker/psql/data/` on host filesystem (survives container restarts)

### 7.2 Makefile Targets

| Target | Command | Description |
|--------|---------|-------------|
| `docker-build` | Build PostgreSQL image | Creates echosub-psql image with multi-database support |
| `docker-create` | Create container | One-time setup with POSTGRES_MULTIPLE_DATABASES=development,test |
| `docker-start` | Start container | Starts existing echosub-psql-container |
| `docker-stop` | Stop container | Stops running container |
| `migrate-diff` | `atlas migrate diff --env local` | Generate new migration from schema.hcl changes |
| `migrate-apply` | `atlas migrate apply --env local` | Apply migrations to development database |
| `migrate-apply-test` | `atlas migrate apply --env test` | Apply migrations to test database |
| `generate` | `sqlc generate` | Generate Go code from SQL queries |
| `dev-setup` | Full first-time setup | Builds Docker, creates container, starts it, applies schema |
| `test` | `go test ./...` | Run tests against test database (sets DATABASE_URL) |

### 7.3 First-Time Setup

When first cloning the repository:

```bash
# 1. Install prerequisites (one-time)
brew install atlas sqlc docker

# 2. Run development setup
make dev-setup
# This: builds Docker image → creates container → starts it → applies schema

# 3. Generate code
make generate

# 4. Verify setup
make migrate-apply-test
make test
```

**Total Time**: ~5 minutes (mostly Docker image download)

### 7.4 Schema Change Workflow

When modifying the database schema:

```bash
# 1. Edit schema definition
vim schema.hcl

# 2. Generate migration (review generated SQL before proceeding)
make migrate-diff

# 3. Apply to development database
make migrate-apply

# 4. Regenerate Go code (sqlc needs updated schema)
make generate

# 5. Update queries if needed
vim queries/videos.sql

# 6. Regenerate again if queries changed
make generate

# 7. Update public API in db.go to expose new functionality
vim db.go

# 8. Test changes
make migrate-apply-test
make test
```

**Important**: Always review generated migration SQL before applying to catch unintended changes (e.g., accidental column drops).

## 8. Implementation Details

### 8.1 Error Mapping

The public API translates pgx-specific errors to module sentinel errors for a stable API:

**Mapping Rules**:
- `pgx.ErrNoRows` → `ErrNotFound`
- `pgconn.PgError` with code `23505` (unique violation) → `ErrDuplicateKey`
- `pgconn.PgError` with code `23503` (foreign key violation) → `ErrDuplicateKey`
- Connection errors (auth failure, host unreachable) → `ErrConnection`
- Other errors → wrapped with context using `fmt.Errorf("operation(%s): %w", param, err)`

**Error Context**: Errors include operation name and input parameters (sanitized) for debugging:
```
GetVideo(123e4567-e89b-12d3-a456-426614174000): connection refused
```

**Benefits**: Consumers don't need to import pgx or understand PostgreSQL error codes. Sentinel errors provide a stable API that won't break if the underlying driver changes.

### 8.2 UUID Generation

**Strategy**: UUIDs are generated **application-side** using `github.com/google/uuid`, not by the database.

**Workflow**:
1. Caller generates UUID using `uuid.New()` before calling Create methods
2. UUID passed as a string field in the struct (e.g., `Video.ID`)
3. Database columns are `UUID` type but have no `DEFAULT` value
4. sqlc-generated code expects UUID strings as parameters

**Benefits**:
- **Consistency**: Record IDs are known immediately after creation (useful for logging, returning to caller)
- **Testability**: Tests can provide deterministic UUIDs for reproducibility
- **Flexibility**: Easy to switch UUID versions (v4, v7) without schema changes
- **No Round-Trip**: Client doesn't need to query `RETURNING id` to learn the generated UUID

**Trade-off**: Application must remember to generate UUIDs (enforced at compile time by sqlc's required parameters).

### 8.3 Thread Safety

**DB Struct Concurrency**: The `DB` struct is **safe for concurrent use** across multiple goroutines.

**Why**:
- The underlying `pgxpool.Pool` is explicitly designed for concurrent access
- No mutable state exists in the `DB` struct beyond the pool
- All query methods accept `context.Context` for per-operation lifecycle management

**Best Practice**: Create one `DB` instance per application (or Lambda container) and reuse it across requests.

**Lambda Pattern**:
```go
var db *echosubdb.DB

func init() {
    db, err = echosubdb.New(context.Background(), os.Getenv("DATABASE_URL"))
    // Handle error
}

func handler(ctx context.Context, event Event) (Response, error) {
    // Reuse db instance across invocations (container reuse)
    video, err := db.GetVideo(ctx, videoID)
    // ...
}
```

### 8.4 Connection Pooling

**Development/Test**: pgxpool handles local connection management with default settings (sufficient for single-developer usage).

**Production (Supabase)**: Two-layer pooling strategy:

1. **Supabase PgBouncer**: Serverless-optimized connection pooler
   - URL points to `[project-ref].[region].pooler.supabase.com:6543` (not direct PostgreSQL)
   - Handles global connection limits across all Lambda invocations
   - Manages connection reuse across containers

2. **pgxpool (in module)**: Local pool within each Lambda container
   - Reduces overhead of creating new connections for every query
   - Maintains small pool of connections to Supabase pooler
   - Uses pgxpool defaults (no custom configuration)

**No Custom Pool Configuration**: The module uses pgxpool defaults. This keeps the implementation simple and relies on battle-tested defaults.

### 8.5 Foreign Key Cascade Behavior

Foreign keys enforce referential integrity with the following cascade rules:

**CASCADE DELETE**:
- `Transcription.VideoID` → `Video.ID` (CASCADE)
  - Deleting a video automatically deletes its transcription
- `Translation.VideoID` → `Video.ID` (CASCADE)
  - Deleting a video automatically deletes all translations
- `Translation.TranscriptionID` → `Transcription.ID` (CASCADE)
  - Deleting a transcription automatically deletes all translations derived from it

**SET NULL**:
- `Job.VideoID` → `Video.ID` (SET NULL)
  - Deleting a video sets `jobs.video_id` to NULL
  - Job record remains for historical tracking (doesn't erase that processing was attempted)

**Benefits**:
- **Data Integrity**: Prevents orphaned records (translation without video)
- **Convenience**: Application doesn't need to manually delete dependent records
- **Performance**: Database handles cascade in a single transaction (more efficient than application-side loops)

### 8.6 Fingerprint Matching

**Algorithm**: Video identification uses perceptual hashing with Hamming distance similarity matching.

**Database Function**: PostgreSQL 16 provides `bit_count()` for popcount. A custom function wraps this:

```sql
CREATE FUNCTION hamming_distance(a BIGINT, b BIGINT) RETURNS INTEGER AS $$
    SELECT bit_count(a # b)::INTEGER;
$$ LANGUAGE SQL IMMUTABLE PARALLEL SAFE;
```

- `#` is XOR operator (differences between bit patterns)
- `bit_count()` counts set bits (number of differing bits)
- `IMMUTABLE` allows PostgreSQL to optimize and parallelize
- `PARALLEL SAFE` enables parallel query execution

**Query Logic** (FindVideosByFingerprint):
1. **Duration Filter**: `WHERE duration_ms BETWEEN (target - 100) AND (target + 100)`
2. **Hamming Distance Calculation**: Compute distance for each of 5 hash positions using `hamming_distance()`
3. **Average Distance**: `(dist_10 + dist_25 + dist_50 + dist_75 + dist_90) / 5.0`
4. **Threshold Filter**: `WHERE avg_distance <= maxAvgDistance`
5. **Ordering**: `ORDER BY avg_distance ASC` (best matches first)

**Single Query**: All steps execute in one SQL query. PostgreSQL computes distances in-memory for the filtered candidate set.

**Why 5 Hash Positions**: Temporal distribution captures visual characteristics across timeline, robust to scene changes at one position, balanced precision/performance.

## 9. Testing Strategy

### 9.1 Testing Approach

**90% Integration Tests**: All CRUD operations, fingerprint matching, error handling against real PostgreSQL 16

**10% Unit Tests**: Pure logic (error mapping functions, validation helpers) that don't require database access

**Rationale**: This module is a thin wrapper around database operations. The value—and risk—lies in correct SQL queries, proper marshaling/unmarshaling, and connection handling. Unit tests would require mocking the database layer (duplicates implementation logic without verifying correctness). Integration tests directly verify that queries execute correctly against PostgreSQL.

### 9.2 Test Database Setup

**Environment**:
- PostgreSQL 16 in Docker (same container used for development)
- Separate `test` database (isolated from `development` database)
- Connection via `DATABASE_URL` environment variable:
  ```
  DATABASE_URL="postgres://postgres:123456@localhost:5432/test?sslmode=disable"
  ```

**Schema Initialization**: Before running tests, migrations must be applied:
```bash
make migrate-apply-test
```

The `Makefile` test target handles this automatically.

### 9.3 Test Structure

**Pattern**:
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
}
```

**Testing Libraries**:
- `testing` (standard library): Test framework
- `github.com/stretchr/testify/assert` and `require`: Assertions

### 9.4 Test Isolation

**Strategy**: Table truncation after each test run (not transaction rollback).

**Why Truncation Over Transactions**:
- Simplicity: No need to wrap every test in BEGIN/ROLLBACK
- Matches Production Behavior: Tests see committed data, same as production
- Migration Testing: Can verify CASCADE delete behavior directly

**Trade-off**: Tests cannot run in parallel (truncation would interfere). Tests must run sequentially with `go test -p 1`.

**Fixture Management**: Test data created using the module's own public API methods (dogfooding, finds API ergonomics issues, DRY principle).

### 9.5 Test Coverage Goals

**Target Coverage**:
- **100% of public API methods**: Every exported function in `db.go` has at least one integration test
- **Happy Path**: All CRUD operations work correctly
- **Error Cases**: Not found, duplicate key, connection errors
- **Edge Cases**: Fingerprint matching with no matches, existence checks, nullable fields

**Not Covered**: Database crashes mid-query, network partitions, concurrency stress tests (pgxpool is already battle-tested).

## 10. Environment Configuration

### 10.1 DATABASE_URL Formats

The module connects via a `DATABASE_URL` environment variable. The format differs by environment:

**Local Development**:
```
postgres://postgres:123456@localhost:5432/development?sslmode=disable
```
- Host: `localhost`
- Port: `5432` (PostgreSQL default)
- Database: `development`
- SSL: Disabled (local trust)

**Local Test**:
```
postgres://postgres:123456@localhost:5432/test?sslmode=disable
```
- Host: `localhost`
- Port: `5432` (PostgreSQL default)
- Database: `test`
- SSL: Disabled (local trust)

**Production (Supabase)**:
```
postgres://[project-ref].[region].pooler.supabase.com:6543/postgres?sslmode=require
```
- Host: `[project-ref].[region].pooler.supabase.com` (Supabase connection pooler)
- Port: `6543` (Supabase pooler, NOT standard PostgreSQL 5432)
- Database: `postgres` (default Supabase database name)
- SSL: Required (enforced encryption)
- Pooling: Implicit (URL points to PgBouncer instance)

### 10.2 Environment Differences

| Aspect | Development/Test | Production (Supabase) |
|--------|------------------|----------------------|
| **Hosting** | Local Docker container | Supabase managed PostgreSQL |
| **Port** | 5432 (PostgreSQL default) | 6543 (Supabase pooler) |
| **SSL** | Disabled (`sslmode=disable`) | Required (`sslmode=require`) |
| **Pooling** | pgxpool only (local) | Two-layer (Supabase PgBouncer + pgxpool) |
| **Data Persistence** | Local filesystem (`docker/psql/data/`) | Supabase storage (persistent, backed up) |
| **Schema Management** | `make migrate-apply` (local) | Deployment pipeline applies migrations |
