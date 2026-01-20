# Research: Copilot CLI Server Mode Startup

## Overview

The Copilot SDK uses a client-server architecture where SDK implementations spawn and communicate with the Copilot CLI via JSON-RPC. The CLI's server mode is a built-in capability of the standard CLI, not a separate binary.

## Key Finding

The `--server` flag is a mode built into the normal Copilot CLI. Users install a single `copilot` CLI tool, and the SDKs launch it with appropriate flags to enable server mode.

## CLI Command Structure

```bash
copilot --server --log-level <level> [--stdio | --port <number>]
```

### Required Arguments

| Argument | Description |
|----------|-------------|
| `--server` | Enables server mode for JSON-RPC communication |
| `--log-level <level>` | Sets logging level (info, debug, error, etc.) |

### Transport Options (mutually exclusive)

| Argument | Description |
|----------|-------------|
| `--stdio` | Uses stdin/stdout for communication (default) |
| `--port <number>` | Uses TCP on specified port |

## Startup Flow

1. **Check for external server** - If `cliUrl` is configured, skip spawning and connect directly
2. **Build command arguments** - Construct args array with `--server` and transport options
3. **Spawn CLI process** - Execute CLI with configured environment and working directory
4. **Wait for ready signal** - For TCP mode, parse stdout for regex `listening on port (\d+)`
5. **Establish JSON-RPC connection** - Connect via stdio pipes or TCP socket
6. **Verify protocol version** - Ensure SDK and CLI are compatible

## Implementation Locations

| SDK | File | Method | Lines |
|-----|------|--------|-------|
| Node.js | `nodejs/src/client.ts` | `startCLIServer()` | 692-795 |
| Node.js | `nodejs/src/client.ts` | `connectToServer()` | 800-831 |
| Python | `python/copilot/client.py` | `_start_cli_server()` | 618-691 |
| Python | `python/copilot/client.py` | `_connect_via_stdio()` | 693-741 |
| Go | `go/client.go` | `startCLIServer()` | 773-873 |
| Go | `go/client.go` | `connectViaTcp()` | 877-885 |
| .NET | `dotnet/src/Client.cs` | `StartCliServer()` | 579-662 |
| .NET | `dotnet/src/Client.cs` | `ConnectViaStdio()` | 684-714 |

## Configuration Options

Common options across all SDK implementations:

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `cliPath` | string | `"copilot"` | Path to CLI executable |
| `cliArgs` | string[] | `[]` | Additional arguments to pass to CLI |
| `port` | number | `0` | TCP port (0 = use stdio instead) |
| `useStdio` | boolean | `true` | Use stdio transport vs TCP |
| `cliUrl` | string | `undefined` | URL of external server (skips spawning) |
| `logLevel` | string | `"info"` | CLI log level |
| `autoStart` | boolean | `true` | Automatically start CLI on client start |
| `autoRestart` | boolean | `true` | Restart CLI if it crashes |
| `cwd` | string | current dir | Working directory for CLI process |
| `env` | object | `process.env` | Environment variables for CLI process |

Option naming varies by SDK convention:
- Node.js: camelCase (`cliPath`, `useStdio`)
- Python: snake_case (`cli_path`, `use_stdio`)
- Go/.NET: PascalCase (`CLIPath`, `UseStdio`)

Environment variable override: `COPILOT_CLI_PATH`

## Transport Mechanisms

### stdio Mode (Default)

- CLI spawned with `--stdio` flag
- Communication via stdin/stdout pipes
- Uses StreamMessageReader/StreamMessageWriter (Node.js) or equivalent
- Ready immediately after process spawn

### TCP Mode

- CLI spawned with `--port <number>` flag
- Must wait for stdout message: `listening on port <number>`
- Timeout: 10 seconds (30 seconds in .NET)
- Uses socket-based streams with header-delimited message format

## JSON-RPC Protocol

- **Protocol version**: JSON-RPC 2.0
- **Message format**: Header-delimited (Content-Length header followed by JSON body)
- **Bidirectional**: Supports requests, responses, and notifications in both directions

## Platform-Specific Handling

### Windows
- Uses `cmd /c` to resolve executables when not using absolute paths
- Handles `.cmd` and `.bat` extensions

### Node.js Files
- Files ending in `.js` are spawned with `node` command

### Environment Sanitization
- `NODE_DEBUG` explicitly removed from environment to prevent output pollution

## Process Lifecycle

### Startup
1. Validate configuration
2. Spawn process with appropriate arguments
3. Establish transport connection
4. Initialize JSON-RPC client
5. Register notification/request handlers
6. Verify protocol compatibility

### Shutdown (Graceful)
1. Destroy all active sessions
2. Close JSON-RPC connection
3. Terminate CLI process
4. Clean up resources

### Shutdown (Force)
1. Clear sessions immediately
2. Forcefully kill process
3. Clean up resources

### Auto-Restart
- Triggered when CLI process exits unexpectedly
- Only if `autoRestart` is enabled
- Only after successful initial connection (not during startup failures)

## Error Handling

| Error | Handling |
|-------|----------|
| CLI not found | Throw error with path information |
| Startup timeout | Throw timeout error (10-30s depending on SDK) |
| Connection failure | Retry or throw based on configuration |
| Process crash | Auto-restart if enabled, otherwise propagate error |
| Protocol mismatch | Throw version incompatibility error |

## References

- Main README: `/home/user/copilot-sdk/README.md`
- Types definition: `/home/user/copilot-sdk/nodejs/src/types.ts:16-77`
- Architecture diagram: `/home/user/copilot-sdk/README.md:28-40`
