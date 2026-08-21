# Go CLI Reference: Testing & Coverage

A reference for testing and coverage commands used in my Credora Project.

---

## Testing

### Run all tests

```bash
go test ./...
```

Recursively runs all tests across every package in the module. Use this before pushing code or opening a PR.

---

### Run tests in a particular folder

```bash
go test ./service/account/... -v
```

Recursively runs all tests across every package in that module. Use this before pushing code or opening a PR.

---

### Run tests with verbose output

```bash
go test ./... -v
```

Prints each test name as it executes and surfaces output from `t.Log()`. Useful when debugging failures or tracing execution order.

---

### Run a specific test function

```bash
go test ./service/account/... -v -run TestFunctionName
```

Prints each test name as it executes and surfaces output from `t.Log()`. Useful when debugging failures or tracing execution order.

---

### Run integration tests

```bash
go test -tags=integration ./service/operation -v
```

Includes files tagged with `integration`, which are excluded from normal test runs. Use this when tests depend on a real database, external services, or full application wiring.

To mark a file as integration-only, add this at the top:

```go
//go:build integration
// +build integration
```

---

### Run a specific test function

```bash
go test -tags=integration -v -run ^TestCreateUser$ ./service/operation
```

The `-run` flag accepts a regex. Using `^TestCreateUser$` ensures an exact match rather than a prefix match.

---

## Code Coverage

### Basic per-package coverage

```bash
go test ./... -cover
```

Displays a coverage percentage per package. Note that this only measures coverage within each package individually — it does not account for cross-package execution.

---

### Application-wide coverage (recommended)

```bash
go test ./... -coverpkg=./... -coverprofile=coverage.out
```

Instruments all packages and measures cross-package execution, producing a unified coverage profile. This is the correct approach for measuring true system-wide coverage, and should be used in CI pipelines.

---

### View coverage by function

```bash
go tool cover -func=coverage.out
```

Prints coverage per function with a total at the bottom:

```
service/operation/operation.go:45: InternalTransfer  87.5%
total: (statements) 78.3%
```

---

### View coverage as HTML

```bash
go tool cover -html=coverage.out
```

Opens an interactive browser report with covered lines highlighted in green and uncovered lines in red. Useful for identifying untested edge cases or reviewing critical code paths.

---

## Recommended Workflow

```bash
# During development — fast feedback
go test ./...

# Before merging — include integration tests
go test -tags=integration ./...

# Measure full system coverage
go test ./... -coverpkg=./... -coverprofile=coverage.out
go tool cover -func=coverage.out
```