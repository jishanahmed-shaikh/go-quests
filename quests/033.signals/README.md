# Signals: The Process Supervisor

## Concept

When you run a background server or worker (a "daemon") in production, the operating system communicates with it using Signals. In modern Go, graceful shutdown is typically handled via Contexts, while other control signals (like reloading configuration) are handled manually.

Additionally, many Go programs act as **supervisors** that spin up background child processes. When the supervisor is told to shut down, it should politely tell its child processes to shut down securely, by explicitly sending a term signal, rather than forcefully killing them.

- **`signal.NotifyContext(ctx, os.Interrupt)`**: Modern Go way to handle `SIGINT` (Ctrl+C). It returns a new Context that is automatically cancelled when `os.Interrupt` is received.
- **`signal.Notify(c, syscall.SIGHUP, ...)`**: The classic way to subscribe to non-termination signals.
- **`cmd.Process.Signal(syscall.SIGTERM)`**: Used to send a signal to a running child process.

## Quest

### Objective

You are building the master control loop for the Multiplayer Game Server itself (following the initialization step from the previous quest). The server acts as a supervisor that spins up a background database backup worker (`workerCmd`). Your server must handle `SIGINT` gracefully to shut down the worker, and handle `SIGHUP` and `SIGUSR1` for manual administrative commands.

## `RunGameServer(ready chan bool, workerCmd string) error`

Starts a managed game server worker with graceful shutdown and signal handling.

### Steps

| #   | What to do                                                                                                                          |
| --- | ----------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Create a shutdown context with `signal.NotifyContext(context.Background(), os.Interrupt / syscall.SIGINT)` — defer its `stop()`     |
| 2   | Create a buffered signal channel (`make(chan os.Signal, 1)`) and register it for `SIGHUP` + `SIGUSR1` via `signal.Notify`           |
| 3   | Prepare the child process with `exec.Command(workerCmd)` _(use `Command`, not `CommandContext` — shutdown is manual via `SIGTERM`)_ |
| 4   | Start the child process with `cmd.Start()` — return error immediately if it fails                                                   |
| 5   | Signal readiness: `ready <- true`                                                                                                   |
| 6   | Run the event loop below ↓                                                                                                          |

### Event Loop — `for { select { ... } }`

| Signal / Event | Action                                                                                                                                                               |
| -------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `SIGHUP`       | Print `"Reloading game rules..."` → continue                                                                                                                         |
| `SIGUSR1`      | Print `"Dumping player coordinates..."` → continue                                                                                                                   |
| `ctx.Done()`   | Print `"Shutting down worker..."` → send `SIGTERM` via `cmd.Process.Signal(syscall.SIGTERM)` → call `cmd.Wait()` → return its result _(even if it's an `ExitError`)_ |

### Inputs

- `ready`: A `chan bool` that you must send `true` to once `cmd.Start()` has succeeded.
- `workerCmd`: The executable name or path of the worker command.

### Outputs

- `error`: Start errors or the final wait error, otherwise `nil`.

### Example

```go
// In tests, we will send signals to your server Process.
ready := make(chan bool)
go RunGameServer(ready, "sleep 100")
<-ready // Server running and child started!

// If someone runs `kill -HUP <pid>`:
// Output: Reloading game rules...

// If someone runs `kill -INT <pid>` (or presses Ctrl+C):
// Output: Shutting down worker...
// (And the function gracefully terminates the child and returns nil)
```

## Testing

```bash
go test -v ./quests/033.signals
```

Or from the quest directory:

```bash
go test -v
```

Expected output:

```text
=== RUN   TestRunGameServer
    solution_test.go:60: Process exited with wait error (acceptable if killed): signal: terminated
--- PASS: TestRunGameServer (0.21s)
PASS
```
