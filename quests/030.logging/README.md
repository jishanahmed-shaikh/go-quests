# Logging: E-Commerce Search Tracking

## Concept

Imagine you get a vague error log in production that just says "failed to process". You have no idea _which_ user failed, _which_ request it was part of, or _what_ they were searching for. Structured logging solves this.

With the modern `log/slog` package, your logs are emitted as neat JSON objects instead of flat text. This makes it trivial for dashboards (like Datadog or Splunk) to search and filter logs.

You can do two powerful things to organize these JSON logs:

1. **Child Loggers (`logger.With()`)**: Think of this like taking a blank notebook and writing a specific `request_id` at the top of every single page. You derive a new logger from an existing one, permanently attaching specific fields. Any logs written by the child logger will automatically include those fields.
2. **Grouped Attributes (`slog.Group()`)**: Think of this like putting related papers into a manila folder before handing them over. Instead of polluting the top-level log with `user_id`, `user_name`, and `user_email`, you can nest them into a single `user` JSON object.

## References

- [Go by Example: Logging](https://gobyexample.com/logging)
- [pkg.go.dev/log/slog](https://pkg.go.dev/log/slog)

## Quest

### Objective

Implement search event logging for an E-Commerce site, handling empty queries, invalid users, and valid searches with structured log data.

### Requirements

Implement: `func ProcessSearch(logger *slog.Logger, reqID string, query string, userID int)`

1. **Child logger**: Derive from `logger` using `.With()`, attaching `"request_id": reqID`.
2. **Debug log**: Log `"processing search request"` at **Debug** level using the child logger.
3. **Empty query**: If `query == ""`, log `"empty search query"` at **Warn** level and return.
4. **Invalid userID**: If `userID < 0`, log `"invalid user ID"` at **Error** level and return.
5. **Valid search**: Log `"search executed"` at **Info** level with two attributes:
   - `"query"`: the query string.
   - A grouped user object: `slog.Group("user", slog.Int("id", userID))`.

### Inputs

- `logger`: A configured `*slog.Logger`.
- `reqID`: A unique string for the request, e.g. `"req-123"`.
- `query`: The search term, e.g. `"shoes"`.
- `userID`: The ID of the searching user, e.g. `42`.

### Outputs

- The function does not output or return any variables natively, but instead writes directly to the configured `logger` parameter.

### Example

```go
var buf bytes.Buffer
opts := &slog.HandlerOptions{Level: slog.LevelDebug}
logger := slog.New(slog.NewJSONHandler(&buf, opts))

ProcessSearch(logger, "req-999", "laptop", 10)
// The JSON output will look something like this (one per line):
// {"level":"DEBUG","msg":"processing search request","request_id":"req-999"}
// {"level":"INFO","msg":"search executed","request_id":"req-999","query":"laptop","user":{"id":10}}
```

## Tips

- To add fields to `logger.With`, pass alternating keys and values: `logger.With("key", "value")`.
- `slog.Group` expects a name and a list of attributes. `slog.Int("key", value)` creates a strongly typed integer attribute.

## Testing

```bash
go test -v ./quests/030.logging
```

Or from the quest directory:

```bash
go test -v
```

Expected output:

```text
=== RUN   TestProcessSearch
=== RUN   TestProcessSearch/success_valid_query
=== RUN   TestProcessSearch/fail_empty_query
=== RUN   TestProcessSearch/fail_invalid_user
--- PASS: TestProcessSearch (0.00s)
    --- PASS: TestProcessSearch/success_valid_query (0.00s)
    --- PASS: TestProcessSearch/fail_empty_query (0.00s)
    --- PASS: TestProcessSearch/fail_invalid_user (0.00s)
PASS
```
