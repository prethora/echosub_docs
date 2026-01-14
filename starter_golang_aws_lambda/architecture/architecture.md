# Architecture: AWS Lambda Go Starter Package

## 1. Package Overview

This document defines the architecture for `prethora/starter_golang_aws_lambda`, a production-ready template for building AWS Lambda functions with Go and Docker.

### 1.1 Project Structure

```
starter_golang_aws_lambda/
├── cmd/
│   └── lambda/
│       └── main.go              # Lambda entry point and initialization
├── internal/
│   ├── handlers/
│   │   ├── errors.go            # Sentinel errors
│   │   ├── constants.go         # Constants and defaults
│   │   ├── apigateway.go        # API Gateway event handler
│   │   ├── customevent.go       # Custom event handler
│   │   └── stepfunction.go      # Step Function handler
│   └── router/
│       ├── types.go             # EventType enum
│       └── router.go            # Event type detection and routing
├── tests/
│   ├── unit/                    # Unit tests for handlers
│   └── e2e/                     # End-to-end integration tests
├── terraform/
│   ├── main.tf                  # Main Terraform configuration
│   ├── variables.tf             # Variable definitions
│   ├── outputs.tf               # Output values
│   ├── api_gateway.tf           # HTTP API Gateway resources
│   ├── step_functions.tf        # Step Functions state machine
│   └── lambda.tf                # Lambda function and ECR resources
└── scripts/
    ├── deploy.sh                # Deployment automation
    ├── destroy.sh               # Teardown script
    └── test-e2e.sh              # End-to-end test runner
```

### 1.2 Build Requirements

- **Go Version**: 1.23 or higher
- **Target Runtime**: AWS Lambda (Docker container runtime)
- **Base Image**: `public.ecr.aws/lambda/provided:al2023`
- **Binary Output**: Single statically-linked binary named `bootstrap`

### 1.3 Go Module Definition (`go.mod`)

The project uses Go modules for dependency management. The `go.mod` file defines the module path, Go version requirement, and runtime dependencies:

```go
module github.com/prethora/starter_golang_aws_lambda

go 1.23

require (
	github.com/aws/aws-lambda-go v1.51.2
)
```

**Module Path:** `github.com/prethora/starter_golang_aws_lambda`

**Runtime Dependencies:**
- `github.com/aws/aws-lambda-go v1.51.2` — AWS Lambda runtime for Go

**Test Dependencies:**

The E2E tests require additional dependencies that are **not** included in the Lambda binary. These will be added automatically when running `go mod tidy` after implementing E2E tests:

```go
// Test-only dependencies (added by go mod tidy)
require (
	github.com/aws/aws-sdk-go-v2 v1.41.1
	github.com/aws/aws-sdk-go-v2/config v1.28.6
	github.com/aws/aws-sdk-go-v2/service/lambda v1.68.4
	github.com/aws/aws-sdk-go-v2/service/sfn v1.34.7
)
```

**Important:** The Lambda function binary only depends on `aws-lambda-go`. The AWS SDK v2 packages are only imported in the `tests/e2e` package and do not increase the deployed binary size.

## 2. Shared Types and Constants

### 2.1 Errors (`internal/handlers/errors.go`)

```go
// Package handlers provides Lambda event handlers for API Gateway,
// custom events, and Step Functions state machine invocations.
package handlers

import "errors"

// Sentinel errors for Lambda handler operations.
// Use errors.Is() to check for specific error conditions.
var (
	// ErrInvalidEvent is returned when the handler is unable to parse
	// the incoming event payload into the expected structure.
	ErrInvalidEvent = errors.New("handler: unable to parse event")

	// ErrUnsupportedRoute is returned when an API Gateway request
	// targets a route that is not implemented by the handler.
	ErrUnsupportedRoute = errors.New("handler: unsupported API route")

	// ErrIterationFailed is returned when a Step Function iteration
	// encounters an error during counter state management.
	ErrIterationFailed = errors.New("handler: step function iteration failed")
)
```

### 2.2 Constants (`internal/handlers/constants.go`)

```go
package handlers

const (
	// DefaultAWSRegion is the AWS region used when not specified
	// in environment variables.
	DefaultAWSRegion = "us-east-1"

	// DefaultIterations is the default number of iterations for
	// Step Function state machine executions.
	DefaultIterations = 5

	// DefaultDelayMs is the default delay in milliseconds between
	// Step Function iterations.
	DefaultDelayMs = 1000
)
```

### 2.3 Event Types (`internal/router/types.go`)

```go
// Package router provides event type detection and routing logic
// for AWS Lambda invocations, enabling a single Lambda function
// to handle multiple event sources (API Gateway, Step Functions, custom events).
package router

// EventType identifies the source of a Lambda invocation event.
type EventType string

const (
	// EventTypeAPIGateway indicates an HTTP request from API Gateway.
	EventTypeAPIGateway EventType = "api-gateway"

	// EventTypeStepFunction indicates an invocation from AWS Step Functions.
	EventTypeStepFunction EventType = "step-function"

	// EventTypeCustomEvent indicates a direct Lambda invocation
	// with a custom JSON payload.
	EventTypeCustomEvent EventType = "custom-event"
)
```

