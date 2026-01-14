# Design Document: AWS Lambda Go Starter Package

## Overview

The AWS Lambda Go Starter Package (`prethora/starter_golang_aws_lambda`) is a production-ready template for building AWS Lambda functions with Go and Docker. It provides a complete foundation supporting three Lambda invocation patterns through a single, intelligent Lambda function: API Gateway HTTP endpoints, direct custom event invocations, and AWS Step Functions state machines.

This is a **template project** designed to be cloned and customized for specific microservices. It eliminates boilerplate and configuration complexity, providing a battle-tested starting point with infrastructure, deployment, and testing patterns already solved.

**Primary Goal**: Enable rapid spinup of new Lambda-based microservices without re-solving Docker/Terraform/Lambda configuration each time.

## High-Level Project Structure

The project is organized to separate concerns clearly while maintaining simplicity for template users:

```
starter_golang_aws_lambda/
├── cmd/
│   └── lambda/
│       └── main.go              # Lambda entry point and initialization
├── internal/
│   ├── handlers/
│   │   ├── apigateway.go        # API Gateway event handler
│   │   ├── customevent.go       # Custom event handler
│   │   └── stepfunction.go      # Step Function handler
│   └── router/
│       └── router.go            # Event type detection and routing
├── terraform/                   # Infrastructure-as-code
├── tests/                       # Unit and E2E tests
├── scripts/                     # Deployment automation
├── Dockerfile                   # Multi-stage Docker build
├── config.env                   # Single configuration file
└── README.md                    # Comprehensive documentation
```

### Rationale for Structure

**Lambda Entry Point (`cmd/lambda/main.go`)**:
- Single entry point for AWS Lambda runtime
- Minimal responsibility: initialize dependencies and register the router's handler
- Follows Go convention of executable code in `cmd/` directory
- Makes it clear where the Lambda starts executing

**Internal Packages (`internal/`)**:
- Uses Go's `internal/` pattern to prevent external imports
- Appropriate for template code that will be customized per-project
- Signals to users: "this is implementation detail you'll modify"

**Handler Separation (`internal/handlers/`)**:
- Each invocation pattern gets its own handler file
- Users can easily locate and customize specific handler behavior
- Supports independent development and testing of each pattern
- Enables clean removal of unused patterns (delete file + update router)

**Router Package (`internal/router/`)**:
- Centralizes event type detection logic
- Single responsibility: determine event source and dispatch
- Decouples detection mechanism from handler implementations
- Makes the "intelligent routing" feature explicit and maintainable

**Infrastructure and Tooling at Root**:
- Terraform, Dockerfile, scripts at root level for discoverability
- Template users need to find these quickly
- Flat structure for operational files reduces navigation complexity

## Package Organization Strategy

### `internal/handlers` Package

Contains three handler files, each responsible for one invocation pattern:

**`apigateway.go`**:
- Handles `events.APIGatewayProxyRequest` events from API Gateway
- Responsible for HTTP request processing and response formatting
- Returns `events.APIGatewayProxyResponse` with status codes, headers, and body
- Encapsulates all API Gateway-specific logic

**`customevent.go`**:
- Handles arbitrary JSON payloads from direct Lambda invocations
- Provides generic event echo capability
- Returns structured response wrapping the input event
- Demonstrates how to handle custom business events

**`stepfunction.go`**:
- Handles Step Functions task invocations with state management
- Maintains counter state across iterations
- Returns progress status (`counter`, `iterations_remaining`, `complete`)
- Demonstrates stateful processing across multiple invocations

Each handler file exports a single handler function with a consistent signature, making the router's job straightforward.

### `internal/router` Package

Contains the event detection and routing logic:

**`router.go`**:
- Exports the main handler function registered with AWS Lambda runtime
- Receives `context.Context` and `json.RawMessage` (raw event payload)
- Detects event type by inspecting event structure
- Dispatches to appropriate handler
- Returns handler result or routing errors

This package is the "brain" of the multi-pattern support - it makes the single Lambda function intelligent about its invocation context.

### Customization-Friendly Organization

This structure supports the template's goal of easy customization:

1. **Add Custom Logic**: Users modify handler files directly - clear where business logic goes
2. **Remove Unused Patterns**: Delete handler file + remove from router - no complex dependency cleanup
3. **Add New Patterns**: Create new handler file + add case to router - follows established pattern
4. **Test Handlers**: Each handler can be unit tested independently - no router involvement needed

## Event Type Detection and Routing Mechanism

The core technical challenge is: **How does a single Lambda function determine which invocation pattern triggered it?**

### Detection Strategy

AWS Lambda events have distinct structural characteristics that allow automatic detection:

**API Gateway Events**:
- Contain `requestContext` field with API Gateway metadata
- Have `httpMethod`, `path`, `headers` fields
- Structure matches `events.APIGatewayProxyRequest` from `aws-lambda-go`
- Detection: Check for presence of `requestContext.apiId` or `requestContext.requestId`

**Step Functions Events**:
- Contain custom fields defined by the state machine: `start`, `iterations`, `delay_ms`
- May include Step Functions execution context
- Have specific expected schema for the iteration logic
- Detection: Check for presence of expected fields (`start`, `iterations`) and absence of API Gateway fields

**Custom Events**:
- Arbitrary JSON payloads that don't match other patterns
- Fallback category when event doesn't match API Gateway or Step Functions
- Detection: If event doesn't match API Gateway or Step Functions characteristics, treat as custom

### Routing Flow

```
Lambda Invocation
    ↓
Router receives json.RawMessage
    ↓
Inspect event structure
    ↓
┌────────────────────────────────┐
│  Has API Gateway fields?       │ → Yes → API Gateway Handler
│  requestContext, httpMethod    │
└────────────────────────────────┘
    ↓ No
┌────────────────────────────────┐
│  Has Step Function fields?     │ → Yes → Step Functions Handler
│  start, iterations             │
└────────────────────────────────┘
    ↓ No
Custom Event Handler (fallback)
```

### Router Responsibilities

1. **Event Unmarshaling**: Parse `json.RawMessage` to inspect structure
2. **Type Detection**: Apply detection rules in priority order
3. **Handler Dispatch**: Call appropriate handler with typed event
4. **Error Handling**: Return clear errors for unparseable events
5. **Response Marshaling**: Ensure handler response is properly formatted for Lambda runtime

### Design Benefits

**Automatic Context Detection**:
- Users don't specify which handler to use - it's automatic
- Single Lambda function deployed once handles all patterns
- Reduces infrastructure complexity (one function, not three)

**Graceful Degradation**:
- Unknown event formats fall back to custom event handler
- Won't crash on unexpected events - will echo them
- Useful for debugging and gradual pattern addition

**Clean Handler Contract**:
- Handlers don't need to know about routing logic
- Each handler assumes its event type is correct
- Separation of concerns between detection and processing

## Handler Responsibilities

### API Gateway Handler

**Purpose**: Process HTTP requests from API Gateway and return HTTP responses.

**Input**: API Gateway event containing HTTP request details (method, path, headers, body, query parameters)

**Output**: API Gateway response with status code, headers, and JSON body

**Behavior**:
- Responds to `GET /` with `{"message": "Hello World", "source": "api-gateway"}`
- Returns proper HTTP response structure expected by API Gateway
- Handles HTTP-specific concerns (status codes, content-type headers)
- Provides foundation for users to add REST API routes

**Customization Point**: Users replace demo logic with their API routes and business logic

### Custom Event Handler

**Purpose**: Demonstrate handling of arbitrary business events not tied to specific AWS services.

**Input**: Any valid JSON payload from direct Lambda invocation

**Output**: Structured response echoing the input: `{"source": "custom-event", "echo": <original_event>}`

**Behavior**:
- Accepts any JSON structure
- Echoes entire input back wrapped in response envelope
- Demonstrates Lambda's flexibility for event-driven architectures
- Provides starting point for custom business event processing

**Customization Point**: Users replace echo logic with domain-specific event processing (e.g., process transcription job, handle webhook payload)

### Step Functions Handler

**Purpose**: Manage stateful iteration logic for Step Functions state machines.

**Input**: State machine input with `start`, `iterations`, `delay_ms` fields

**Output**: Progress status: `{"counter": N, "iterations_remaining": M, "complete": bool}`

**Behavior**:
- Maintains counter state across invocations
- Increments counter and calculates remaining iterations
- Returns completion flag when iterations are exhausted
- Step Functions state machine handles loop and wait logic
- Demonstrates how Lambda integrates with Step Functions for workflows

**Customization Point**: Users replace iteration demo with actual workflow steps (e.g., batch processing, multi-stage data transformations)

### Handler Independence

Each handler is **independent and self-contained**:

- **No Shared State**: Handlers don't communicate with each other
- **No Cross-Dependencies**: API Gateway handler doesn't import Step Functions handler
- **Isolated Testing**: Each handler can be unit tested without others
- **Independent Modification**: Changing one handler doesn't affect others

This independence is critical for the template goal: users should be able to delete unused handlers without breaking the remaining ones.

### Handler Customization Flow

The template's handler structure guides users through customization:

1. **Identify Needed Patterns**: Decide which invocation patterns are needed for the microservice
2. **Remove Unused Handlers**: Delete handler files for unused patterns, remove from router
3. **Customize Remaining Handlers**: Replace demo logic with business logic
4. **Add Domain Logic**: Import domain packages, call services, transform data
5. **Maintain Handler Contracts**: Keep input/output structures compatible with event sources

The clear separation makes it obvious where to add custom code while maintaining the template's infrastructure and deployment automation.

## Design Principles

This foundational architecture is guided by several key principles aligned with the client's needs:

**Clone and Deploy**:
- Structure is simple and flat - no deep nesting
- Clear separation between "what users modify" (handlers) and "what stays stable" (router, infrastructure)
- Obvious entry point (main.go) and obvious customization points (handler files)

**Single Function, Multiple Patterns**:
- Router enables intelligent handling without external configuration
- Reduces AWS resource complexity (one function instead of three)
- Simplifies deployment (one Docker image, one Terraform resource)

**Easy Customization**:
- Handler files are self-contained and independently modifiable
- Removing patterns is as simple as deleting files
- Adding patterns follows established conventions

**Production Ready**:
- Clean package structure follows Go best practices
- Clear separation of concerns enables testing
- Handler independence prevents cascade failures

## External Dependencies

The starter package requires carefully selected external dependencies that balance functionality, maintainability, and deployment efficiency.

### Dependency Categories

Dependencies are separated into two categories based on where they're used:

