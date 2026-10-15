# VHAgent

VHAgent is a command-line bridge between AI models and a running VoidHexed instance. It connects over a local named pipe and exposes VoidHexed's full feature set through structured JSON, allowing AI agents to query state, execute commands, stream events, and manage the C2 server programmatically.

## Prerequisites

- VoidHexed must be running (it hosts the IPC server)
- VHAgent must be on the same machine
- No configuration needed — transport is automatic

**Transport:** Named pipe `VoidHexed.Agent`
- Windows: standard named pipe
- Linux: Unix domain socket at `/tmp/CoreFxPipe_VoidHexed.Agent`

## How AI Uses VHAgent

An AI model (Claude, GPT, etc.) invokes VHAgent as a subprocess. It sends a command, reads the JSON response from stdout, and acts on the result. All human-readable text (prompts, progress) goes to stderr so it never pollutes the JSON stream.

Typical AI flow:
```
AI decides what to do
  -> spawns: VHAgent list-clients
  -> reads stdout JSON: {"ok":true,"data":{"count":3,"clients":[...]}}
  -> parses the client list
  -> spawns: VHAgent exec "select 192.168.1.10" --wait 0
  -> spawns: VHAgent exec "cpu"
  -> reads the captured command output (includes async client response)
  -> reasons about results
```

The AI never interacts with VoidHexed's GUI — VHAgent is its sole interface. The `exec` command is the most important: it can run **any** command that works in VoidHexed's command bar, and returns the captured log output as a JSON array.

Commands sent through VHAgent are tagged with an `[AI]` prefix in VoidHexed's UI log (e.g., `[12:00:41] [AI] > cpu`) so the operator can distinguish AI-issued commands from manual ones.

## Three Operating Modes

### 1. CLI Shorthand (default)

Single command in, JSON out, process exits. This is what AI agents use.

```bash
VHAgent ping
VHAgent list-clients
VHAgent exec "select 192.168.1.10" --wait 0   # sync command, no wait needed
VHAgent exec "cpu"                              # waits up to 5s for client response
VHAgent exec "cpu" --wait 10000                 # custom wait: 10 seconds
VHAgent stats
VHAgent db events --filter 192.168 --limit 20
```

Exit codes: `0` = success, `1` = connection error, `2` = command error.

### 2. Interactive REPL (`-i`)

Persistent connection with a prompt. **Auto-subscribes to `log` and `server_log` events on connect**, so shell output and command results stream in real-time between prompts — no manual `subscribe` needed. Useful for humans exploring the API, debugging, or watching live output from shell sessions.

```
$ VHAgent -i
VHAgent interactive mode (subscribed to log, server_log). Type 'help' for commands, 'quit' to exit.
vhagent> ping
{"ok":true,"data":{"version":"0.9.5.0","uptime":"1h 05m 22s",...}}
vhagent> list-clients
{"ok":true,"data":{"count":2,"clients":[...]}}
vhagent> exec "select 192.168.1.10:54321" --wait 0
{"ok":true,...}
vhagent> exec cpu
{"ok":true,"data":{"output":["[14:32:15] [AI] > cpu"],"durationMs":1250}}
  ... async CPU result streams as a log event ...
{"type":"event","event":"log","data":{"message":"[14:32:15] CPU: Intel Core i7-12700K @ 3.60GHz","source":"command"}}
vhagent> quit
```

Commands in the REPL are the same as CLI shorthand (without the `VHAgent` prefix). Type `help` for the full list or `quit`/`exit` to disconnect.

### 3. Stdin JSON Pipe

For programmatic use when the AI needs full control over the request structure. Send one JSON object per line to stdin; each gets a JSON response line on stdout.

```bash
echo '{"method":"list_clients"}' | VHAgent
echo '{"method":"exec","params":{"command":"cpu","wait_ms":5000}}' | VHAgent
```

Multiple requests can be piped in sequence (one per line). This mode is useful when the AI constructs requests dynamically rather than using the CLI shorthand.

## Command Reference

### State Queries

| Command | Description |
|---------|-------------|
| `ping` | Server version, uptime, session start |
| `list-clients` | All connected clients (IP, HWID, OS, user, country, privilege, platform) |
| `list-selected` | Currently selected clients |
| `list-servers` | Active listeners (port, label, client count) |
| `stats` | Bytes sent/recv, connection counts, OS distribution, rates, peak, uptime |
| `settings` | Certificates, auto-tasks, theme, firewall policy, heartbeat, rate limits |
| `plugins` | Loaded plugins (id, version, status, available binary formats) |
| `firewall` | Firewall policy + all rules with hit counts |

### Command Execution

```bash
VHAgent exec "<any VoidHexed command>" [--wait ms] [--idle ms]
```