### 2.4 Dependencies and Import Requirements

**Required Dependencies:**
- `github.com/aws/aws-lambda-go v1.51.2` — Lambda runtime and event types

**Standard Library Imports:**
- `context` — For context propagation through handlers
- `encoding/json` — For JSON marshaling/unmarshaling
- `errors` — For sentinel error definitions

**Import Boundary Rule:**

AWS SDK v2 imports (`github.com/aws/aws-sdk-go-v2/*`) are **only permitted** in the `tests/e2e` package. They must **never** be imported in `cmd/` or `internal/` packages. The Lambda function itself only depends on the Lambda runtime SDK, keeping the binary size minimal.

## 3. Handler Request/Response Types

### 3.1 API Gateway Types (`internal/handlers/apigateway.go`)

```go
package handlers

import (
	"encoding/json"

	"github.com/aws/aws-lambda-go/events"
)

// APIGatewayRequest represents an HTTP request from API Gateway.
// This is an alias to the AWS SDK's API Gateway proxy request type.
type APIGatewayRequest = events.APIGatewayProxyRequest

// APIGatewayResponse represents an HTTP response to API Gateway.
// This is an alias to the AWS SDK's API Gateway proxy response type.
type APIGatewayResponse = events.APIGatewayProxyResponse

// APIGatewaySuccessResponse is the JSON structure returned by the
// API Gateway handler for successful requests.
type APIGatewaySuccessResponse struct {
	// Message contains the response message content.
	Message string `json:"message"`

	// Source identifies the handler that generated the response.
	Source string `json:"source"`
}
```

### 3.2 Custom Event Types (`internal/handlers/customevent.go`)

```go
package handlers

import "encoding/json"

// CustomEventRequest represents an arbitrary JSON payload from a direct
// Lambda invocation. The handler receives json.RawMessage and echoes it back.
//
// Note: There is no fixed struct type for custom events since they can
// contain any valid JSON. Handlers work with json.RawMessage directly.
type CustomEventRequest = json.RawMessage

// CustomEventResponse is the JSON structure returned by the custom event
// handler, wrapping the original event payload.
type CustomEventResponse struct {
	// Source identifies the handler that generated the response.
	Source string `json:"source"`

	// Echo contains the original event payload that was received.
	Echo json.RawMessage `json:"echo"`
}
```

### 3.3 Step Function Types (`internal/handlers/stepfunction.go`)

```go
package handlers

// StepFunctionInput represents the input payload for a Step Functions
// state machine task invocation. It manages iteration state across
// multiple Lambda invocations.
type StepFunctionInput struct {
	// Start is the initial counter value (default: 0).
	Start int `json:"start"`

	// Iterations is the total number of iterations to perform (default: 5).
	Iterations int `json:"iterations"`

	// DelayMs is the delay in milliseconds between iterations (default: 1000).
	DelayMs int `json:"delay_ms"`

	// Counter is the current counter value, maintained across invocations.
	// The Step Functions state machine passes this value between iterations.
	Counter int `json:"counter"`
}

// StepFunctionOutput represents the output payload from a Step Functions
// task invocation, indicating iteration progress and completion status.
type StepFunctionOutput struct {
	// Counter is the updated counter value after this iteration.
	Counter int `json:"counter"`

	// IterationsRemaining indicates how many iterations are left to perform.
	IterationsRemaining int `json:"iterations_remaining"`

	// Complete indicates whether the iteration cycle has finished.
	// When true, the Step Functions state machine should terminate.
	Complete bool `json:"complete"`

	// DelaySeconds is the delay duration in seconds for the Step Functions Wait state.
	// Calculated as DelayMs / 1000.0 from the input.
	// Step Functions Wait states require duration in seconds (float64).
	DelaySeconds float64 `json:"delay_seconds"`
}
```

## 4. Handler Functions

### 4.1 API Gateway Handler (`internal/handlers/apigateway.go`)

```go
package handlers

import (
	"context"
	"encoding/json"
)

// HandleAPIGateway processes HTTP requests from API Gateway.
//
// It handles GET requests to the root path ("/") and returns a success response
// with a greeting message. Other routes return ErrUnsupportedRoute.
//
// Response Structure:
// - Status Code: 200 for success, 404 for unsupported routes, 500 for errors
// - Headers: Content-Type set to application/json
// - Body: JSON-encoded APIGatewaySuccessResponse
//
// Returns ErrInvalidEvent if the request cannot be processed.
// Returns ErrUnsupportedRoute for unsupported HTTP routes.
func HandleAPIGateway(ctx context.Context, req APIGatewayRequest) (APIGatewayResponse, error)
```

### 4.2 Custom Event Handler (`internal/handlers/customevent.go`)

