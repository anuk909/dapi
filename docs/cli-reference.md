# CLI Reference

## Commands

| Command | Description |
|---------|-------------|
| `start <script> [options]` | Start a debug session |
| `attach --pid <PID> [options]` | Attach to a running process by PID |
| `attach [host:]port [options]` | Attach to an existing debug server |
| `step [over\|into\|out]` | Step (default: over) |
| `continue` | Resume execution / wait for next breakpoint |
| `context` | Re-fetch auto-context without stepping |
| `eval <expression>` | Evaluate in current frame |
| `vars` | List local variables |
| `stack` | Show call stack |
| `output` | Drain buffered stdout/stderr since last stop |
| `break <file:line[:cond]>` | Add breakpoint mid-session |
| `source [file] [line]` | Show source around current line |
| `status` | Show session state |
| `close` | End the debug session |

## start options

```
--break, -b <file:line[:condition]>    Set a breakpoint (repeatable)
--runtime <path>                       Path to language runtime (e.g. /path/to/venv/python)
--break-on-exception <filter>          Stop on exceptions (repeatable)
--stop-on-entry                        Pause on the first line
--args <...>                           Arguments for the debugged script
--session <name>                       Session name (default: "default")
```

## attach options

```
--pid <PID>                            Attach by PID (injects debugpy via lldb/gdb)
--break, -b <file:line[:condition]>    Set a breakpoint (repeatable)
--break-on-exception <filter>          Stop on exceptions (repeatable)
--runtime <path>                       Language runtime path (optional)
--language <name>                      Adapter: python, node, go, rust (default: python)
--session <name>                       Session name
```

## Multi-Session

Run independent debug sessions in parallel using `--session <name>`:

```bash
dapi --session api    start api.py    --break routes.py:42
dapi --session worker start worker.py --break tasks.py:88

dapi --session api    eval "request.user"
dapi --session worker eval "queue.size()"

dapi --session api    close
dapi --session worker close
```

Each session has its own daemon at `~/.dapi/<name>.sock`.

## Auto-Context

Every execution command (`start`, `step`, `continue`, `context`) returns full context in one response:

```
paused at compute() · app.py:41 [breakpoint]

    37 │ def compute(items):
    38 │     result = None
    39 │     for item in items:
    40 │         result += item
→   41 │     return result

Locals:
  items = [1, 2, 3]  (list)
  result = None  (NoneType)

Stack:
  compute [app.py:41]
  main [app.py:10]

Output:
  Processing batch...
```

No follow-up `vars`, `stack`, or `source` calls needed.
