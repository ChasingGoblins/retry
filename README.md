# retry: Production Retry Logic for Go

[![Release](https://img.shields.io/github/release/codeGROOVE-dev/retry.svg?style=flat-square)](https://github.com/codeGROOVE-dev/retry/releases/latest)
[![Software License](https://img.shields.io/badge/license-MIT-brightgreen.svg?style=flat-square)](LICENSE.md)
[![Go Report Card](https://goreportcard.com/badge/github.com/codeGROOVE-dev/retry?style=flat-square)](https://goreportcard.com/report/github.com/codeGROOVE-dev/retry)
[![Go Reference](https://pkg.go.dev/badge/github.com/codeGROOVE-dev/retry.svg)](https://pkg.go.dev/github.com/codeGROOVE-dev/retry)

Modern fork of [avast/retry-go/v4](https://github.com/avast/retry-go) focused on correctness, reliability and efficiency. 100% API-compatible drop-in replacement.

**Production guarantees:**
- Memory bounded: Max 1000 errors stored (configurable via maxErrors constant)
- No goroutine leaks: Uses caller's goroutine exclusively
- Integer overflow safe: Backoff capped at 2^62 to prevent wraparound
- Context-aware: Cancellation checked before each attempt
- No panics: All edge cases return errors
- Predictable jitter: Uses math/rand/v2 for consistent performance
- Zero allocations after init in success path

## Quick Start

### Installation

```bash
go get github.com/codeGROOVE-dev/retry
```

### Simple Retry

```go
// Retry a flaky operation up to 10 times (default)
err := retry.Do(func() error {
    return doSomethingFlaky()
})
```

### Retry with Custom Attempts

```go
// Retry up to 5 times with exponential backoff
err := retry.Do(
    func() error {
        resp, err := http.Get("https://api.example.com/data")
        if err != nil {
            return err
        }
        defer resp.Body.Close()
        return nil
    },
    retry.Attempts(5),
)
```

### Overly-complicated production configuration

```go

// Overly-complex production pattern: bounded retries with circuit breaking
err := retry.Do(
    func() error {
        return processPayment(ctx, req)
    },
    retry.Attempts(3),                        // Hard limit
    retry.Context(ctx),                       // Respect cancellation
    retry.MaxDelay(10*time.Second),           // Cap backoff
    retry.AttemptsForError(0, ErrRateLimit),  // Stop on rate limit
    retry.OnRetry(func(n uint, err error) {
        log.Printf("retry attempt %d: %v", n, err)
    }),
    retry.RetryIf(func(err error) bool {
        // Only retry on network errors
        var netErr net.Error
        return errors.As(err, &netErr) && netErr.Temporary()
    }),
)
```

### Preventing Cascading Failures

```go
// Stop retry storms with Unrecoverable
if errors.Is(err, context.DeadlineExceeded) {
    return retry.Unrecoverable(err) // Don't retry timeouts
}

// Per-error type limits prevent thundering herd
retry.AttemptsForError(0, ErrCircuitOpen)    // Fail fast on circuit breaker
retry.AttemptsForError(1, sql.ErrTxDone)     // One retry for tx errors
retry.AttemptsForError(5, ErrServiceUnavailable) // More retries for 503s
```

## Changes from avast/retry-go/v4

This fork will always be a 100% compatible drop-in replacement. There are some minor tweaks that have been made though:

New APIs added:

- `UntilSucceeded() Option` - Convenience wrapper for Attempts(0) (infinite retries)
- `FullJitterBackoffDelay() Strategy` - New delay type with full jitter exponential backoff
- `WrapContextErrorWithLastError() Option` - Wraps context errors with last function error
- `IfFunc Type` - New stutter-proof name (RetryIfFunc is now an alias)

Safety improvements:

- Memory bounded: Max 1000 errors (prevents OOM)
- Uses `math/rand/v2` (no lock contention)
- Overflow protection: Backoff capped at 2^62
- Enhanced validation and nil checks
- Better context cancellation with `context.Cause()`

## Documentation

- [API Docs](https://pkg.go.dev/github.com/codeGROOVE-dev/retry)
- [Examples](https://github.com/codeGROOVE-dev/retry/tree/master/examples)
- [Tests](https://github.com/codeGROOVE-dev/retry/tree/master/retry_test.go)