```go
package handlers

import (
	"context"
	"encoding/json"
)

// HandleCustomEvent processes direct Lambda invocations with arbitrary JSON payloads.
//
// This handler accepts any valid JSON event and echoes it back wrapped in a
// CustomEventResponse structure. This demonstrates handling of custom invocations
// that don't match API Gateway or Step Functions event schemas.
//
// The original event payload is returned in the Echo field, allowing the caller
// to verify that their custom payload was received correctly.
func HandleCustomEvent(ctx context.Context, event json.RawMessage) (CustomEventResponse, error)
```

### 4.3 Step Function Handler (`internal/handlers/stepfunction.go`)

```go
package handlers

import "context"

// HandleStepFunction manages iteration state for AWS Step Functions state machine tasks.
//
// State Management:
// - Increments Counter by 1 on each invocation
// - Calculates IterationsRemaining as (Iterations - Counter)
// - Sets Complete to true when Counter >= Iterations
// - Calculates DelaySeconds as (DelayMs / 1000.0) for Step Functions Wait state
//
// The Step Functions state machine passes the Counter value between iterations,
// maintaining state across multiple Lambda invocations. The state machine uses
// the Complete field to determine when to exit the iteration loop, and uses
// the DelaySeconds field in a Wait state to control the delay between iterations.
//
// Returns ErrIterationFailed if state management encounters an error.
func HandleStepFunction(ctx context.Context, input StepFunctionInput) (StepFunctionOutput, error)
```

## 5. Event Router

### 5.1 Event Detection (`internal/router/router.go`)

```go
package router

import "encoding/json"

// DetectEventType analyzes a raw Lambda event payload and determines its source.
//
// Detection Logic (in order):
// 1. API Gateway: Checks for presence of "requestContext" and "httpMethod" fields
// 2. Step Functions: Checks for presence of "start", "iterations", "delay_ms" fields
// 3. Custom Event: Default for any other valid JSON payload
//
// This function uses partial JSON unmarshaling via the eventProbe type to
// efficiently detect event structure without fully parsing the entire payload.
func DetectEventType(event json.RawMessage) EventType
```

### 5.2 Event Probe Type (`internal/router/router.go`)

```go
package router

// eventProbe is an internal type used for partial JSON unmarshaling during
// event type detection. It includes only the fields needed to distinguish
// between different event sources.
type eventProbe struct {
	// RequestContext is present in API Gateway events.
	// We only check for its existence, not its structure.
	RequestContext json.RawMessage `json:"requestContext,omitempty"`

	// HTTPMethod is present in API Gateway events.
	// Used in combination with RequestContext to detect API Gateway invocations.
	HTTPMethod string `json:"httpMethod,omitempty"`

	// Iterations is present in Step Function events.
	// Its presence indicates a Step Functions invocation.
	Iterations *int `json:"iterations,omitempty"`
}
```

### 5.3 Handler Response Requirements

**API Gateway Response Structure:**

All API Gateway responses must include:
- `statusCode` (int): HTTP status code (200, 404, 500, etc.)
- `headers` (map[string]string): HTTP headers including Content-Type
- `body` (string): JSON-encoded response payload

**Error Handling:**

All handlers return error types that the Lambda runtime can properly serialize. Errors are returned as standard Go errors and handled by the Lambda runtime's error marshaling.

**Step Function State Logic:**

The Step Function handler correctly manages iteration state:
- Counter increments by exactly 1 per invocation
- IterationsRemaining calculated as: `Iterations - Counter`
- Complete flag set when: `Counter >= Iterations`
- DelaySeconds calculated as: `DelayMs / 1000.0` (for Step Functions Wait state)

## 6. Main Lambda Handler

### 6.1 Universal Handler Function (`cmd/lambda/main.go`)

```go
package main

import (
	"context"
	"encoding/json"
)

// Handler is the main entry point for all Lambda invocations.
//
// This universal handler receives raw JSON events from any source (API Gateway,
// Step Functions, or direct invocations) and routes them to the appropriate
// handler based on event structure detection.
//
// Routing Logic:
// 1. Use router.DetectEventType to analyze the event payload
// 2. Unmarshal event into the appropriate request type
// 3. Call the corresponding handler function
// 4. Return the handler's response (type varies by handler)
//
// The return type is 'any' because different handlers return different response types:
// - API Gateway: APIGatewayResponse
// - Step Functions: StepFunctionOutput
// - Custom Events: CustomEventResponse
//
// The Lambda runtime handles marshaling the response back to JSON.
func Handler(ctx context.Context, event json.RawMessage) (any, error)
```

### 6.2 Configuration Management (`cmd/lambda/main.go`)