This routes through `MainUIArg.Parser()` — the same parser used by the UI command bar. The response contains an `output` array of captured log lines (timestamped) and a `durationMs` field.

**Async result capture:** After the command is dispatched, `exec` waits for async results. Two timeouts control this:

- **`--wait`** (default: 5000ms) — hard deadline: maximum total time to wait.
- **`--idle`** (default: 1000ms) — idle window: if no new output arrives within this window, `exec` returns early. Each received line resets the idle timer.

Use `--wait 0` for sync-only commands (e.g., `select`, `server start`). For **bursty commands** like `shellcmd ping` (output in ~1-second intervals), increase `--idle` to avoid cutting off between bursts.

**`[AI]` prefix:** Commands sent through VHAgent are logged in the VoidHexed UI with an `[AI]` tag (e.g., `[12:00:41] [AI] > cpu`) so the operator can distinguish them from manually typed commands.

Examples:
```bash
VHAgent exec "select 192.168.1.10:54321" --wait 0  # Select a client (sync, no wait)
VHAgent exec "cpu"                                  # Query CPU info (waits for response)
VHAgent exec "module"                               # List loaded modules
VHAgent exec "shell"                                # Open remote CMD shell
VHAgent exec "shellcmd ipconfig"                    # Run command in active shell
VHAgent exec "shellcmd ping 1.2.3.4" --wait 15000 --idle 2000  # Bursty output
VHAgent exec "server start MyServer 4443"           # Start a listener
VHAgent exec "build 192.168.1.1 4443 myclient w"   # Build a Windows payload
```

### Database Queries

| Command | Flags | Description |
|---------|-------|-------------|
| `db stats` | — | Row counts per table |
| `db events` | `--filter X`, `--limit N` | Connection event history |
| `db builds` | `--limit N` | Build history |
| `db stealer` | `--ip X`, `--limit N` | Stealer results |
| `db hits` | `--rule X`, `--limit N` | Firewall hit log |
| `db clear <table>` | — | Clear a table |

Valid table names for `db clear`: `events`, `hits`, `builds`, `stealer`, `asm`.

Default `--limit` is 50.

### Event Streaming

```bash
VHAgent subscribe client_connected,client_disconnected,log,server_log
```

Subscribes to real-time events and streams them as JSON lines on stdout until Ctrl+C. Available events:

| Event | Fires when | Key fields |
|-------|-----------|------------|
| `client_connected` | Client authenticates | ip, hwid, os, username, country, privilege, platform, serverPort |
| `client_disconnected` | Client drops | ip, hwid |
| `log` | Command output is logged | message, source |
| `server_log` | Server-level log entry | message |

## Output Format

All responses are JSON on stdout:

```json
{"ok":true,"data":{...}}
```

Errors:
```json
{"ok":false,"error":"Missing 'command' parameter"}
```

Events (after subscribing):
```json
{"type":"event","event":"client_connected","data":{"ip":"192.168.1.10",...}}
```

Human-readable text (prompts, usage help, progress messages) goes to stderr only.

## Architecture

```
AI Model / Human
      |
      v
  VHAgent (CLI, AOT-compiled, net10.0)
      |  named pipe: VoidHexed.Agent
      v
  AgentIpcServer (inside VoidHexed process)
      |
      +---> AgentIpcHandler
              |
              +---> MainUIArg.Parser()    [command execution]
              +---> MainUI._clients       [client state]
              +---> ServerStats            [network metrics]
              +---> VoidDb                 [SQLite queries]
              +---> PluginRegistry         [plugin state]
              +---> Program.GlobalObject   [servers, settings]
```

VHAgent is a thin client. All logic runs inside VoidHexed. Synchronous log output is captured via `AsyncLocal<List<string>>` in the logging path. Asynchronous results (e.g., client responses arriving after command dispatch) are captured via a global `Channel<string>` with a configurable idle timeout (`idle_ms`, default 1s), so `exec` calls return both immediate and delayed output.

## Limitations

- **Single connection:** Only one VHAgent instance can connect at a time. A new connection displaces the previous one.
- **Selection state is global:** `exec "select ..."` changes the selection for the entire VoidHexed instance, including what the UI shows. The AI and the human operator share this state.
- **`proxy recon` is Windows-only:** `proxy start/stop/list/connections` work on both Windows and Linux clients. Only `proxy recon` requires a Windows client (uses Registry + iphlpapi.dll).
- **No binary data over IPC:** File transfer results, screenshots, etc. are not streamed through VHAgent. Use `exec` to trigger them; results appear in VoidHexed's UI/filesystem.

## Building

VHAgent is a separate console project targeting `net10.0` with AOT compilation:

```bash
dotnet publish VHAgent -c Release
```

The output is a single native executable with no runtime dependency.
