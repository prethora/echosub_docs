# Client Need: AWS Lambda Go Starter Package

**Package**: `prethora/starter_golang_aws_lambda`

## Overview

A production-ready starter template for building AWS Lambda functions with Go and Docker. This starter provides a complete foundation including Terraform infrastructure-as-code, Docker containerization, and pre-configured support for three common Lambda invocation patterns: API Gateway HTTP endpoints, direct custom event invocations, and AWS Step Functions state machines.

**This is a template project** designed to be cloned and customized for specific microservices. It eliminates the boilerplate and configuration complexity of setting up Lambda functions correctly, providing a battle-tested starting point with all infrastructure, deployment, and testing patterns already solved.

**Primary Use Case**: Rapidly spin up new Lambda-based microservices (e.g., transcription services, API backends, batch processors) without re-solving Docker/Terraform/Lambda configuration each time.

## Platform Support

### Target Platforms

**Runtime Environment:**
- AWS Lambda (Docker container runtime)
- Go 1.22+ (use latest stable version at time of implementation)
- Docker with multi-stage builds
- Amazon Linux 2023 base image (or latest AWS-recommended base)

**Development Environment:**
- macOS (arm64, amd64)
- Linux (amd64)
- Windows with WSL2

**AWS Services:**
- AWS Lambda
- API Gateway (HTTP API - simpler, cheaper, sufficient for Lambda backends)
- Step Functions
- IAM (for execution roles)
- ECR (for container images)
- CloudWatch (for logs)

**Infrastructure:**
- Terraform 1.6+ (use latest stable version at time of implementation)

**Version Requirements**: All dependencies (Go, Terraform, Docker base images, AWS provider) should use the **latest stable versions** at implementation time. The implementing agents must research current versions rather than relying on training data.

## Architecture

### Project Design Philosophy

This starter must be structured as a **ready-to-clone template** that:

- Works immediately after cloning with minimal configuration (set name + AWS credentials)
- Demonstrates all three invocation patterns in a single deployable stack
- Allows easy disabling of unused invocation patterns
- Uses a single configuration point for naming across all resources
- Includes comprehensive testing at both unit and end-to-end levels
- Documents everything needed to go from clone to production deployment

### Project Structure

```
starter_golang_aws_lambda/
├── cmd/
│   └── lambda/
│       └── main.go              # Lambda entry point
├── internal/
│   ├── handlers/
│   │   ├── apigateway.go        # API Gateway event handler
│   │   ├── customevent.go       # Custom event handler
│   │   └── stepfunction.go      # Step Function handler
│   └── router/
│       └── router.go            # Event type detection and routing
├── terraform/
│   ├── main.tf                  # Main Terraform configuration
│   ├── variables.tf             # Variable definitions
│   ├── outputs.tf               # Output values
│   ├── api_gateway.tf           # HTTP API Gateway resources
│   ├── step_functions.tf        # Step Functions resources
│   └── lambda.tf                # Lambda + ECR resources
├── tests/
│   ├── unit/                    # Unit tests
│   └── e2e/                     # End-to-end tests
├── scripts/
│   ├── deploy.sh                # Deploy script
│   ├── destroy.sh               # Teardown script
│   └── test-e2e.sh              # End-to-end test runner
├── Dockerfile                   # Multi-stage Docker build
├── config.env                   # Single configuration file
├── go.mod
├── go.sum
└── README.md                    # Comprehensive documentation
```

## Configuration

### Single Configuration Point

All resource naming must be controlled from a single file (`config.env`) that sets the stack name:

```bash
# config.env - The ONLY place to configure the stack name
STACK_NAME=my-service

# Optional overrides (derived from STACK_NAME by default)
# AWS_REGION=us-east-1
```