```go
package main

import "os"

// Config holds the Lambda function's configuration loaded from environment variables.
type Config struct {
	// StackName is the unique identifier for this deployment stack.
	// Required. Loaded from STACK_NAME environment variable.
	StackName string

	// AWSRegion is the AWS region for this deployment.
	// Optional. Defaults to "us-east-1" if not specified.
	// Loaded from AWS_REGION environment variable.
	AWSRegion string
}

// LoadConfig loads configuration from environment variables.
//
// Environment Variables:
// - STACK_NAME (required): Unique name for this deployment
// - AWS_REGION (optional): AWS region (default: "us-east-1")
//
// Error Handling:
// Returns an error if STACK_NAME is missing or empty, as it's required
// for proper resource naming and identification.
//
// If AWS_REGION is not set, DefaultAWSRegion ("us-east-1") is used.
func LoadConfig() (Config, error)
```

### 6.3 Main Entry Point (`cmd/lambda/main.go`)

```go
package main

import (
	"github.com/aws/aws-lambda-go/lambda"

	"github.com/prethora/starter_golang_aws_lambda/internal/handlers"
	"github.com/prethora/starter_golang_aws_lambda/internal/router"
)

// main initializes the Lambda function and registers the handler with the runtime.
//
// Initialization Sequence:
// 1. Load configuration from environment variables via LoadConfig()
// 2. Register the universal Handler function with the Lambda runtime
//
// The router package provides stateless event detection via router.DetectEventType,
// which is called by the Handler function for each invocation.
//
// The Lambda runtime takes control after lambda.Start() is called.
// All subsequent invocations are routed through the Handler function.
//
// This function must call lambda.Start(Handler) per AWS Lambda Go runtime requirements.
// The lambda.Start() call blocks indefinitely, processing events as they arrive.
func main()
```

### 6.4 Import Requirements for Main Package

The `cmd/lambda/main.go` file requires these imports:

**Standard Library:**
- `context` — Context propagation through handlers
- `encoding/json` — JSON marshaling/unmarshaling
- `os` — Environment variable access for LoadConfig

**AWS Lambda SDK:**
- `github.com/aws/aws-lambda-go/lambda` — Lambda runtime integration

**Internal Packages:**
- `github.com/prethora/starter_golang_aws_lambda/internal/handlers` — Handler functions
- `github.com/prethora/starter_golang_aws_lambda/internal/router` — Event type detection

## 7. Docker Build Configuration

### 7.1 Dockerfile Structure

The project uses a **multi-stage Docker build** to produce a minimal Lambda container image.

#### Builder Stage

```dockerfile
FROM golang:1.23-alpine AS builder

# Builder stage compiles the Go binary
# - Uses Alpine for smaller build image
# - CGO_ENABLED=0 for static linking (required for Lambda)
# - Output: Single binary named 'bootstrap'
```

**Build Configuration:**
- Base Image: `golang:1.23-alpine` (or latest stable Go version)
- CGO: Disabled (`CGO_ENABLED=0`) for static linking
- Output Binary: `bootstrap` (AWS Lambda Go runtime requirement)
- Build Flags: Static linking flags for portable binary

#### Runtime Stage

```dockerfile
FROM public.ecr.aws/lambda/provided:al2023

# Runtime stage creates the Lambda execution environment
# - Uses AWS Lambda custom runtime base (Amazon Linux 2023)
# - Copies only the compiled binary from builder stage
# - Minimal image size for fast cold starts
```

**Runtime Configuration:**
- Base Image: `public.ecr.aws/lambda/provided:al2023`
- Binary Location: `/var/task/bootstrap` (or `${LAMBDA_TASK_ROOT}/bootstrap`)
- CMD: `["bootstrap"]` per AWS Lambda requirements

**Important:** The binary **must** be named `bootstrap` and placed at `/var/task/bootstrap` for the AWS Lambda custom runtime to recognize and execute it. The `/var/task` directory (accessible via `${LAMBDA_TASK_ROOT}`) is where Lambda expects custom runtime binaries. Do NOT use `/var/runtime/bootstrap` - that path is reserved for the Lambda runtime itself.

## 8. Terraform Infrastructure

### 8.1 Variable Definitions (`terraform/variables.tf`)

```hcl
variable "stack_name" {
  description = "Unique identifier for this deployment stack (used for all resource naming)"
  type        = string
}

variable "enable_api_gateway" {
  description = "Deploy API Gateway HTTP API integration"
  type        = bool
  default     = true
}

variable "enable_step_functions" {
  description = "Deploy Step Functions state machine"
  type        = bool
  default     = true
}

variable "enable_custom_events" {
  description = "Allow direct Lambda invocation with custom events"
  type        = bool
  default     = true
}

variable "lambda_memory" {
  description = "Lambda function memory allocation in MB (128-10240)"
  type        = number
  default     = 512
}

variable "lambda_timeout" {
  description = "Lambda function timeout in seconds (1-900)"
  type        = number
  default     = 30
}

variable "aws_region" {
  description = "AWS region for resource deployment"
  type        = string
  default     = "us-east-1"
}
```

### 8.2 Output Definitions (`terraform/outputs.tf`)

