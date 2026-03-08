# Context: The Smart File Downloader

## Concept

Think of Go's `context` as a "backpack" that gets passed down the chain from the very first function call (like handling an HTTP request) down to the very last Database query. It carries important request-scoped rules: deadlines, cancellation signals, and metadata (like trace IDs).

More importantly, Contexts are **hierarchical**. When you create a new context, you derive it from a `parent` context. If the parent gets cancelled, all its children (and their children) get cancelled automatically.

Understanding when to use which type of context is a core Go skill:

1. **`context.Background()`**: The root context. It is never cancelled and has no deadline. Use it at the very top level of an incoming request or main function.
2. **`context.TODO()`**: Identical to `Background()`, but signals to other developers: _"I don't know what context to use here yet, I'll figure it out later."_
3. **`context.WithTimeout(parent, duration)`**: Derives a new context from the given parent that automatically cancels itself after the duration expires. Perfect for network calls.
4. **`context.WithCancel(parent)`**: Derives a new context from the given parent and provides a `cancel()` function. You can manually call `cancel()` to signal workers to stop.
5. **`context.WithValue(parent, key, val)`**: Derives a new context carrying a key-value pair. Used for request-scoped data like trace IDs. (Note: standard Go practice requires using custom types for keys to avoid collisions).

## References

- [Go by Example: Context](https://gobyexample.com/context)
- [pkg.go.dev/context](https://pkg.go.dev/context)

## Quest

### Objective

Implement the core orchestration flow for a **Smart File Downloader** CLI that fetches large CDN assets, using all four context types across each step.

### Requirements

Implement: `func RunSmartDownloader(api *CDNAPI)`

Call the following `api` methods **in order**, each with the correct context:

1. **Root context**: Create with `context.Background()`.

2. **`api.LogActivity(ctx)`** — attach a trace ID for CDN request tracking:
   - Derive from root via `context.WithValue`, using key `TraceKey("trace_id")` and value `"smart-dl-123"`.

3. **`api.DNSLookup(ctx)`** — DNS must resolve within 100ms:
   - Derive from root via `context.WithTimeout(100 * time.Millisecond)`.
   - `defer cancel()` to avoid memory leaks.

4. **`api.DownloadChunks(ctx)`** — cancel connections once download completes:
   - Derive from root via `context.WithCancel`.
   - Immediately call `cancel()` to simulate early completion, then pass the cancelled context in.

5. **`api.VerifyChecksum(ctx)`** — legacy library with unknown cancellation support:
   - Pass `context.TODO()`.

### Inputs

- `api`: A pointer to a `CDNAPI` instance (simulated by the test suite).

### Outputs

- None (just execution side effects).

## Testing

```bash
go test -v ./quests/029.context
```

Or from the quest directory:

```bash
go test -v
```

Expected output:

```text
=== RUN   TestRunSmartDownloader
--- PASS: TestRunSmartDownloader (0.00s)
PASS
```