The `STACK_NAME` propagates to:
- Lambda function name: `${STACK_NAME}-lambda`
- ECR repository: `${STACK_NAME}-repo`
- API Gateway: `${STACK_NAME}-api`
- Step Function: `${STACK_NAME}-state-machine`
- IAM roles: `${STACK_NAME}-lambda-role`
- CloudWatch log groups: `/aws/lambda/${STACK_NAME}`

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `STACK_NAME` | Yes | Unique name for this deployment |
| `AWS_ACCESS_KEY_ID` | Yes | AWS credentials |
| `AWS_SECRET_ACCESS_KEY` | Yes | AWS credentials |
| `AWS_REGION` | No | Default: `us-east-1` |

## Lambda Handler Behavior

### Event Type Detection

The Lambda handler must automatically detect the event source and route to the appropriate handler:

```go
// Pseudocode for event routing
func Handler(ctx context.Context, event json.RawMessage) (any, error) {
    switch detectEventType(event) {
    case APIGatewayEvent:
        return handleAPIGateway(ctx, event)
    case StepFunctionEvent:
        return handleStepFunction(ctx, event)
    default:
        return handleCustomEvent(ctx, event)
    }
}
```

### API Gateway Handler

**Trigger**: HTTP requests via API Gateway

**Behavior**: 
- Responds to `GET /` with `{"message": "Hello World", "source": "api-gateway"}`
- Returns proper HTTP response structure with status code, headers, body

**Test Verification**: HTTP GET to the API Gateway URL returns 200 with expected JSON body.

### Custom Event Handler

**Trigger**: Direct Lambda invocation with arbitrary JSON payload

**Behavior**:
- Echoes the entire input event back in the response
- Wraps response as `{"source": "custom-event", "echo": <original_event>}`

**Test Verification**: Invoking Lambda with `{"foo": "bar"}` returns `{"source": "custom-event", "echo": {"foo": "bar"}}`.

### Step Function Handler

**Trigger**: Step Functions state machine invocation

**Input Schema**:
```json
{
    "start": 0,           // Starting counter value (default: 0)
    "iterations": 5,      // Number of iterations (default: 5)
    "delay_ms": 1000      // Milliseconds between iterations (default: 1000)
}
```

**Behavior**:
- Maintains counter state across iterations
- Each invocation increments counter and checks if iterations complete
- Returns `{"counter": N, "iterations_remaining": M, "complete": bool}`
- Step Function definition handles the iteration loop and wait states

**State Machine Flow**:
```
[Start] → [Initialize] → [Invoke Lambda] → [Check Complete]
                              ↑                    │
                              │       No           │ Yes
                              └────[Wait]←─────────┴────→ [End]
```

**Test Verification**: Start execution, poll until complete, verify counter reached expected value.

## Terraform Infrastructure

### Resource Toggles

Each invocation pattern can be disabled via Terraform variables:

```hcl
variable "enable_api_gateway" {
  description = "Deploy API Gateway integration"
  type        = bool
  default     = true
}

variable "enable_step_functions" {
  description = "Deploy Step Functions state machine"
  type        = bool
  default     = true
}

variable "enable_custom_events" {
  description = "Allow direct Lambda invocation"
  type        = bool
  default     = true
}
```

### IAM Configuration

The starter must create a properly scoped IAM role with:
- Basic Lambda execution permissions (CloudWatch Logs)
- Step Functions invocation permissions (if enabled)
- No excessive permissions

## Docker Configuration

### Dockerfile Requirements

- Multi-stage build (builder + runtime)
- Use official AWS Lambda Go base image (latest version)
- Proper executable naming for Lambda runtime
- Minimal final image size
- Non-root user execution where possible

### Build Output

- Single statically-linked Go binary
- Placed in Lambda-expected location
- Named according to Lambda runtime requirements

## Testing Strategy

### Unit Tests

Test each handler in isolation:
- API Gateway handler with mock events
- Custom event handler with various payloads
- Step Function handler with state progression

### End-to-End Tests

Automated E2E test suite that:

