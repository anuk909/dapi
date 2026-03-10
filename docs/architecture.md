# Architecture

```
CLI (stateless)  ──unix socket──▶  Daemon  ──TCP/DAP──▶  Debug Adapter
                                   ~/.dapi/<session>.sock   (debugpy, dlv, etc.)
```

**Attach by PID:**

```
CLI  ──▶  Daemon  ──lldb/gdb──▶  Target Process (injects debugpy.listen())
                  ──TCP/DAP──▶   debugpy adapter (spawned by debugpy.listen)
```

- **CLI** (`dapi`): Stateless. Sends JSON commands over a Unix socket, prints results.
- **Daemon**: Background process per session. Manages DAP session, buffers output.
- **Debug Adapter**: Language-specific process (debugpy, dlv, js-debug, CodeLLDB).

The daemon starts automatically on the first command and exits when the session closes.

## Development

```bash
git clone https://github.com/anuk909/dapi
cd dapi
bun install
bun test
```