```hcl
output "lambda_function_arn" {
  description = "ARN of the deployed Lambda function"
  value       = aws_lambda_function.main.arn
}

output "lambda_function_name" {
  description = "Lambda function name for AWS CLI convenience commands"
  value       = aws_lambda_function.main.function_name
}

output "ecr_repository_url" {
  description = "ECR repository URL for Docker image push operations"
  value       = aws_ecr_repository.main.repository_url
}

output "api_gateway_url" {
  description = "HTTP API Gateway endpoint URL (only if API Gateway is enabled)"
  value       = var.enable_api_gateway ? aws_apigatewayv2_api.main[0].api_endpoint : null
}

output "step_function_arn" {
  description = "ARN of the Step Functions state machine (only if Step Functions enabled)"
  value       = var.enable_step_functions ? aws_sfn_state_machine.main[0].arn : null
}
```

**Output Usage:**

- `lambda_function_arn`: Full ARN for IAM policies, triggers, and resource references
- `lambda_function_name`: Convenient short name for AWS CLI commands:
  ```bash
  aws lambda invoke --function-name $(terraform output -raw lambda_function_name) response.json
  ```
- `ecr_repository_url`: Required by deployment script for Docker image push:
  ```bash
  docker tag myimage:latest $(terraform output -raw ecr_repository_url):latest
  docker push $(terraform output -raw ecr_repository_url):latest
  ```
- `api_gateway_url`: HTTP endpoint for testing and integration (conditional)
- `step_function_arn`: State machine ARN for starting executions (conditional)

### 8.3 Resource Naming Patterns

All AWS resources follow a consistent naming pattern using the `stack_name` variable:

| Resource Type | Naming Pattern | Example (stack_name="myservice") |
|--------------|----------------|----------------------------------|
| Lambda Function | `${stack_name}-lambda` | `myservice-lambda` |
| ECR Repository | `${stack_name}-repo` | `myservice-repo` |
| API Gateway | `${stack_name}-api` | `myservice-api` |
| Step Functions | `${stack_name}-state-machine` | `myservice-state-machine` |
| CloudWatch Logs | `/aws/lambda/${stack_name}` | `/aws/lambda/myservice` |
| Lambda IAM Role | `${stack_name}-lambda-role` | `myservice-lambda-role` |
| Step Functions IAM Role | `${stack_name}-sfn-role` | `myservice-sfn-role` |

**Rationale:** Consistent naming enables:
- Easy identification of related resources
- Clean separation between multiple deployments
- Simple resource filtering and cost tracking

### 8.4 IAM Policy Requirements

#### CloudWatch Logs Policy (Always Required)

The Lambda execution role must have permissions to write logs:

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

This policy enables:
- Log group creation for the Lambda function
- Log stream creation for each invocation
- Writing log events from the function

#### Execution Role Architecture

**Important:** This architecture involves **two separate execution roles** with different permissions:

1. **Lambda Execution Role** (attached to the Lambda function)
2. **Step Functions Execution Role** (attached to the state machine, if enabled)

#### Lambda Execution Role Summary

The Lambda execution role requires **only** CloudWatch Logs permissions (defined above). It does **NOT** need:
- ❌ `states:StartExecution` — Lambda never starts Step Functions executions
- ❌ Additional AWS service permissions beyond CloudWatch Logs

The Lambda function is purely reactive - it responds to invocations from API Gateway, Step Functions, or direct invocations. It does not initiate any AWS service calls.

#### Step Functions Execution Role Policy (Conditional)