**Runtime Dependencies** (included in Lambda deployment):
- Required by the Lambda function code itself
- Compiled into the final Go binary
- Impact deployment size and cold start time
- Must be minimal and well-maintained

**Test-Only Dependencies** (excluded from Lambda deployment):
- Only used by unit and end-to-end tests
- Not compiled into Lambda binary
- Can be heavier since they don't affect production performance
- Enable comprehensive testing without bloating the deployment

### Runtime Dependencies

#### AWS Lambda Go Runtime (v1.51.2)

**Package**: `github.com/aws/aws-lambda-go`
**Version**: v1.51.2 (released January 12, 2025)
**Repository**: https://github.com/aws/aws-lambda-go

**Purpose**:
- Official AWS Lambda runtime library for Go
- Provides the Lambda handler interface and execution model
- Includes event type definitions for all AWS event sources
- Handles communication between Lambda runtime and handler code

**What It Provides**:
- `lambda.Start()` - Entry point for Lambda functions
- `events.APIGatewayProxyRequest` / `events.APIGatewayProxyResponse` - API Gateway types
- Event structures for Step Functions, S3, SNS, SQS, and more
- Context helpers for request ID, deadline, function metadata

**Why This Library**:
1. **Official AWS Support**: Maintained by AWS, guaranteed compatibility with Lambda runtime
2. **Complete Event Coverage**: Includes type definitions for all AWS event sources
3. **Runtime Protocol**: Handles Lambda Runtime API communication automatically
4. **Zero Alternatives**: This is the standard Go Lambda library - no viable alternatives exist
5. **Lightweight**: Minimal dependencies, small binary size impact

**Version Strategy**: Use v1.51.2 as baseline (January 2025), but implementers should check for newer stable releases. This library is actively maintained with regular updates for new Lambda features.

### Test-Only Dependencies

The following dependencies are only imported by the `tests/` directory and are never compiled into the Lambda deployment binary.

#### AWS SDK for Go v2 - Core (v1.41.1)

**Package**: `github.com/aws/aws-sdk-go-v2`
**Version**: v1.41.1 (published January 9, 2026)
**Repository**: https://github.com/aws/aws-sdk-go-v2

**Purpose**: Core SDK module providing foundational AWS client functionality

**Why SDK v2 (Not v1)**:
- **v1 End-of-Support**: `aws-sdk-go` (v1) reached end-of-support - only v2 receives updates
- **Modern Design**: v2 uses Go modules properly and has better API design
- **Current Standard**: All new AWS Go projects should use v2

**Important Constraint**: Requires Go 1.23 or later (this sets our minimum Go version)

#### AWS SDK for Go v2 - Configuration (v1.32.7)

**Package**: `github.com/aws/aws-sdk-go-v2/config`
**Version**: v1.32.7 (published January 9, 2026)

**Purpose**: Handles AWS credential and region configuration

**What It Provides**:
- Loads credentials from environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
- Supports IAM role credentials (useful for CI/CD environments)
- Handles AWS region configuration
- Credential chain resolution following AWS SDK conventions

**Used By**: E2E test suite for authenticating with AWS to invoke deployed resources

#### AWS SDK for Go v2 - Lambda Service (Latest January 2026)

**Package**: `github.com/aws/aws-sdk-go-v2/service/lambda`
**Version**: Latest from January 2026 release cycle

**Purpose**: Lambda service client for programmatic function invocation

**What It Provides**:
- `Invoke()` - Direct Lambda function invocation
- Payload marshaling and response unmarshaling
- Synchronous and asynchronous invocation modes

**Used By**: E2E tests for the **custom event handler pattern**
- Test script invokes deployed Lambda with arbitrary JSON payloads
- Verifies that custom event handler correctly echoes input
- Validates handler selection logic for non-API Gateway events

#### AWS SDK for Go v2 - Step Functions Service (Latest January 2026)

**Package**: `github.com/aws/aws-sdk-go-v2/service/sfn`
**Version**: Latest from January 2026 release cycle

**Purpose**: Step Functions service client for state machine operations

**What It Provides**:
- `StartExecution()` - Initiate state machine execution
- `DescribeExecution()` - Poll execution status
- Execution history and output retrieval

**Used By**: E2E tests for the **Step Functions handler pattern**
- Test script starts a state machine execution
- Polls until completion or timeout
- Verifies iteration counter reached expected value
- Validates handler's stateful processing logic

### Dependency Scope and Binary Size

**Critical Design Decision**: The Lambda function binary only includes `aws-lambda-go`.

The AWS SDK v2 dependencies are **never imported by Lambda function code**:
- `cmd/lambda/main.go` - Only imports `aws-lambda-go`
- `internal/handlers/*.go` - Only import `aws-lambda-go/events`
- `internal/router/*.go` - Only imports `aws-lambda-go` and `encoding/json`

The SDK v2 dependencies are **only imported by test code**:
- `tests/e2e/*.go` - Imports SDK v2 to invoke deployed resources
- No production code path imports SDK v2

**Why This Matters**:
1. **Smaller Deployment**: Lambda binary is ~15-20MB instead of ~50MB+ with full SDK
2. **Faster Cold Starts**: Less code to load means faster Lambda initialization
3. **Clear Separation**: Production code has minimal dependencies, test code has what it needs
4. **Template Simplicity**: Users see minimal dependencies in handler code

### Version Pinning Strategy

**Baseline Versions (January 2026)**:
- `aws-lambda-go`: v1.51.2
- `aws-sdk-go-v2`: v1.41.1
- `aws-sdk-go-v2/config`: v1.32.7
- Service clients (lambda, sfn): Latest from January 2026

**Versioning Approach**:

The client explicitly requires "latest stable versions at implementation time" rather than relying on outdated training data. These versions represent a **January 2026 baseline**.

**Recommendation for Implementers**:
1. **Check for Newer Releases**: Before implementation, check GitHub releases for all dependencies
2. **Use Patch Updates**: Configure `go.mod` to allow patch version updates (`~v1.51` allows v1.51.x)
3. **Test Before Minor Bumps**: Pin specific versions but test newer minors before upgrading
4. **Document Versions**: Include actual version numbers in README prerequisites

**Example `go.mod` constraints**:
```
require (
    github.com/aws/aws-lambda-go v1.51.2           // Pin exact version
    github.com/aws/aws-sdk-go-v2 v1.41.1           // Pin exact version
    github.com/aws/aws-sdk-go-v2/config v1.32.7    // Pin exact version
)
```

Alternative with minor version flexibility:
```
require (
    github.com/aws/aws-lambda-go v1.51             // Allow v1.51.x patches
    github.com/aws/aws-sdk-go-v2 v1.41             // Allow v1.41.x patches
)
```

### Go Version Requirement

**Minimum Go Version**: **1.23**

**Rationale**:
- AWS SDK for Go v2 (v1.41.1) requires Go 1.23 or later
- Using Go 1.23+ ensures compatibility with all dependencies
- Go 1.23 includes performance improvements and standard library updates
- Maintains modern Go toolchain alignment

**Recommendation**: Use **Go 1.23 or the latest stable Go version** at implementation time. The client requires using current stable versions, so if Go 1.24 is released before implementation, use that instead.

**Docker Implications**: The Dockerfile builder stage must use a Go 1.23+ base image.

### Dependency Maintenance

**Active Maintenance Status** (as of January 2026):
- `aws-lambda-go`: Active - AWS maintains with Lambda feature releases
- `aws-sdk-go-v2`: Active - Daily releases with service updates
- All packages have recent commits and ongoing development

**Long-term Stability**:
- `aws-lambda-go` is the official Lambda runtime - will be maintained as long as AWS supports Go on Lambda
- `aws-sdk-go-v2` is AWS's current-generation SDK - will receive long-term support
- No risk of dependency abandonment

**Security Updates**:
- AWS actively patches security issues
- Recommend periodic dependency updates (quarterly review cycle)
- Use `go list -m -u all` to check for available updates

### Integration Points

**Handler Code** (`internal/handlers/*.go`):
```go
import (
    "github.com/aws/aws-lambda-go/events"
)

func HandleAPIGateway(req events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Handler implementation
}
```

**Main Entry Point** (`cmd/lambda/main.go`):
```go
import (
    "github.com/aws/aws-lambda-go/lambda"
)

func main() {
    lambda.Start(router.Handler)
}
```

**E2E Tests** (`tests/e2e/*.go`):
```go
import (
    "github.com/aws/aws-sdk-go-v2/config"
    "github.com/aws/aws-sdk-go-v2/service/lambda"
    "github.com/aws/aws-sdk-go-v2/service/sfn"
)

// Test code invokes deployed Lambda and Step Functions
```

The dependency design establishes a clean separation: lightweight runtime dependencies for production, comprehensive SDK dependencies for testing.

## Docker Containerization

AWS Lambda supports containerized functions, enabling complete control over the runtime environment. The starter package uses a multi-stage Docker build to produce a minimal, optimized Lambda container image.

### Multi-Stage Build Strategy

The Dockerfile uses a **two-stage build** that separates compilation from runtime:

**Stage 1: Builder**
- Contains full Go toolchain (compiler, linker, build tools)
- Includes all source code and dependencies
- Performs compilation with static linking
- Produces a single, self-contained binary
- Discarded after build - not part of final image

**Stage 2: Runtime**
- Minimal AWS Lambda base image
- Contains only the compiled binary
- No build tools, source code, or unnecessary files
- Results in smallest possible image size
- This is what gets deployed to Lambda

**Why Two Stages?**
1. **Size Reduction**: Builder stage can be 800MB+, final image is <50MB
2. **Security**: No build tools or source code in production image
3. **Performance**: Smaller images mean faster Lambda cold starts
4. **Clean Separation**: Build concerns separate from runtime concerns

### Builder Stage Design

**Base Image**: `golang:1.23-alpine` (or latest stable Go version)

**Why Alpine**:
- Minimal base image (~5MB vs ~800MB for standard Golang image)
- Contains necessary build tools (gcc, musl-dev for CGO compatibility)
- Well-maintained and security-focused
- Fast layer downloads

**Build Process**:

1. **Set Working Directory**: `/build` for compilation workspace
2. **Copy Dependency Files First**: `go.mod` and `go.sum` only
   - Enables Docker layer caching
   - Dependencies rarely change, so this layer is usually cached
   - Subsequent builds skip dependency download if unchanged
3. **Download Dependencies**: `go mod download`
   - Downloads all packages specified in go.mod
   - Cached layer unless dependencies change
