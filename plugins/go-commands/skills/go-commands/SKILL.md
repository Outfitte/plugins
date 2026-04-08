---
name: go-commands
description: Go build and test commands reference. Use when building Go projects, running Go tests, checking coverage, managing dependencies, or cross-compiling for targets like Lambda.
---

## Dependency Management

```bash
go mod download        # download all dependencies
go mod verify          # verify dependency integrity
go mod tidy            # remove unused, add missing
go get <module>        # add or update a dependency
```

## Build Commands

```bash
# Build a single package
go build -o bin/program ./cmd/program

# Build all packages
go build ./...

# Build with size optimizations (strip debug info)
go build -ldflags="-s -w" -o bin/program ./cmd/program

# Cross-compile for AWS Lambda (linux/amd64)
GOOS=linux GOARCH=amd64 go build -o bin/bootstrap ./cmd/program
```

## Test Execution

```bash
# Run all tests recursively
go test ./...

# Run all tests in a package
go test ./internal/mypackage

# Run a specific test
go test -run ^TestFunctionName$ ./internal/mypackage

# Run with verbose output
go test -v ./...

# Run with race detector
go test -race ./...

# Run with timeout
go test -timeout 30s ./...
```

## Coverage

```bash
# Show coverage percentage
go test -cover ./...

# Generate coverage profile
go test -cover -coverprofile=coverage.out ./...

# View per-function coverage
go tool cover -func=coverage.out

# Filter to a specific file
go tool cover -func=coverage.out | grep filename.go

# Open interactive HTML report
go tool cover -html=coverage.out
```