If `enable_step_functions = true`, the **Step Functions state machine** needs its own execution role with permission to invoke the Lambda function:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "lambda:InvokeFunction"
      ],
      "Resource": "<lambda-function-arn>"
    }
  ]
}
```

**Resource:** The resource ARN should reference the specific Lambda function ARN.

**Why this matters:** The Step Functions state machine invokes the Lambda function as a task in its workflow. The state machine's execution role needs permission to call Lambda, NOT the other way around.

#### Permission Scope

Both execution roles follow the **principle of least privilege**:
- **Lambda role**: Only CloudWatch Logs permissions
- **Step Functions role**: Only lambda:InvokeFunction for the specific Lambda function
- No excessive permissions beyond what's documented
- No wildcard permissions on sensitive actions
- No cross-account or administrative permissions

## 9. Deployment Scripts

### 9.1 Deployment Script (`scripts/deploy.sh`)

The deployment script automates the complete deployment process from Docker build to Terraform provisioning.

**Deployment Sequence:**

1. **Load Configuration**
   - Source `config.env` to load `STACK_NAME`
   - Validate that `STACK_NAME` is set and non-empty
   - Exit with error if configuration is missing

2. **Build Docker Image**
   - Execute `docker build` with multi-stage Dockerfile
   - Tag image as `${STACK_NAME}-lambda:latest`
   - Verify build succeeded

3. **Push to ECR**
   - Authenticate with ECR: `aws ecr get-login-password | docker login`
   - Tag image with ECR repository URL: `${AWS_ACCOUNT}.dkr.ecr.${REGION}.amazonaws.com/${STACK_NAME}-repo:latest`
   - Push image to ECR
   - Verify push succeeded

4. **Deploy Infrastructure**
   - Initialize Terraform: `terraform init` (if not already initialized)
   - Apply configuration: `terraform apply -var="stack_name=${STACK_NAME}"`
   - Wait for deployment to complete

5. **Output Results**
   - Display Lambda function ARN
   - Display API Gateway URL (if enabled)
   - Display Step Function ARN (if enabled)
   - Display CloudWatch log group path

### 9.2 Destroy Script (`scripts/destroy.sh`)

The destroy script tears down all deployed infrastructure.

**Teardown Sequence:**

1. **Load Configuration**
   - Source `config.env` to load `STACK_NAME`
   - Validate configuration

2. **Destroy Terraform Resources**
   - Run `terraform destroy -var="stack_name=${STACK_NAME}"`
   - Confirm destruction (or use `-auto-approve` flag)
   - Wait for all resources to be removed

3. **Optional: Clean ECR Images**
   - Delete all images from ECR repository
   - Note: ECR repository itself is managed by Terraform

4. **Verify Cleanup**
   - Confirm all resources removed
   - Display cleanup summary

### 9.3 End-to-End Test Script (`scripts/test-e2e.sh`)

The E2E test script runs the complete test lifecycle with a temporary stack.

**Test Lifecycle:**

1. **Generate Test Stack Name**
   - Create unique identifier: `test-$(date +%s)` (Unix timestamp)
   - Export as `STACK_NAME` for deployment

2. **Deploy Test Stack**
   - Call `deploy.sh` with test stack name
   - Capture deployment outputs (ARNs, URLs)
   - Store in test configuration

3. **Run E2E Tests**
   - Execute Go test suite: `go test ./tests/e2e/...`
   - Tests use captured deployment outputs
   - Report test results

4. **Cleanup Test Stack**
   - Call `destroy.sh` with test stack name
   - Verify all resources removed
   - Delete temporary test artifacts

5. **Report Results**
   - Display test summary (pass/fail counts)
   - Exit with appropriate status code
   - Preserve logs on failure for debugging

## 10. End-to-End Testing Infrastructure

### 10.1 Test Stack Type (`tests/e2e/types.go`)

```go
package e2e

// TestStack represents a deployed test stack with all resource identifiers
// needed for end-to-end testing.
type TestStack struct {
	// StackName is the unique identifier for this test deployment.
	StackName string

	// LambdaARN is the ARN of the deployed Lambda function.
	LambdaARN string

	// APIGatewayURL is the HTTP API endpoint URL.
	// Empty if API Gateway is not enabled.
	APIGatewayURL string

	// StepFunctionARN is the ARN of the Step Functions state machine.
	// Empty if Step Functions are not enabled.
	StepFunctionARN string

	// AWSRegion is the AWS region where the stack is deployed.
	AWSRegion string
}
```

### 10.2 Stack Management Functions (`tests/e2e/stack.go`)

```go
package e2e

import (
	"context"

	"github.com/aws/aws-sdk-go-v2/config"
	"github.com/aws/aws-sdk-go-v2/service/lambda"
	"github.com/aws/aws-sdk-go-v2/service/sfn"
)

// DeployTestStack deploys a temporary test stack for E2E testing.
//
// This function:
// 1. Generates a unique stack name with timestamp
// 2. Builds and pushes Docker image
// 3. Deploys infrastructure via Terraform
// 4. Captures and returns all resource identifiers
//
// Returns TestStack with resource ARNs and URLs, or error on failure.
func DeployTestStack(ctx context.Context, stackName string) (TestStack, error)