4. **Copy Source Code**: Copy `cmd/` and `internal/` directories
   - Source changes frequently, so this layer rebuilds often
   - But dependency layer above is still cached
5. **Compile Binary**: Build statically-linked executable
   - Output: Single binary with all dependencies embedded
   - No external library dependencies required at runtime

**Static Linking Requirements**:

Lambda containers require statically-linked binaries - all dependencies must be embedded in the executable. This is achieved through build flags:

```bash
CGO_ENABLED=0 \
GOOS=linux \
GOARCH=amd64 \
go build -ldflags='-w -s' -o bootstrap cmd/lambda/main.go
```

**Build Flag Explanations**:
- `CGO_ENABLED=0`: Disable CGO, forces pure Go compilation with no C dependencies
- `GOOS=linux`: Target operating system is Linux (Lambda environment)
- `GOARCH=amd64`: Target architecture is x86-64 (alternatively arm64 for Graviton)
- `-ldflags='-w -s'`: Strip debug information and symbol table (reduces binary size ~30%)
- `-o bootstrap`: Output executable named 'bootstrap' (Lambda runtime requirement)

**Why Static Linking?**:
- No dependency on host system libraries
- Binary runs in any Linux environment (Lambda uses Amazon Linux 2023)
- No runtime linker overhead
- Eliminates "library not found" errors

### Runtime Stage Design

**Base Image**: `public.ecr.aws/lambda/provided:al2023`

**What is `provided:al2023`?**:
- AWS-maintained public container image for custom Lambda runtimes
- Based on Amazon Linux 2023 (latest AWS-recommended Linux distribution)
- Includes Lambda Runtime Interface Client (RIC) for handling invocations
- Pre-configured for Lambda execution environment
- Optimized for cold start performance

**Why This Image?**:
1. **Official AWS Support**: Maintained by AWS, guaranteed Lambda compatibility
2. **Latest Runtime**: `al2023` supersedes `al2` (Amazon Linux 2)
3. **Includes RIC**: Runtime Interface Client handles Lambda event lifecycle
4. **Optimized**: Tuned for Lambda's execution environment and performance characteristics
5. **Security**: Regular security updates from AWS

**Alternative**: `public.ecr.aws/lambda/provided:al2` (older, but still supported)
- Use al2023 for new projects unless specific al2 compatibility needed

### Executable Naming and Placement

**Lambda Runtime Requirements**:

Lambda's `provided` runtime looks for an executable to run. The runtime follows this priority:

1. **ENTRYPOINT** (if specified in Dockerfile)
2. **CMD** (if specified and no ENTRYPOINT)
3. **Executable named `bootstrap`** in `/var/task/` or `$LAMBDA_TASK_ROOT`

**Starter Package Approach**: Name the binary `bootstrap`

**Placement**:
- Copy `bootstrap` to `/var/task/bootstrap`
- This is Lambda's working directory (`$LAMBDA_TASK_ROOT`)
- Runtime automatically finds and executes `bootstrap`

**Why `bootstrap` Name?**:
- Convention for custom Lambda runtimes
- No need to specify ENTRYPOINT or CMD in Dockerfile
- Runtime automatically discovers it
- Clear signal that this is a Lambda entry point

**File Permissions**: Executable must have execute permission (`chmod +x bootstrap`)

### Build Optimization Strategy

**Docker Layer Caching**:

```dockerfile
# Stage 1: Builder
FROM golang:1.23-alpine AS builder
WORKDIR /build

# Copy dependency files FIRST (changes infrequently)
COPY go.mod go.sum ./
RUN go mod download

# Copy source code AFTER (changes frequently)
COPY cmd/ cmd/
COPY internal/ internal/

# Build
RUN CGO_ENABLED=0 GOOS=linux GOARCH=amd64 \
    go build -ldflags='-w -s' -o bootstrap cmd/lambda/main.go

# Stage 2: Runtime
FROM public.ecr.aws/lambda/provided:al2023
COPY --from=builder /build/bootstrap ${LAMBDA_TASK_ROOT}/bootstrap
CMD [ "bootstrap" ]
```

**Layer Caching Benefits**:
- `go.mod` changes: Only dependency download layer rebuilds
- Source code changes: Dependency layer is cached, only compilation runs
- No changes: Entire build uses cache, rebuilds in seconds

**Minimal Final Image**:

The runtime stage contains **only**:
- AWS Lambda base image (~30MB)
- Compiled `bootstrap` binary (~15-20MB)
- **Total: ~45-50MB**

What's **NOT** included:
- Go compiler and build tools
- Source code files
- Intermediate build artifacts
- Test files
- Git repository data

**Security Considerations**:

- **No Source Code**: Production image doesn't contain source - reduces information leakage risk
- **No Build Tools**: Can't compile code in running container
- **Minimal Attack Surface**: Fewer packages means fewer potential vulnerabilities
- **Lambda Runtime User**: AWS Lambda runs as non-root user automatically

### Cross-Platform Build Support

**Development Environment Portability**:

The client requires support for:
- macOS (arm64 M1/M2/M3, amd64 Intel)
- Linux (amd64)
- Windows with WSL2

**Docker Solves Cross-Platform Compilation**:

Docker builds produce **consistent Linux amd64 binaries** regardless of host platform:

- **macOS arm64** (M1/M2/M3): Docker builds amd64 Linux binary via emulation
- **macOS amd64** (Intel): Docker builds amd64 Linux binary natively
- **Linux amd64**: Docker builds amd64 Linux binary natively
- **Windows WSL2**: Docker in WSL2 builds amd64 Linux binary

**How It Works**:
1. Developer runs `docker build` on their platform (Mac/Linux/Windows)
2. Builder stage uses `golang:1.23-alpine` with `GOARCH=amd64`
3. Go cross-compiler produces Linux amd64 binary
4. Binary runs identically in Lambda regardless of where it was built

**No Platform-Specific Code Required**:
- Developers don't need Linux machines
- Developers don't need to configure cross-compilation manually
- CI/CD systems work the same way
- Template is truly clone-and-build on any supported platform

**ARM64/Graviton Support (Optional)**:

For AWS Graviton processors (arm64), change build flag:
```bash
GOARCH=arm64
```

Lambda base image also supports arm64:
```dockerfile
FROM public.ecr.aws/lambda/provided:al2023-arm64
```

**Why amd64 Default?**:
- Widest compatibility
- Most Lambda deployments use x86-64
- Template users can switch to arm64 if needed for cost savings

### Build Process for Static Linking

**Compilation Command**:

```bash
CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build \
  -ldflags='-w -s' \
  -trimpath \
  -o bootstrap \
  cmd/lambda/main.go
```

**Flag Details**:

