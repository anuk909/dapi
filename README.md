# agent-debugger

CLI debugger for AI agents. Set breakpoints, inspect variables, evaluate expressions, and step through code — in Python, JavaScript, Go, Rust, C, and C++.

Built on the [Debug Adapter Protocol](https://microsoft.github.io/debug-adapter-protocol/) (DAP), the same protocol that powers VS Code's debugger. One CLI, multiple language backends.

## Install

```bash
npm install -g agent-debugger
```

Requires Node.js >= 18.

## Quick Start

```bash
# Start a debug session, paused at line 25
agent-debugger start app.py --break app.py:25

# Inspect variables at the breakpoint
agent-debugger vars

# Evaluate any expression in the current scope
agent-debugger eval "type(data['age'])"

# Continue to the next breakpoint
agent-debugger continue

# Done
agent-debugger close
```

## Commands

| Command | Description |
|---------|-------------|
| `start <script> [options]` | Start a debug session |
| `attach [host:]port [options]` | Attach to a running debug server |
| `vars` | List local variables in the current frame |
| `eval <expression>` | Evaluate an expression in the current scope |
| `step [into\|out]` | Step over, into a function, or out of a function |
| `continue` | Resume execution / wait for next breakpoint |
| `stack` | Show the call stack |
| `break <file:line[:condition]>` | Add a breakpoint mid-session |
| `source [file] [line]` | Show source code around the current line |
| `status` | Show session state and current location |
| `close` | Detach or end the debug session |

### Start Options

```bash
agent-debugger start <script> [options]

Options:
  --break, -b <file:line[:condition]>   Set a breakpoint (repeatable)
  --runtime <path>                      Path to language runtime (e.g. python, node)
  --stop-on-entry                       Pause on the first line
  --args <...>                          Arguments to pass to the script
```

### Attach Options

```bash
agent-debugger attach [host:]port [options]

Options:
  --break, -b <file:line[:condition]>   Set a breakpoint (repeatable)
  --language <name>                     Language adapter (default: python)
```

### Attaching to a Running Server

Debug a server (uvicorn, Flask, FastAPI, etc.) without restarting it:

```bash
# Terminal 1: Start your server with debugpy listening
python -m debugpy --listen 5678 --wait-for-client -m uvicorn app:main

# Terminal 2: Attach and set breakpoints
agent-debugger attach 5678 --break routes.py:42

# Terminal 3: Make a request to trigger the breakpoint
curl localhost:8000/api/endpoint

# Terminal 2: Wait for the hit, then inspect
agent-debugger continue
agent-debugger vars
agent-debugger eval "request.body"
agent-debugger close          # detaches without killing the server
```

You can also embed debugpy in your code:
```python
import debugpy
debugpy.listen(5678)
# debugpy.wait_for_client()  # uncomment to pause until debugger attaches
```

### Breakpoints

Multiple breakpoints and conditional breakpoints are supported:

```bash
# Multiple breakpoints
agent-debugger start app.py --break app.py:25 --break app.py:40

# Conditional breakpoint — only pause when the condition is true
agent-debugger start app.py --break "app.py:30:i == 50"

# Add a breakpoint to a running session
agent-debugger break app.py:60
```

## Supported Languages

| Language | Extensions | Debug Adapter | Setup |
|----------|------------|---------------|-------|
| Python | `.py` | [debugpy](https://github.com/microsoft/debugpy) | `pip install debugpy` |
| JavaScript | `.js`, `.mjs`, `.cjs` | @vscode/js-debug | VS Code installed, or `JS_DEBUG_PATH` env var |
| TypeScript | `.ts`, `.mts`, `.tsx` | @vscode/js-debug | Same as JavaScript |
| Go | `.go` | Delve | `go install github.com/go-delve/delve/cmd/dlv@latest` |
| Rust | `.rs` | CodeLLDB | `CODELLDB_PATH` env var |
| C/C++ | `.c`, `.cpp`, `.cc` | CodeLLDB | Same as Rust |

### Language-specific setup

**Python** — install debugpy in the environment you want to debug:
```bash
pip install debugpy

# Use a specific Python interpreter
agent-debugger start app.py --break app.py:10 --runtime /path/to/venv/bin/python
```

**JavaScript/TypeScript** — requires VS Code's js-debug extension, which ships with any VS Code install. The adapter auto-detects it from `~/.vscode/extensions/`. To use a custom location:
```bash
export JS_DEBUG_PATH=/path/to/ms-vscode.js-debug-x.x.x
```

**Go** — install Delve:
```bash
go install github.com/go-delve/delve/cmd/dlv@latest
```

**Rust/C/C++** — set the path to the CodeLLDB adapter binary:
```bash
export CODELLDB_PATH=/path/to/codelldb/adapter/codelldb
```

## How It Works

**Launch mode** (`start`) — the daemon spawns the debug adapter:
```
CLI (stateless)  ──unix socket──▶  Daemon (session state)  ──TCP/DAP──▶  Debug Adapter
                                                                          (debugpy, dlv, etc.)
```

**Attach mode** (`attach`) — the daemon connects to an existing debug server:
```
CLI (stateless)  ──unix socket──▶  Daemon (session state)  ──TCP/DAP──▶  Your Server
                                                                          (with debugpy listening)
```

- **CLI** (`agent-debugger`): Stateless client. Parses arguments, sends JSON commands over a Unix socket, prints results.
- **Daemon**: Background process that manages the debug session. Spawns or connects to a debug adapter via DAP, and translates CLI commands into DAP requests.
- **Debug Adapter**: Language-specific process (debugpy, Delve, js-debug, CodeLLDB) that implements the Debug Adapter Protocol.

The daemon starts automatically on the first command and shuts down when the session closes. Only one debug session runs at a time.