// CleanupTestStack destroys a test stack and all its resources.
//
// This function:
// 1. Runs Terraform destroy for the stack
// 2. Removes ECR images
// 3. Verifies all resources are deleted
//
// Returns error if cleanup fails.
func CleanupTestStack(ctx context.Context, stack TestStack) error
```

### 10.3 Test Phases

The E2E test suite follows a structured five-phase approach to verify all invocation patterns.

#### Phase 1: Deploy

```go
// Deploy phase creates a unique test stack
func TestMain(m *testing.M) {
	// Generate unique stack name
	stackName := fmt.Sprintf("test-%d", time.Now().Unix())

	// Deploy test stack
	testStack, err := DeployTestStack(context.Background(), stackName)
	if err != nil {
		log.Fatal("Failed to deploy test stack:", err)
	}

	// Run tests
	exitCode := m.Run()

	// Cleanup
	CleanupTestStack(context.Background(), testStack)

	os.Exit(exitCode)
}
```

#### Phase 2: Test API Gateway

**Test: HTTP GET returns expected response**

```go
func TestAPIGateway(t *testing.T) {
	// HTTP GET to testStack.APIGatewayURL
	resp, err := http.Get(testStack.APIGatewayURL)

	// Assert: Status code = 200
	assert.Equal(t, 200, resp.StatusCode)

	// Assert: Response body matches expected structure
	var body APIGatewaySuccessResponse
	json.NewDecoder(resp.Body).Decode(&body)

	assert.Equal(t, "Hello World", body.Message)
	assert.Equal(t, "api-gateway", body.Source)
}
```

**Success Criteria:**
- Status code is 200
- Response body contains `{"message": "Hello World", "source": "api-gateway"}`

#### Phase 3: Test Custom Event

**Test: Direct invocation echoes payload**

```go
func TestCustomEvent(t *testing.T) {
	// Create Lambda client
	client := lambda.NewFromConfig(awsConfig)

	// Invoke with test payload
	payload := []byte(`{"foo": "bar"}`)
	output, err := client.Invoke(ctx, &lambda.InvokeInput{
		FunctionName: aws.String(testStack.LambdaARN),
		Payload:      payload,
	})

	// Assert: Response contains echo
	var resp CustomEventResponse
	json.Unmarshal(output.Payload, &resp)

	assert.Equal(t, "custom-event", resp.Source)
	assert.JSONEq(t, `{"foo": "bar"}`, string(resp.Echo))
}
```

**Success Criteria:**
- Response source field is "custom-event"
- Echo field contains the exact input payload `{"foo": "bar"}`

#### Phase 4: Test Step Functions

**Test: State machine completes iteration cycle**

```go
func TestStepFunctions(t *testing.T) {
	// Create Step Functions client
	client := sfn.NewFromConfig(awsConfig)

	// Start execution
	input := `{"start": 0, "iterations": 3, "delay_ms": 500}`
	exec, err := client.StartExecution(ctx, &sfn.StartExecutionInput{
		StateMachineArn: aws.String(testStack.StepFunctionARN),
		Input:           aws.String(input),
	})

	// Poll until complete
	for {
		status, err := client.DescribeExecution(ctx, &sfn.DescribeExecutionInput{
			ExecutionArn: exec.ExecutionArn,
		})

		if status.Status == types.ExecutionStatusSucceeded {
			// Parse output
			var output StepFunctionOutput
			json.Unmarshal([]byte(*status.Output), &output)

			// Assert: Counter reached expected value
			assert.Equal(t, 3, output.Counter)
			assert.Equal(t, 0, output.IterationsRemaining)
			assert.True(t, output.Complete)
			assert.Equal(t, 0.5, output.DelaySeconds) // 500ms = 0.5 seconds
			break
		}

		time.Sleep(1 * time.Second)
	}
}
```

**Success Criteria:**
- State machine execution completes successfully
- Final counter value is 3
- IterationsRemaining is 0
- Complete flag is true
- DelaySeconds is 0.5 (calculated from 500ms input)

#### Phase 5: Cleanup

**Test: All resources are removed**

```go
func TestMain(m *testing.M) {
	// ... (deploy phase)

	// Run tests
	exitCode := m.Run()

	// Cleanup phase
	err := CleanupTestStack(context.Background(), testStack)
	if err != nil {
		log.Error("Failed to cleanup test stack:", err)
		// Don't fail the test run, but log the error
	}

	os.Exit(exitCode)
}
```

### 10.4 E2E Test Dependencies

**AWS SDK v2 Imports (test-only):**

The E2E test package is the **only** location where AWS SDK v2 imports are permitted:

```go
import (
	"github.com/aws/aws-sdk-go-v2/config"          // AWS configuration
	"github.com/aws/aws-sdk-go-v2/service/lambda"  // Lambda invocation
	"github.com/aws/aws-sdk-go-v2/service/sfn"     // Step Functions execution
)
```

**Version Requirement:**
- `github.com/aws/aws-sdk-go-v2 v1.41.1` (test-only dependency)

**Import Boundary Enforcement:**

The Lambda function code (`cmd/` and `internal/` packages) must **never** import AWS SDK v2. This keeps the deployment binary minimal and focused solely on the Lambda runtime SDK.

## 11. Unit Testing Infrastructure

### 11.1 Test Helper Functions (`tests/unit/helpers.go`)

```go
package unit

import (
	"github.com/aws/aws-lambda-go/events"

	"github.com/prethora/starter_golang_aws_lambda/internal/handlers"
)

// NewTestAPIGatewayRequest creates a mock API Gateway request for unit testing.
//
// This helper function constructs a minimal API Gateway proxy request with the
// specified HTTP method, path, and body. It populates the RequestContext with
// required fields to satisfy the API Gateway event structure.
//
// Parameters:
// - method: HTTP method (GET, POST, etc.)
// - path: Request path (e.g., "/", "/api/users")
// - body: Request body as a string (can be empty for GET requests)
//
// Returns a fully populated APIGatewayProxyRequest suitable for testing
// the HandleAPIGateway function without deploying infrastructure.
func NewTestAPIGatewayRequest(method, path string, body string) events.APIGatewayProxyRequest