- **`CGO_ENABLED=0`**: Critical for static linking
  - Disables CGO (Go's C interop)
  - Forces pure Go compilation
  - Results in binary with zero external dependencies
  - Required for Lambda's `provided` runtime

- **`GOOS=linux`**: Target OS is Linux
  - Lambda runs Amazon Linux 2023
  - Binary must be Linux-compatible

- **`GOARCH=amd64`**: Target architecture
  - Standard x86-64 processors
  - Alternative: `arm64` for Graviton

- **`-ldflags='-w -s'`**: Linker flags for size reduction
  - `-w`: Omit DWARF debug information
  - `-s`: Omit symbol table
  - Reduces binary size ~30% (from ~25MB to ~18MB)
  - Debug info not useful in production Lambda

- **`-trimpath`**: Remove absolute file paths from binary
  - Enhances reproducibility
  - Removes build machine details
  - Slight security improvement (doesn't leak local paths)

**Result**: Single-file, statically-linked Linux executable with no external dependencies

### Docker Build Optimization Strategy

**Minimizing Image Size**:

1. **Multi-stage build**: Discard 800MB+ builder, keep 50MB runtime
2. **Alpine builder**: Start with minimal base (~5MB vs ~800MB)
3. **Strip debug symbols**: `-ldflags='-w -s'` saves ~30% binary size
4. **AWS Lambda base**: Pre-optimized, minimal runtime image
5. **No unnecessary files**: Only copy the binary, nothing else

**Minimizing Build Time**:

1. **Layer caching**: `go.mod` before source code caching
2. **Parallel builds**: Docker BuildKit parallelizes layers when possible
3. **Dependency download once**: Cached until `go.mod` changes

**Example Build Times**:
- First build: 60-90 seconds (download dependencies, compile)
- Source-only change: 10-15 seconds (recompile only)
- No changes: 1-2 seconds (all cached)

## Terraform Infrastructure Organization

The starter package uses Terraform to define all AWS infrastructure as code. Infrastructure is organized into logical modules with a single configuration point for easy customization.

### Terraform File Organization

Terraform code is separated across multiple files for maintainability and clarity:

```
terraform/
├── main.tf                  # Provider configuration and core settings
├── variables.tf             # All input variables with descriptions
├── outputs.tf               # All output values (URLs, ARNs)
├── lambda.tf                # Lambda function, IAM role, ECR, logs
├── api_gateway.tf           # HTTP API Gateway and integration
└── step_functions.tf        # State machine, IAM role, definition
```

**File Responsibilities**:

#### `main.tf` - Core Configuration

Contains:
- Terraform version requirements (`terraform {}` block)
- AWS provider configuration with region
- Backend configuration (if using remote state)
- No resource definitions - just configuration

**Purpose**: Centralize Terraform and provider setup. Users know to look here for Terraform version requirements and AWS configuration.

#### `variables.tf` - Input Variables

Defines all input variables:
- `stack_name` - Unique identifier for this deployment (required)
- `aws_region` - AWS region for deployment (default: "us-east-1")
- `enable_api_gateway` - Deploy API Gateway (default: true)
- `enable_step_functions` - Deploy Step Functions (default: true)
- `enable_custom_events` - Allow direct Lambda invocation (default: true)
- `lambda_memory` - Lambda memory allocation (default: 512)
- `lambda_timeout` - Lambda timeout in seconds (default: 30)

Each variable includes:
- `description` - Clear explanation of purpose
- `type` - Variable type (string, bool, number)
- `default` - Default value (except `stack_name` which is required)

**Purpose**: Single location for all configuration points. Users can customize deployment behavior by modifying variables.

#### `outputs.tf` - Output Values

Defines outputs that users need after deployment:
- `api_gateway_url` - HTTP endpoint for API Gateway (if enabled)
- `lambda_function_arn` - Lambda function ARN for programmatic invocation
- `lambda_function_name` - Lambda function name for AWS CLI usage
- `step_functions_arn` - State machine ARN (if enabled)
- `ecr_repository_url` - ECR repository URL for image push

**Purpose**: Provide deployment results to users and other automation. Scripts can parse outputs to get resource identifiers.

#### `lambda.tf` - Lambda Resources

Contains all Lambda-related resources:
- `aws_ecr_repository` - Container image registry
- `aws_lambda_function` - Lambda function definition
- `aws_iam_role` - Lambda execution role
- `aws_iam_role_policy_attachment` - Attach policies to role
- `aws_iam_policy` - Custom policies for Step Functions invocation
- `aws_cloudwatch_log_group` - Lambda logs destination

**Purpose**: Group all Lambda infrastructure together. If users want to modify Lambda configuration (memory, timeout, permissions), they edit this file.

#### `api_gateway.tf` - API Gateway Resources

Contains HTTP API Gateway infrastructure:
- `aws_apigatewayv2_api` - HTTP API definition
- `aws_apigatewayv2_integration` - Lambda integration
- `aws_apigatewayv2_route` - Route definition (GET /)
- `aws_apigatewayv2_stage` - Deployment stage ($default)
- `aws_lambda_permission` - Grant API Gateway invocation permission

**Purpose**: Isolate API Gateway configuration. Uses `count` to conditionally create based on `enable_api_gateway` variable.

#### `step_functions.tf` - Step Functions Resources

Contains state machine infrastructure:
- `aws_sfn_state_machine` - State machine definition
- `aws_iam_role` - Step Functions execution role
- `aws_iam_role_policy_attachment` - Policies for Lambda invocation
- State machine definition JSON (inline or file)

**Purpose**: Isolate Step Functions configuration. Uses `count` to conditionally create based on `enable_step_functions` variable.

**Why This Organization?**:
1. **Logical Grouping**: Related resources together (all Lambda stuff in lambda.tf)
2. **Easy Navigation**: Users know where to find specific resource types
3. **Independent Modification**: Change API Gateway without touching Step Functions
4. **Conditional Resources**: Easy to disable entire service categories
5. **Standard Pattern**: Follows common Terraform multi-file organization conventions

### Single Configuration Point Strategy

The client requires a **single configuration file** (`config.env`) that controls all resource naming. This eliminates configuration scattered across multiple files.

**Configuration Flow**:

```
config.env
    ↓
STACK_NAME=my-service
    ↓
Deployment script sources config.env
    ↓
terraform apply -var="stack_name=my-service"
    ↓
Terraform variables.tf receives stack_name
    ↓
All resources use ${var.stack_name}-suffix naming
    ↓
AWS resources created:
  - my-service-lambda
  - my-service-repo
  - my-service-api
  - my-service-state-machine
```

**Implementation Details**:

**1. config.env File**:
```bash
# config.env - Single configuration point
STACK_NAME=my-service

# Optional overrides
AWS_REGION=us-east-1
```

**2. Deployment Script Integration**:
```bash
# scripts/deploy.sh
source config.env

if [ -z "$STACK_NAME" ]; then
    echo "ERROR: STACK_NAME not set in config.env"
    exit 1
fi

terraform apply \
    -var="stack_name=$STACK_NAME" \
    -var="aws_region=${AWS_REGION:-us-east-1}"
```

**3. Terraform Variables Declaration**:
```hcl
# terraform/variables.tf
variable "stack_name" {
  description = "Unique name for this deployment (sets all resource names)"
  type        = string
  # No default - must be provided
}

variable "aws_region" {
  description = "AWS region for deployment"
  type        = string
  default     = "us-east-1"
}
```

**4. Resource Name Construction**:
```hcl
# terraform/lambda.tf
resource "aws_lambda_function" "main" {
  function_name = "${var.stack_name}-lambda"
  # ...
}

resource "aws_ecr_repository" "lambda" {
  name = "${var.stack_name}-repo"
}
```

**Why This Matters**:
- **Single Edit Point**: Change `STACK_NAME=my-service` to `STACK_NAME=prod-api`, redeploy - all resources renamed
- **No Terraform Editing**: Users never edit Terraform files for basic deployment
- **Multiple Environments**: Different config.env per environment (dev, staging, prod)
- **Prevents Conflicts**: Different stack names = isolated AWS resources

### Resource Naming Convention

All AWS resource names derive from `STACK_NAME` with consistent suffixes:

| AWS Resource | Naming Pattern | Example (STACK_NAME=demo) |
|--------------|----------------|---------------------------|
| Lambda Function | `${STACK_NAME}-lambda` | `demo-lambda` |
| ECR Repository | `${STACK_NAME}-repo` | `demo-repo` |
| API Gateway HTTP API | `${STACK_NAME}-api` | `demo-api` |
| Step Functions State Machine | `${STACK_NAME}-state-machine` | `demo-state-machine` |
| Lambda IAM Role | `${STACK_NAME}-lambda-role` | `demo-lambda-role` |
| Step Functions IAM Role | `${STACK_NAME}-sfn-role` | `demo-sfn-role` |
| CloudWatch Log Group | `/aws/lambda/${STACK_NAME}` | `/aws/lambda/demo` |

**Naming Convention Rules**:
1. **Consistent Suffix Pattern**: Resource type clearly indicated (-lambda, -api, -repo)
2. **Kebab-case**: Use hyphens, not underscores or camelCase
3. **Predictable**: Given STACK_NAME, you can predict all resource names
4. **AWS Constraints**: Respect AWS naming rules (length limits, allowed characters)

**Why Consistent Naming?**:
- **Discoverability**: In AWS Console, search for "demo-" finds all related resources
- **Scripting**: Scripts can construct resource names programmatically
- **Debugging**: Clear resource ownership and purpose from name alone
- **Multiple Deployments**: Different STACK_NAMEs = different resource sets = no conflicts

### Feature Toggle Variables

The starter supports **conditional resource creation** through boolean toggle variables. Users can disable unused invocation patterns to reduce infrastructure.

**Toggle Variables**:

```hcl
# terraform/variables.tf

variable "enable_api_gateway" {
  description = "Deploy API Gateway HTTP API for REST endpoints"
  type        = bool
  default     = true
}

variable "enable_step_functions" {
  description = "Deploy Step Functions state machine for workflows"
  type        = bool
  default     = true
}

variable "enable_custom_events" {
  description = "Allow direct Lambda invocation (programmatic event triggering)"
  type        = bool
  default     = true
}
```

**Implementation with Terraform `count`**:

```hcl
# terraform/api_gateway.tf

resource "aws_apigatewayv2_api" "main" {
  count = var.enable_api_gateway ? 1 : 0

  name          = "${var.stack_name}-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  count = var.enable_api_gateway ? 1 : 0

  api_id           = aws_apigatewayv2_api.main[0].id
  integration_type = "AWS_PROXY"
  # ...
}
```

**How It Works**:
- `count = var.enable_api_gateway ? 1 : 0`
  - If `enable_api_gateway = true`: Creates 1 instance of resource
  - If `enable_api_gateway = false`: Creates 0 instances (resource not created)
- Resource references use `[0]` index: `aws_apigatewayv2_api.main[0].id`
- All related resources use same condition (API, integration, routes, stages)

**Step Functions Toggle**:

```hcl
# terraform/step_functions.tf

resource "aws_sfn_state_machine" "main" {
  count = var.enable_step_functions ? 1 : 0

  name     = "${var.stack_name}-state-machine"
  role_arn = aws_iam_role.sfn[0].arn
  # ...
}
```

**Custom Events Toggle**:

This toggle affects Lambda **permissions**, not a separate resource:

```hcl
# terraform/lambda.tf

resource "aws_lambda_permission" "allow_direct_invocation" {
  count = var.enable_custom_events ? 1 : 0

  statement_id  = "AllowDirectInvocation"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "*"  # Or restrict to specific AWS accounts
}
```

**Usage Examples**:

**Disable API Gateway** (only Step Functions and custom events):
```hcl
# terraform.tfvars
enable_api_gateway = false
```

**Only Custom Events** (no API Gateway or Step Functions):
```hcl
# terraform.tfvars
enable_api_gateway    = false
enable_step_functions = false
enable_custom_events  = true
```

**Why Feature Toggles?**:
1. **Reduce Costs**: Don't pay for unused API Gateway or Step Functions
2. **Simplify Deployment**: Fewer resources = faster deploys, simpler IAM
3. **Template Flexibility**: Same template serves different use cases
4. **Easy Customization**: Add/remove features without editing resource definitions

### Environment Variable Handling

The starter distinguishes between **required** and **optional** configuration:

**Required Variables** (no defaults):

```bash
# config.env
STACK_NAME=my-service     # Required
```

**Optional Variables** (have defaults):

```bash
# config.env
AWS_REGION=us-west-2      # Optional, defaults to us-east-1
```

**Deployment Script Validation**:

```bash
# scripts/deploy.sh
source config.env

# Validate required variables
if [ -z "$STACK_NAME" ]; then
    echo "ERROR: STACK_NAME must be set in config.env"
    echo "Example: STACK_NAME=my-service"
    exit 1
fi

# Use defaults for optional variables
AWS_REGION=${AWS_REGION:-us-east-1}

echo "Deploying stack: $STACK_NAME"
echo "Region: $AWS_REGION"

# Pass to Terraform
terraform apply \
    -var="stack_name=$STACK_NAME" \
    -var="aws_region=$AWS_REGION"
```

**Why Validation?**:
- **Early Failure**: Catch missing config before Terraform runs (fast feedback)
- **Clear Errors**: Tell users exactly what's wrong and how to fix it
- **Prevent Accidents**: Don't allow deployment without stack name (could overwrite existing stack)

### Terraform Provider Configuration

**Provider Setup** (`terraform/main.tf`):

```hcl
terraform {
  required_version = ">= 1.6.0"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.aws_region

  default_tags {
    tags = {
      Project     = var.stack_name
      ManagedBy   = "Terraform"
      Environment = "production"
    }
  }
}
```

**Provider Configuration Details**:

- **Terraform Version**: Require >= 1.6.0 (latest stable as of January 2026 baseline)
- **AWS Provider Version**: Use ~> 5.0 (allows 5.x updates, blocks 6.0 breaking changes)
- **Region Sourcing**: Uses `var.aws_region` from variables.tf (set via config.env)
- **Default Tags**: All AWS resources automatically tagged with Project, ManagedBy, Environment

**AWS Credentials**:

Provider doesn't specify credentials - relies on standard AWS SDK credential chain:
1. Environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`)
2. AWS credentials file (`~/.aws/credentials`)
3. IAM role (when running in EC2/ECS/Lambda)

**Deployment Script Sets Credentials**:

```bash
# scripts/deploy.sh
export AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
export AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}

terraform apply ...
```

**Why No Hardcoded Credentials?**:
- **Security**: Never commit credentials to version control
- **Flexibility**: Different credential sources (env vars, files, IAM roles) all work
- **CI/CD Compatible**: Works in GitHub Actions, GitLab CI, Jenkins (different credential methods)

### Configuration Management Benefits

This design achieves the client's goals:

**Single Configuration Point**:
- ✓ All configuration in `config.env`
- ✓ One variable (`STACK_NAME`) controls all resource names
- ✓ No need to edit multiple files

**Easy Customization**:
- ✓ Change `STACK_NAME` → instant rename of entire stack
- ✓ Toggle features with boolean variables
- ✓ Adjust Lambda settings (memory, timeout) in variables.tf

**Multiple Environments**:
- ✓ Different `config.env` per environment (dev, staging, prod)
- ✓ Same Terraform code, different configuration
- ✓ Isolated resource sets (different STACK_NAMEs)

**Template-Friendly**:
- ✓ Users clone and edit one file (`config.env`)
- ✓ Infrastructure adapts automatically
- ✓ Clear separation: config vs infrastructure code

## AWS Resource Configurations

This section defines the specific AWS resource settings that Terraform will provision.

### IAM Roles and Permissions

The starter requires two IAM roles with minimal, scoped permissions following the principle of least privilege.

#### Lambda Execution Role

**Purpose**: Allows Lambda function to execute and interact with AWS services.

**Trust Policy** (who can assume this role):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Required Permissions** (always attached):

**CloudWatch Logs** - Lambda needs to write logs:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:logs:*:*:*"
      }
  ]
}
```

Alternatively, attach AWS managed policy: `arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole`

**Conditional Permissions** (only if Step Functions enabled):

If `enable_step_functions = true`, Lambda needs to start Step Functions executions:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "sfn:StartExecution",
      "Resource": "${step_functions_state_machine_arn}"
    }
  ]
}
```

**Terraform Implementation**:
```hcl
# terraform/lambda.tf