1. **Deploy**: Creates a test stack with unique name (e.g., `test-${timestamp}`)
2. **Test API Gateway**: 
   - HTTP GET to deployed URL
   - Verify 200 response with expected body
3. **Test Custom Event**:
   - AWS SDK Lambda invoke with test payload
   - Verify echo response
4. **Test Step Functions**:
   - Start execution with `{"start": 0, "iterations": 3, "delay_ms": 500}`
   - Poll execution status until complete
   - Verify final counter value is 3
5. **Cleanup**: Destroy test stack completely

### Test Execution

```bash
# Run unit tests
go test ./...

# Run E2E tests (deploys, tests, destroys)
./scripts/test-e2e.sh
```

## Error Handling

### Handler Errors

```go
var (
    ErrInvalidEvent     = errors.New("handler: unable to parse event")
    ErrUnsupportedRoute = errors.New("handler: unsupported API route")
    ErrIterationFailed  = errors.New("handler: step function iteration failed")
)
```

### Deployment Errors

The deployment scripts must provide clear error messages for common issues:
- Missing AWS credentials
- Missing `STACK_NAME` configuration
- Docker build failures
- Terraform state conflicts

## Documentation Requirements

### README Sections

1. **Quick Start**: Clone → Configure → Deploy in under 5 minutes
2. **Prerequisites**: Required tools and versions, AWS account setup
3. **Configuration**: How to set `STACK_NAME` and AWS credentials
4. **IAM Setup**: Step-by-step IAM user/role creation with required permissions
5. **Deployment**: Full deployment walkthrough
6. **Testing**: How to run unit and E2E tests
7. **Invocation Patterns**: How each pattern works with examples
8. **Disabling Features**: How to disable API Gateway, Step Functions, or Custom Events
9. **Customization**: Where to add your business logic
10. **Cleanup**: How to destroy deployed resources
11. **Troubleshooting**: Common issues and solutions

### Code Comments

All Go code must include:
- Package-level documentation
- Function documentation with parameter descriptions
- Inline comments for non-obvious logic

## Out of Scope

- Application business logic beyond demo handlers
- Custom domain configuration for API Gateway
- REST API features (caching, request transformation, usage plans) - HTTP API chosen for simplicity
- VPC configuration
- Secrets management (beyond environment variables)
- CI/CD pipeline configuration
- Multi-region deployment
- Lambda layers
- Provisioned concurrency configuration

## Success Criteria

1. **Zero-to-Deploy**: New user can clone and deploy in under 15 minutes following README
2. **Single Config Point**: Changing `STACK_NAME` updates all resources correctly
3. **API Gateway Works**: HTTP GET returns expected response
4. **Custom Events Work**: Direct invocation echoes payload correctly
5. **Step Functions Work**: State machine completes iteration cycle correctly
6. **Event Detection**: Handler correctly identifies and routes all three event types
7. **E2E Tests Pass**: Automated tests deploy, verify, and cleanup successfully
8. **Clean Teardown**: `destroy.sh` removes all AWS resources completely
9. **Documentation Complete**: README covers all setup and usage scenarios
10. **Latest Versions**: All dependencies use current stable versions (not outdated training data)
11. **Minimal Permissions**: IAM role has only required permissions
12. **Toggle Features**: Each invocation pattern can be independently disabled

## Dependencies

### Go Dependencies

```
module github.com/prethora/starter_golang_aws_lambda

go 1.22

require (
    github.com/aws/aws-lambda-go v1.47.0
    github.com/aws/aws-sdk-go-v2 v1.26.0
    github.com/aws/aws-sdk-go-v2/config v1.27.0
    github.com/aws/aws-sdk-go-v2/service/lambda v1.54.0
    github.com/aws/aws-sdk-go-v2/service/sfn v1.27.0
)
```

*Note: Version numbers are indicative. Implementing agents must research and use the latest stable versions.*

### Infrastructure Dependencies

- Terraform AWS Provider (latest stable)
- Docker (latest stable)
- AWS CLI v2 (for E2E tests)