// NewTestStepFunctionInput creates a test Step Function input for unit testing.
//
// This helper function constructs a StepFunctionInput with all fields populated
// using the provided values. It's useful for testing the HandleStepFunction
// function with various iteration states.
//
// Parameters:
// - start: Initial counter value
// - iterations: Total number of iterations to perform
// - delayMs: Delay in milliseconds between iterations
// - counter: Current counter value (for testing mid-iteration state)
//
// Returns a StepFunctionInput ready for testing.
func NewTestStepFunctionInput(start, iterations, delayMs, counter int) handlers.StepFunctionInput
```

### 11.2 Mock Interfaces (`tests/unit/mocks.go`)

```go
package unit

import (
	"context"
	"encoding/json"

	"github.com/prethora/starter_golang_aws_lambda/internal/handlers"
)

// MockLambdaHandler is a mock interface for testing the main Handler routing logic.
//
// This interface mirrors the three handler functions and allows for dependency
// injection in tests. By implementing this interface with a mock, tests can verify
// that the main Handler function correctly routes events to the appropriate handler
// based on event type detection.
//
// Example usage:
//   mock := &MockHandlerImpl{}
//   mock.On("HandleAPIGateway", ...).Return(response, nil)
//   // Test that main Handler calls HandleAPIGateway for API Gateway events
type MockLambdaHandler interface {
	// HandleAPIGateway handles API Gateway events.
	HandleAPIGateway(ctx context.Context, req handlers.APIGatewayRequest) (handlers.APIGatewayResponse, error)

	// HandleCustomEvent handles custom event payloads.
	HandleCustomEvent(ctx context.Context, event json.RawMessage) (handlers.CustomEventResponse, error)

	// HandleStepFunction handles Step Functions state machine tasks.
	HandleStepFunction(ctx context.Context, input handlers.StepFunctionInput) (handlers.StepFunctionOutput, error)
}
```

### 11.3 Unit Test Organization

Unit tests are organized in the `tests/unit/` directory with dedicated test files for each component:

**Test Files:**

- `apigateway_test.go` — Tests for `HandleAPIGateway` function
  - Test GET / returns success response
  - Test unsupported routes return 404
  - Test invalid requests return 500
  - Test response structure matches APIGatewayResponse schema

- `customevent_test.go` — Tests for `HandleCustomEvent` function
  - Test echo functionality with various payloads
  - Test response wraps input in CustomEventResponse structure
  - Test source field is set correctly

- `stepfunction_test.go` — Tests for `HandleStepFunction` function
  - Test counter increments correctly
  - Test iterations_remaining calculation
  - Test complete flag set when counter >= iterations
  - Test state management across multiple invocations

- `router_test.go` — Tests for `DetectEventType` function
  - Test detects API Gateway events (presence of requestContext)
  - Test detects Step Function events (presence of iterations field)
  - Test defaults to CustomEvent for other JSON payloads
  - Test handles malformed JSON

- `main_test.go` — Tests for main `Handler` routing logic
  - Test routes API Gateway events to HandleAPIGateway
  - Test routes Step Function events to HandleStepFunction
  - Test routes custom events to HandleCustomEvent
  - Test error handling and response marshaling

**Test Execution:**

```bash
# Run all unit tests
go test ./tests/unit/...

# Run with coverage
go test -cover ./tests/unit/...

# Run specific test file
go test ./tests/unit/apigateway_test.go
```

### 11.4 Logging

**Note on Logging Abstraction:**

The design document does not specify a custom logging abstraction. If logging is needed, the implementation may use:
- Standard library `log` package for simple logging
- `fmt.Println` for debugging during development
- CloudWatch Logs (automatic for Lambda) for production observability

No Logger interface is required at this time. A logging abstraction can be added in future iterations if the need arises.

## 12. Summary

This architecture document defines the complete type system, interfaces, and infrastructure specifications for the AWS Lambda Go Starter Package.

**Coverage:**

✓ **Package Structure** — Complete directory layout and file organization
✓ **Types & Interfaces** — All request/response types, handler signatures, router types
✓ **Error Handling** — Sentinel errors and error handling patterns
✓ **Configuration** — Environment variable management and validation
✓ **Main Entry Point** — Lambda handler registration and initialization
✓ **Docker Build** — Multi-stage build with proper base images
✓ **Terraform Infrastructure** — Variables, outputs, naming patterns, IAM policies
✓ **Deployment Scripts** — Automated deployment, destruction, and E2E testing
✓ **Testing Infrastructure** — Unit test helpers and E2E test framework
✓ **Dependencies** — Version requirements and import boundary rules

**Implementation Readiness:**

The architecture is **ready for implementation**. All types, interfaces, and signatures are defined with sufficient detail for code generation. Infrastructure specifications are complete and testable. The design follows AWS Lambda best practices and Go conventions.

**Next Steps:**

1. Generate implementation code from this architecture specification
2. Implement handler logic according to defined signatures
3. Create Dockerfile following the multi-stage build pattern
4. Write Terraform configurations per the specifications
5. Implement deployment scripts with proper error handling
6. Write unit tests using the defined helper functions
7. Write E2E tests following the five-phase structure