resource "aws_iam_role" "lambda" {
  name = "${var.stack_name}-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "lambda.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_basic" {
  role       = aws_iam_role.lambda.name
  policy_arn = "arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole"
}

resource "aws_iam_policy" "lambda_sfn" {
  count = var.enable_step_functions ? 1 : 0

  name = "${var.stack_name}-lambda-sfn-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "sfn:StartExecution"
      Resource = aws_sfn_state_machine.main[0].arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "lambda_sfn" {
  count = var.enable_step_functions ? 1 : 0

  role       = aws_iam_role.lambda.name
  policy_arn = aws_iam_policy.lambda_sfn[0].arn
}
```

**Why Minimal Permissions?**:
- **Security**: Lambda can't access services it doesn't need
- **Compliance**: Meets least-privilege security requirements
- **Auditability**: Clear what Lambda is authorized to do
- **Template Clarity**: Users see exactly what permissions are granted

#### Step Functions Execution Role

**Purpose**: Allows Step Functions state machine to invoke Lambda function.

**Trust Policy**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "states.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

**Required Permissions**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "lambda:InvokeFunction",
      "Resource": "${lambda_function_arn}"
    }
  ]
}
```

**Terraform Implementation**:
```hcl
# terraform/step_functions.tf

resource "aws_iam_role" "sfn" {
  count = var.enable_step_functions ? 1 : 0

  name = "${var.stack_name}-sfn-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect = "Allow"
      Principal = {
        Service = "states.amazonaws.com"
      }
      Action = "sts:AssumeRole"
    }]
  })
}

resource "aws_iam_policy" "sfn_lambda" {
  count = var.enable_step_functions ? 1 : 0

  name = "${var.stack_name}-sfn-lambda-policy"

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [{
      Effect   = "Allow"
      Action   = "lambda:InvokeFunction"
      Resource = aws_lambda_function.main.arn
    }]
  })
}

resource "aws_iam_role_policy_attachment" "sfn_lambda" {
  count = var.enable_step_functions ? 1 : 0

  role       = aws_iam_role.sfn[0].name
  policy_arn = aws_iam_policy.sfn_lambda[0].arn
}
```

**Why Separate Roles?**:
- **Isolation**: Lambda execution permissions separate from Step Functions orchestration permissions
- **Clarity**: Each service has its own role with clear purpose
- **Security**: Compromise of one role doesn't grant permissions of the other

### Lambda Function Configuration

**Resource Definition**:

```hcl
# terraform/lambda.tf

resource "aws_lambda_function" "main" {
  function_name = "${var.stack_name}-lambda"
  role          = aws_iam_role.lambda.arn

  package_type  = "Image"
  image_uri     = "${aws_ecr_repository.lambda.repository_url}:latest"

  memory_size = var.lambda_memory
  timeout     = var.lambda_timeout

  architectures = ["x86_64"]

  environment {
    variables = {
      # Add environment variables if needed by handlers
      # STACK_NAME = var.stack_name
    }
  }
}
```

**Configuration Parameters**:

**Memory Allocation**:
- Default: 512 MB
- Configurable via `var.lambda_memory`
- Range: 128 MB to 10,240 MB (10 GB)
- **Why 512MB?**: Reasonable default for lightweight handlers, balance between performance and cost

**Timeout**:
- Default: 30 seconds
- Configurable via `var.lambda_timeout`
- Range: 1 second to 900 seconds (15 minutes)
- **Why 30s?**: Sufficient for HTTP responses, Step Functions iterations, custom events

**Package Type**: `Image`
- Uses container image from ECR
- Alternative: `Zip` (for zip file deployments)
- **Why Image?**: Full control over runtime, dependencies, build process (client requirement)

**Image URI Pattern**: `${ecr_repository_url}:latest`
- Points to ECR repository created by Terraform
- Tag: `latest` for simplicity (production might use git SHA)
- Updated when new image pushed to ECR

**Architecture**: `x86_64` (amd64)
- Default: x86-64 processors
- Alternative: `arm64` for AWS Graviton (lower cost, same performance)
- Matches Docker build `GOARCH=amd64`

**Environment Variables**:
- Currently none required for demo handlers
- Users can add: database URLs, API keys, feature flags
- Example: `STACK_NAME` if handlers need to construct resource names

**Terraform Variables**:
```hcl
# terraform/variables.tf

variable "lambda_memory" {
  description = "Lambda function memory allocation in MB"
  type        = number
  default     = 512
}

variable "lambda_timeout" {
  description = "Lambda function timeout in seconds"
  type        = number
  default     = 30
}
```

### ECR Repository Configuration

**Resource Definition**:

```hcl
# terraform/lambda.tf

resource "aws_ecr_repository" "lambda" {
  name                 = "${var.stack_name}-repo"
  image_tag_mutability = "MUTABLE"

  image_scanning_configuration {
    scan_on_push = true
  }
}
```

**Repository Naming**: `${stack_name}-repo`
- Derives from STACK_NAME (e.g., "demo" → "demo-repo")
- Ensures unique repository per deployment

**Image Tag Mutability**: `MUTABLE`
- Allows overwriting tags (e.g., pushing new `latest`)
- Alternative: `IMMUTABLE` (requires unique tag per push)
- **Why Mutable?**: Simplifies development workflow, `latest` tag always points to newest image

**Image Scanning**: `scan_on_push = true`
- AWS scans images for vulnerabilities when pushed
- Provides security reports in ECR console
- **Why Enable?**: Free security scanning, catches known vulnerabilities

**Image Tagging Strategy**:

**Development/Template Default**: Use `latest` tag
- Simple: Always push to same tag
- Deployment references `${ecr_url}:latest`
- Works well for single-environment setups

**Production Alternative**: Use Git commit SHA
- Traceability: Tag = `${git_sha}` (e.g., `a1b2c3d`)
- Immutable: Each commit has unique image
- Rollback capability: Deploy previous commit's image

**Lifecycle Policy (Optional)**:
```hcl
resource "aws_ecr_lifecycle_policy" "lambda" {
  repository = aws_ecr_repository.lambda.name

  policy = jsonencode({
    rules = [{
      rulePriority = 1
      description  = "Keep last 10 images"
      selection = {
        tagStatus   = "any"
        countType   = "imageCountMoreThan"
        countNumber = 10
      }
      action = {
        type = "expire"
      }
    }]
  })
}
```

**Why Lifecycle Policy?**:
- **Cost Control**: Old images consume storage
- **Cleanup**: Automatically remove images after threshold (keep last 10)
- **Optional**: Template can omit this for simplicity

### CloudWatch Logs Configuration

**Resource Definition**:

```hcl
# terraform/lambda.tf

resource "aws_cloudwatch_log_group" "lambda" {
  name              = "/aws/lambda/${var.stack_name}"
  retention_in_days = var.log_retention_days
}
```

**Log Group Naming**: `/aws/lambda/${stack_name}`
- Standard Lambda log group naming convention
- Lambda automatically writes to this log group when it exists
- Derives from STACK_NAME

**Retention Policy**: Default 7 days
- Configurable via `var.log_retention_days`
- Options: 1, 3, 5, 7, 14, 30, 60, 90, 120, 150, 180, 365, 400, 545, 731, 1827, 3653 days
- **Why 7 days?**: Balances debugging needs with cost control
- **Cost Impact**: CloudWatch Logs charges for storage - shorter retention = lower cost

**Automatic Logging**:
- Lambda runtime automatically sends stdout/stderr to CloudWatch
- Go applications use `fmt.Println()`, `log.Println()` → CloudWatch
- IAM role grants Lambda permission to write logs
- No application code changes needed

**Terraform Variable**:
```hcl
# terraform/variables.tf

variable "log_retention_days" {
  description = "CloudWatch log retention period in days"
  type        = number
  default     = 7
}
```

### API Gateway HTTP API Configuration

**Resource Definitions**:

```hcl
# terraform/api_gateway.tf

resource "aws_apigatewayv2_api" "main" {
  count = var.enable_api_gateway ? 1 : 0

  name          = "${var.stack_name}-api"
  protocol_type = "HTTP"
}

resource "aws_apigatewayv2_integration" "lambda" {
  count = var.enable_api_gateway ? 1 : 0

  api_id           = aws_apigatewayv2_api.main[0].id
  integration_type = "AWS_PROXY"
  integration_uri  = aws_lambda_function.main.arn

  payload_format_version = "2.0"
}

resource "aws_apigatewayv2_route" "root" {
  count = var.enable_api_gateway ? 1 : 0

  api_id    = aws_apigatewayv2_api.main[0].id
  route_key = "GET /"

  target = "integrations/${aws_apigatewayv2_integration.lambda[0].id}"
}

resource "aws_apigatewayv2_stage" "default" {
  count = var.enable_api_gateway ? 1 : 0

  api_id      = aws_apigatewayv2_api.main[0].id
  name        = "$default"
  auto_deploy = true
}

resource "aws_lambda_permission" "api_gateway" {
  count = var.enable_api_gateway ? 1 : 0

  statement_id  = "AllowAPIGatewayInvoke"
  action        = "lambda:InvokeFunction"
  function_name = aws_lambda_function.main.function_name
  principal     = "apigateway.amazonaws.com"
  source_arn    = "${aws_apigatewayv2_api.main[0].execution_arn}/*/*"
}
```

**API Type Choice**: **HTTP API** (not REST API)

**Why HTTP API?**:
- **Simpler**: Fewer configuration options, easier setup
- **Cheaper**: 70% lower cost than REST API for same traffic
- **Sufficient**: Provides all features needed for Lambda backends (routing, proxy integration)
- **Modern**: Latest API Gateway offering, recommended for new projects

**What HTTP API Lacks** (compared to REST API):
- Request/response transformation
- API keys and usage plans
- WAF integration
- Request validation

**Why These Don't Matter**: Template is a simple starter - users needing advanced features can migrate to REST API later

**Integration Type**: `AWS_PROXY`

**What AWS_PROXY Does**:
- Passes entire HTTP request to Lambda as-is
- Lambda receives: method, path, headers, query params, body
- Lambda returns: status code, headers, body
- No transformation or mapping required

**Payload Format Version**: `2.0`
- Latest API Gateway payload format
- Simpler structure than v1.0
- Event includes: `requestContext`, `httpMethod`, `path`, `headers`, `body`

**Route Configuration**: `GET /`
- Demonstration route showing HTTP method + path pattern
- Users can add more routes: `POST /users`, `GET /items/{id}`, etc.
- Wildcard route possible: `ANY /{proxy+}` (catches all paths)

**Stage**: `$default`
- Auto-deploy: Changes automatically deployed
- No manual deployment step required
- Production setups might use named stages: `dev`, `staging`, `prod`

**Lambda Permission**:
- Grants `apigateway.amazonaws.com` permission to invoke Lambda
- Scoped to this specific API Gateway (`source_arn`)
- Without this: API Gateway gets 403 Forbidden when invoking Lambda

**API URL Output**:
```hcl
# terraform/outputs.tf

output "api_gateway_url" {
  value = var.enable_api_gateway ? aws_apigatewayv2_stage.default[0].invoke_url : null
}
```

Format: `https://{api-id}.execute-api.{region}.amazonaws.com`

### Step Functions State Machine Design

**State Machine Definition**:

The state machine implements the iteration logic specified in client needs: maintain counter state, increment per invocation, complete after N iterations.

**State Machine Definition (JSON)**:
```json
{
  "Comment": "Iterative counter demonstration",
  "StartAt": "InvokeLambda",
  "States": {
    "InvokeLambda": {
      "Type": "Task",
      "Resource": "${lambda_function_arn}",
      "ResultPath": "$.result",
      "Next": "CheckComplete"
    },
    "CheckComplete": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.result.complete",
          "BooleanEquals": true,
          "Next": "Complete"
        }
      ],
      "Default": "Wait"
    },
    "Wait": {
      "Type": "Wait",
      "SecondsPath": "$.delay_seconds",
      "Next": "InvokeLambda"
    },
    "Complete": {
      "Type": "Succeed"
    }
  }
}
```

**State Flow Explanation**:

1. **InvokeLambda (Task State)**:
   - Invokes Lambda function with current state
   - Input: `{start: 0, iterations: 5, delay_ms: 1000, counter: N}`
   - Lambda increments counter, returns: `{counter: N+1, iterations_remaining: M, complete: bool}`
   - Result stored in `$.result` path (merges with input)

2. **CheckComplete (Choice State)**:
   - Examines `$.result.complete` boolean
   - If `true`: Transitions to Complete (success)
   - If `false`: Transitions to Wait (continue iterating)

3. **Wait (Wait State)**:
   - Pauses for duration specified in `$.delay_seconds`
   - Allows rate-limited processing (e.g., wait 1 second between iterations)
   - After wait: Loops back to InvokeLambda

4. **Complete (Succeed State)**:
   - Terminal state indicating successful completion
   - Execution status: SUCCEEDED
   - Final output: Last Lambda result

**State Maintenance**:
- State machine automatically maintains state across Lambda invocations
- Each Lambda call receives current state (including counter value)
- Lambda returns updated state (incremented counter)
- State machine merges result and passes to next invocation

**Error Handling**:
```json
{
  "InvokeLambda": {
    "Type": "Task",
    "Resource": "${lambda_function_arn}",
    "ResultPath": "$.result",
    "Retry": [
      {
        "ErrorEquals": ["Lambda.ServiceException", "Lambda.TooManyRequestsException"],
        "IntervalSeconds": 2,
        "MaxAttempts": 3,
        "BackoffRate": 2.0
      }
    ],
    "Catch": [
      {
        "ErrorEquals": ["States.ALL"],
        "Next": "Failed"
      }
    ],
    "Next": "CheckComplete"
  },
  "Failed": {
    "Type": "Fail",
    "Error": "LambdaInvocationFailed",
    "Cause": "Lambda function invocation failed after retries"
  }
}
```

**Retry Logic**:
- Retries on transient Lambda errors (ServiceException, TooManyRequestsException)
- 3 attempts with exponential backoff (2s, 4s, 8s)
- Handles temporary Lambda throttling or service issues

**Catch Block**:
- If retries exhausted or other error occurs, transition to Failed state
- Execution status: FAILED
- Provides error details for debugging

**Terraform Resource**:
```hcl
# terraform/step_functions.tf

resource "aws_sfn_state_machine" "main" {
  count = var.enable_step_functions ? 1 : 0

  name     = "${var.stack_name}-state-machine"
  role_arn = aws_iam_role.sfn[0].arn

  definition = templatefile("${path.module}/state_machine.json", {
    lambda_function_arn = aws_lambda_function.main.arn
  })
}
```

**Why This Design?**:
- **Demonstrates Workflows**: Shows how Step Functions orchestrates Lambda for multi-step processes
- **State Management**: Illustrates stateful processing across invocations
- **Production Pattern**: Real workflows use this pattern (batch processing, polling, retry logic)
- **Easy Customization**: Users replace iteration logic with domain-specific workflow steps

## Testing Strategy

The starter includes comprehensive testing at both unit and end-to-end levels, ensuring handlers work in isolation and integrate correctly with AWS services.

### Unit Testing

**Approach**: Test each handler independently using mock events, no AWS dependencies required.

**Testing Framework**: Go's built-in `testing` package
- Standard library, no external dependencies
- Simple test functions: `func TestXxx(t *testing.T)`
- Built-in assertion via `t.Error()`, `t.Fatal()`

**Test Location**: `tests/unit/`

**Unit Testing Each Handler**:

**API Gateway Handler Tests**:
```go
// tests/unit/apigateway_test.go
package unit

import (
    "testing"
    "github.com/aws/aws-lambda-go/events"
    "github.com/prethora/starter_golang_aws_lambda/internal/handlers"
)

func TestAPIGatewayHandler(t *testing.T) {
    // Create mock API Gateway event
    event := events.APIGatewayProxyRequest{
        HTTPMethod: "GET",
        Path:       "/",
    }

    // Call handler
    response, err := handlers.HandleAPIGateway(event)

    // Verify results
    if err != nil {
        t.Fatalf("Handler returned error: %v", err)
    }
    if response.StatusCode != 200 {
        t.Errorf("Expected status 200, got %d", response.StatusCode)
    }
    // Verify JSON body contains expected fields
}
```

**Custom Event Handler Tests**:
```go
// tests/unit/customevent_test.go
func TestCustomEventHandler(t *testing.T) {
    // Create mock custom event
    event := map[string]interface{}{
        "foo": "bar",
        "baz": 123,
    }

    // Call handler
    response, err := handlers.HandleCustomEvent(event)

    // Verify echo response wraps input correctly
    if response.Source != "custom-event" {
        t.Error("Expected source: custom-event")
    }
    // Verify response.Echo matches input event
}
```

**Step Functions Handler Tests**:
```go
// tests/unit/stepfunction_test.go
func TestStepFunctionHandler(t *testing.T) {
    // Create mock Step Functions input
    input := map[string]interface{}{
        "start":      0,
        "iterations": 3,
        "counter":    1,
    }

    // Call handler
    response, err := handlers.HandleStepFunction(input)

    // Verify counter incremented
    if response.Counter != 2 {
        t.Errorf("Expected counter 2, got %d", response.Counter)
    }
    // Verify iterations_remaining calculated correctly
    // Verify complete flag set appropriately
}
```

**Test Coverage Expectations**:
- Each handler has unit tests
- Tests cover happy path (valid events)
- Tests cover error cases (invalid events, missing fields)
- No AWS SDK calls in unit tests (pure logic testing)
- Tests run quickly (milliseconds) with no external dependencies

**Running Unit Tests**:
```bash
go test ./tests/unit/...
```

### End-to-End Testing Workflow

**Purpose**: Verify complete integration with real AWS services.

**E2E Test Flow**:
1. Deploy test stack with unique name
2. Test API Gateway pattern
3. Test Custom Event pattern
4. Test Step Functions pattern
5. Destroy test stack (cleanup)

**Test Stack Naming**: `test-${timestamp}`
- Timestamp: Unix timestamp or ISO 8601 format
- Example: `test-1704067200` or `test-2026-01-14-10-30`
- Prevents conflicts with production and other test runs

**What E2E Tests Verify**:

**API Gateway Test**:
```bash
# Get API Gateway URL from Terraform outputs
API_URL=$(terraform output -raw api_gateway_url)

# HTTP GET request
RESPONSE=$(curl -s "$API_URL/")

# Verify HTTP 200
# Verify JSON body: {"message": "Hello World", "source": "api-gateway"}
```

**Custom Event Test**:
```bash
# Invoke Lambda directly with test payload
aws lambda invoke \
  --function-name test-1704067200-lambda \
  --payload '{"test": "data"}' \
  --cli-binary-format raw-in-base64-out \
  response.json

# Verify response: {"source": "custom-event", "echo": {"test": "data"}}
```

**Step Functions Test**:
```bash
# Start execution
EXECUTION_ARN=$(aws stepfunctions start-execution \
  --state-machine-arn ${STATE_MACHINE_ARN} \
  --input '{"start": 0, "iterations": 3, "delay_ms": 500}' \
  --query 'executionArn' \
  --output text)

# Poll until complete
while true; do
  STATUS=$(aws stepfunctions describe-execution \
    --execution-arn $EXECUTION_ARN \
    --query 'status' \
    --output text)

  if [ "$STATUS" = "SUCCEEDED" ]; then
    break
  fi
  sleep 1
done

# Verify output: counter = 3, complete = true
```

**Why Deploy Temporary Stack?**:
- **Real Integration**: Tests actual AWS Lambda, API Gateway, Step Functions
- **Isolation**: Doesn't affect production deployments
- **Cleanup**: Automatically destroyed after tests
- **CI/CD Ready**: Can run in automated pipelines

**Test Script Location**: `tests/e2e/` with Go test files using AWS SDK

## Deployment Automation

The starter provides shell scripts for common operations: deploy, destroy, and end-to-end testing.

### Deployment Script (scripts/deploy.sh)

**Responsibilities**:

1. **Load Configuration**:
```bash
source config.env

if [ -z "$STACK_NAME" ]; then
    echo "ERROR: STACK_NAME not set in config.env"
    exit 1
fi
```

2. **Build Docker Image**:
```bash
echo "Building Docker image..."
docker build -t ${STACK_NAME}:latest .
```

3. **ECR Authentication**:
```bash
echo "Authenticating to ECR..."
AWS_REGION=${AWS_REGION:-us-east-1}
aws ecr get-login-password --region $AWS_REGION | \
  docker login --username AWS --password-stdin \
  ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com
```

4. **Tag and Push Image**:
```bash
# Get ECR repository URL from Terraform or construct it
ECR_REPO="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${STACK_NAME}-repo"

docker tag ${STACK_NAME}:latest ${ECR_REPO}:latest
docker push ${ECR_REPO}:latest
```

5. **Deploy Infrastructure**:
```bash
echo "Deploying infrastructure..."
cd terraform
terraform init
terraform apply \
  -var="stack_name=$STACK_NAME" \
  -var="aws_region=$AWS_REGION" \
  -auto-approve
cd ..
```

6. **Display Outputs**:
```bash
echo "Deployment complete!"
echo "API Gateway URL: $(terraform -chdir=terraform output -raw api_gateway_url)"
echo "Lambda Function: $(terraform -chdir=terraform output -raw lambda_function_name)"
```

**Script Flow**: Configuration → Build → Push → Deploy → Outputs

### Destroy Script (scripts/destroy.sh)

**Responsibilities**:

1. **Load Configuration**:
```bash
source config.env
```

2. **Confirmation Prompt**:
```bash
echo "WARNING: This will destroy all resources for stack: $STACK_NAME"
read -p "Are you sure? (yes/no): " CONFIRM

if [ "$CONFIRM" != "yes" ]; then
    echo "Aborted."
    exit 0
fi
```

3. **Destroy Infrastructure**:
```bash
echo "Destroying infrastructure..."
cd terraform
terraform destroy \
  -var="stack_name=$STACK_NAME" \
  -var="aws_region=${AWS_REGION:-us-east-1}" \
  -auto-approve
cd ..
```

4. **ECR Images Note**:
```bash
echo "Infrastructure destroyed."
echo "Note: ECR images were not deleted."
echo "To remove images manually:"
echo "  aws ecr delete-repository --repository-name ${STACK_NAME}-repo --force"
```

**Why Keep ECR Images?**:
- User decision whether to delete
- Rollback capability (keep old images)
- Cost is minimal for a few images

### E2E Test Script (scripts/test-e2e.sh)

**Responsibilities**:

1. **Generate Test Stack Name**:
```bash
TIMESTAMP=$(date +%s)
TEST_STACK="test-${TIMESTAMP}"
echo "Using test stack: $TEST_STACK"
```

2. **Deploy Test Stack**:
```bash
# Temporarily override STACK_NAME
export STACK_NAME=$TEST_STACK
./scripts/deploy.sh
```

3. **Run Tests** (using Go tests with AWS SDK):
```bash
echo "Running E2E tests..."
cd tests/e2e
go test -v -timeout 10m
TEST_RESULT=$?
cd ../..
```

4. **Cleanup (Always)**:
```bash
# Destroy test stack even if tests failed
echo "Cleaning up test stack..."
export STACK_NAME=$TEST_STACK
./scripts/destroy.sh

exit $TEST_RESULT
```

**Test Isolation Strategy**:
- Unique stack name per test run
- No interference with production
- CI/CD can run multiple test jobs in parallel (different timestamps)

## Error Handling

### Handler Error Taxonomy

**Defined Error Types**:

```go
// internal/handlers/errors.go

var (
    // ErrInvalidEvent indicates the event structure couldn't be parsed
    ErrInvalidEvent = errors.New("handler: unable to parse event")

    // ErrUnsupportedRoute indicates an API Gateway route is not implemented
    ErrUnsupportedRoute = errors.New("handler: unsupported API route")

    // ErrIterationFailed indicates Step Functions iteration logic failed
    ErrIterationFailed = errors.New("handler: step function iteration failed")
)
```

**Error Propagation**:

**API Gateway Handler**:
- Handler returns error → Lambda runtime converts to HTTP 500 response
- Alternative: Handler catches error, returns APIGatewayProxyResponse with 400/500 status

**Custom Event Handler**:
- Handler returns error → Lambda runtime wraps in error response
- Alternative: Handler returns success response with error field

**Step Functions Handler**:
- Handler returns error → Step Functions marks task as failed
- Retry/catch logic in state machine handles failures

**Error Response Formats**:

**API Gateway Error Response**:
```json
{
  "statusCode": 500,
  "headers": {"Content-Type": "application/json"},
  "body": "{\"error\": \"Internal server error\"}"
}
```

**Custom Event Error** (if handler catches):
```json
{
  "source": "custom-event",
  "error": "Unable to process event",
  "details": "Invalid JSON structure"
}
```

### HTTP Status Code Mapping for API Gateway Errors

The API Gateway handler must translate application errors into appropriate HTTP status codes. Rather than allowing the Lambda runtime to return generic 500 errors, the handler catches errors and returns properly formatted responses.

**Error to Status Code Mapping**:

| Error Type | HTTP Status | Reason |
|------------|-------------|--------|
| `ErrInvalidEvent` | 400 Bad Request | Client sent malformed event structure |
| `ErrUnsupportedRoute` | 404 Not Found | API route not implemented |
| `ErrIterationFailed` | 500 Internal Server Error | Step Functions logic error (shouldn't occur in API Gateway) |
| Unexpected errors | 500 Internal Server Error | Unhandled server-side failures |

**Implementation Pattern**:

```go
// internal/handlers/apigateway.go

func HandleAPIGateway(event events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Route handling logic
    response, err := processRoute(event.HTTPMethod, event.Path, event.Body)

    if err != nil {
        return errorResponse(err), nil  // Return error as HTTP response, not Go error
    }

    return response, nil
}

// errorResponse translates errors to appropriate HTTP status codes
func errorResponse(err error) events.APIGatewayProxyResponse {
    var statusCode int
    var errorMessage string

    switch {
    case errors.Is(err, ErrInvalidEvent):
        statusCode = 400
        errorMessage = "Invalid request format"
    case errors.Is(err, ErrUnsupportedRoute):
        statusCode = 404
        errorMessage = "Route not found"
    default:
        statusCode = 500
        errorMessage = "Internal server error"
    }

    return events.APIGatewayProxyResponse{
        StatusCode: statusCode,
        Headers: map[string]string{
            "Content-Type": "application/json",
        },
        Body: fmt.Sprintf(`{"error": "%s"}`, errorMessage),
    }
}
```

**Why Explicit Status Codes?**:
1. **Client Understanding**: 400 vs 500 tells clients whether to retry or fix their request
2. **API Standards**: RESTful APIs use semantic HTTP status codes
3. **Debugging**: Specific codes help identify error categories quickly
4. **Monitoring**: Status code metrics distinguish client errors from server errors

**Example Error Scenarios**:

**400 Bad Request** (ErrInvalidEvent):
```bash
# Client sends invalid JSON
curl -X POST https://api.example.com/ -d "invalid-json"

# Response:
HTTP/1.1 400 Bad Request
{"error": "Invalid request format"}
```

**404 Not Found** (ErrUnsupportedRoute):
```bash
# Client requests unimplemented route
curl https://api.example.com/nonexistent

# Response:
HTTP/1.1 404 Not Found
{"error": "Route not found"}
```

**500 Internal Server Error** (Unexpected):
```bash
# Server encounters unexpected error
curl https://api.example.com/

# Response:
HTTP/1.1 500 Internal Server Error
{"error": "Internal server error"}
```

**Error Handling Strategy**:
- **Never return Go error from handler** - Lambda runtime converts all errors to HTTP 502
- **Always return APIGatewayProxyResponse** - Even for error cases
- **Log detailed errors** - Use `fmt.Println()` or structured logging for CloudWatch
- **Return generic messages to clients** - Don't expose internal error details

### Step Functions Handler Input Parameter Validation

The Step Functions handler must validate input parameters and apply defaults for optional fields to ensure robust execution.

**Parameter Requirements**:

| Parameter | Type | Required | Default | Validation |
|-----------|------|----------|---------|------------|
| `start` | int | Yes | N/A | Must be >= 0 |
| `iterations` | int | Yes | N/A | Must be > 0 |
| `counter` | int | No | `start` value | Must be >= 0 |
| `delay_ms` | int | No | 1000 | Must be >= 0 |

**Validation Logic**:

```go
// internal/handlers/stepfunction.go

type StepFunctionInput struct {
    Start      int `json:"start"`
    Iterations int `json:"iterations"`
    Counter    int `json:"counter"`
    DelayMs    int `json:"delay_ms"`
}

type StepFunctionOutput struct {
    Counter             int  `json:"counter"`
    IterationsRemaining int  `json:"iterations_remaining"`
    Complete            bool `json:"complete"`
}

func HandleStepFunction(input StepFunctionInput) (StepFunctionOutput, error) {
    // Apply defaults for optional parameters
    if input.DelayMs == 0 {
        input.DelayMs = 1000  // Default: 1 second delay
    }

    if input.Counter == 0 {
        input.Counter = input.Start  // Initialize counter to start value
    }

    // Validate required parameters
    if input.Iterations <= 0 {
        return StepFunctionOutput{}, fmt.Errorf("%w: iterations must be > 0, got %d",
            ErrIterationFailed, input.Iterations)
    }

    if input.Start < 0 {
        return StepFunctionOutput{}, fmt.Errorf("%w: start must be >= 0, got %d",
            ErrIterationFailed, input.Start)
    }

    // Validate bounds after defaults applied
    if input.Counter < 0 {
        return StepFunctionOutput{}, fmt.Errorf("%w: counter must be >= 0, got %d",
            ErrIterationFailed, input.Counter)
    }

    if input.DelayMs < 0 {
        return StepFunctionOutput{}, fmt.Errorf("%w: delay_ms must be >= 0, got %d",
            ErrIterationFailed, input.DelayMs)
    }

    // Process iteration
    newCounter := input.Counter + 1
    remaining := input.Iterations - newCounter
    complete := remaining <= 0

    return StepFunctionOutput{
        Counter:             newCounter,
        IterationsRemaining: remaining,
        Complete:            complete,
    }, nil
}
```

**Default Value Application Strategy**:

Go's zero-value behavior is leveraged for optional parameters:
- `DelayMs == 0` → Apply default 1000ms
- `Counter == 0` → Apply default from `Start` value

**Why Check for Zero**:
- JSON unmarshaling sets missing fields to zero values
- Zero is valid for some parameters (`Start` can be 0) but indicates "not provided" for others
- Explicit checks distinguish "not provided" from "provided as 0"

**Validation Error Responses**:

```json
// Invalid iterations (must be > 0)
{
  "errorMessage": "handler: step function iteration failed: iterations must be > 0, got -5",
  "errorType": "ErrIterationFailed"
}
```

```json
// Invalid start (must be >= 0)
{
  "errorMessage": "handler: step function iteration failed: start must be >= 0, got -1",
  "errorType": "ErrIterationFailed"
}
```

**Step Functions Error Handling**:

When validation fails:
1. Handler returns Go error
2. Step Functions marks task as FAILED
3. State machine Retry logic may retry (if configured)
4. State machine Catch logic handles persistent failures
5. Error details appear in execution history

**Example State Machine Behavior**:

**Valid Input**:
```json
// Input
{"start": 0, "iterations": 3}

// First invocation (counter defaults to start)
{"counter": 1, "iterations_remaining": 2, "complete": false}

// Second invocation
{"counter": 2, "iterations_remaining": 1, "complete": false}

// Third invocation
{"counter": 3, "iterations_remaining": 0, "complete": true}
```

**Invalid Input**:
```json
// Input with invalid iterations
{"start": 0, "iterations": -1}

// Handler returns error
// Step Functions execution FAILS
// Error: "iterations must be > 0, got -1"
```

**With Default Application**:
```json
// Input missing delay_ms
{"start": 0, "iterations": 3}

// Handler applies default delay_ms = 1000
// Continues normal execution
```

**Parameter Validation Benefits**:
1. **Early Failure**: Invalid inputs fail immediately, not after partial execution
2. **Clear Errors**: Descriptive messages explain what's wrong and what values are acceptable
3. **State Machine Reliability**: Prevents state machines from getting stuck with bad data
4. **Default Convenience**: Common cases work without specifying all parameters

### Deployment Error Handling

**Common Deployment Errors**:

**Missing AWS Credentials**:
```
Error: UnauthorizedOperation: You are not authorized to perform this operation
```
**Solution**: Set AWS_ACCESS_KEY_ID and AWS_SECRET_ACCESS_KEY

**Missing STACK_NAME**:
```
ERROR: STACK_NAME not set in config.env
```
**Solution**: Edit config.env and set STACK_NAME

**Docker Build Failure**:
```
Error building image: go build failed
```
**Solution**: Check Go code compiles, verify go.mod dependencies

**Terraform State Conflict**:
```
Error: Error acquiring the state lock
```
**Solution**: Wait for other Terraform process to finish, or force-unlock if stuck

**Script Validation**:
```bash
# deploy.sh validates before proceeding
if [ -z "$STACK_NAME" ]; then
    echo "ERROR: STACK_NAME must be set"
    exit 1
fi

if ! command -v docker &> /dev/null; then
    echo "ERROR: Docker is not installed"
    exit 1
fi

if ! command -v terraform &> /dev/null; then
    echo "ERROR: Terraform is not installed"
    exit 1
fi
```

### Handler Response Format Differences

**API Gateway**: `events.APIGatewayProxyResponse`
```go
return events.APIGatewayProxyResponse{
    StatusCode: 200,
    Headers: map[string]string{
        "Content-Type": "application/json",
    },
    Body: `{"message": "Hello World", "source": "api-gateway"}`,
}, nil
```

**Custom Events**: Arbitrary structure
```go
return map[string]interface{}{
    "source": "custom-event",
    "echo":   event,
}, nil
```

**Step Functions**: Specific fields
```go
return map[string]interface{}{
    "counter":              counter + 1,
    "iterations_remaining": iterations - (counter + 1),
    "complete":             (counter + 1) >= iterations,
}, nil
```

**Lambda Runtime Handling**:
- Automatically serializes return value to JSON
- Different event sources expect different response formats
- Handler must return appropriate structure for its event source

## Documentation Requirements

### README Structure

The README provides complete onboarding for new users, from cloning to customization.

**Sections**:

**1. Quick Start** (5 minutes to deploy):
```markdown
## Quick Start

1. Clone repository
2. Edit `config.env`: Set `STACK_NAME=my-service`
3. Set AWS credentials: `export AWS_ACCESS_KEY_ID=...`
4. Run: `./scripts/deploy.sh`
5. Test: `curl $(terraform -chdir=terraform output -raw api_gateway_url)`
```

**2. Prerequisites**:
- Go 1.23 or later
- Docker (latest stable)
- Terraform 1.6 or later
- AWS CLI v2
- AWS account with IAM permissions (Lambda, API Gateway, Step Functions, ECR, IAM, CloudWatch)

**3. Configuration**:
- How to edit config.env
- AWS credentials setup (environment variables, ~/.aws/credentials, IAM roles)
- Optional: Feature toggles (enable_api_gateway, enable_step_functions)

**4. Deployment**:
- Step-by-step walkthrough
- Expected terminal output
- How to verify deployment succeeded

**5. Testing**:
- Running unit tests: `go test ./tests/unit/...`
- Running E2E tests: `./scripts/test-e2e.sh`
- Manual testing of each invocation pattern

**6. Invocation Patterns**:
Example requests and responses for each pattern:
- API Gateway: `curl` examples
- Custom Events: `aws lambda invoke` examples
- Step Functions: `aws stepfunctions start-execution` examples

**7. Disabling Features**:
```hcl
# terraform/terraform.tfvars
enable_api_gateway = false  # Disable API Gateway
```

**8. Customization**:
- Where to add business logic (handlers)
- How to add environment variables
- How to add new API routes
- How to modify Step Functions workflow

**9. Cleanup**:
```bash
./scripts/destroy.sh
```

**10. Troubleshooting**:
- Common errors and solutions
- How to view Lambda logs: `aws logs tail /aws/lambda/my-service --follow`
- How to debug failed Step Functions executions

### Code Documentation Standards

**Package-Level Documentation**:
```go
// Package handlers implements Lambda event handlers for API Gateway,
// custom events, and Step Functions.
//
// Each handler is responsible for processing a specific event type
// and returning an appropriately formatted response.
package handlers
```

**Function Documentation**:
```go
// HandleAPIGateway processes HTTP requests from API Gateway.
//
// It expects an APIGatewayProxyRequest event containing HTTP method,
// path, headers, and optional body. Returns an APIGatewayProxyResponse
// with status code, headers, and JSON body.
//
// Example:
//   event := events.APIGatewayProxyRequest{HTTPMethod: "GET", Path: "/"}
//   response, err := HandleAPIGateway(event)
//
// Returns error if event cannot be processed.
func HandleAPIGateway(event events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Implementation
}
```

**Inline Comments for Non-Obvious Logic**:
```go
// Event detection priority: API Gateway first (most specific),
// then Step Functions (has required fields), finally custom events (fallback)
func detectEventType(event json.RawMessage) EventType {
    // Parse event to inspect structure
    var eventMap map[string]interface{}
    json.Unmarshal(event, &eventMap)

    // Check for API Gateway-specific fields
    if _, hasRequestContext := eventMap["requestContext"]; hasRequestContext {
        return EventTypeAPIGateway
    }

    // Check for Step Functions-specific fields
    if _, hasIterations := eventMap["iterations"]; hasIterations {
        return EventTypeStepFunctions
    }

    // Default to custom event
    return EventTypeCustomEvent
}
```

**Why Documentation Matters**:
- **Template Users**: Need to understand where to customize
- **Maintenance**: Future developers (including original author) need context
- **Learning**: Demonstrates best practices for Lambda handlers

## Design Complete

This high-level design document provides comprehensive guidance for implementing the AWS Lambda Go Starter Package. It covers:

- ✓ Project structure and package organization
- ✓ Event routing and handler architecture
- ✓ External dependencies and version management
- ✓ Docker containerization with multi-stage builds
- ✓ Terraform infrastructure-as-code organization
- ✓ Single configuration point (config.env) strategy
- ✓ IAM roles with minimal permissions
- ✓ Lambda, API Gateway, Step Functions, ECR, and CloudWatch configurations
- ✓ Testing strategy (unit and end-to-end)
- ✓ Deployment automation scripts
- ✓ Error handling and taxonomy
- ✓ Documentation requirements (README and code comments)

The design achieves the client's goals:
- **Production-ready**: Comprehensive testing, proper IAM, error handling
- **Template-friendly**: Single config file, easy customization, clear structure
- **Multi-pattern support**: Three Lambda invocation patterns in one function
- **Latest versions**: Go 1.23+, current AWS SDK, modern base images
- **Cross-platform**: Docker handles compilation for any development environment
- **Infrastructure-as-code**: Terraform manages all AWS resources
- **Automated deployment**: Scripts handle build, push, deploy workflow

The next phase (architecture design) will translate these high-level decisions into detailed package structures, type definitions, function signatures, and implementation specifics.
