# samba-common-bin — Samba client and administration utilities

- **Version:** 4.22.11-Debian-4.22.11+dfsg-0+deb13u1
- **Package:** `samba-common-bin`
- **Binaries:** `/usr/bin/net`, `/usr/bin/smbpasswd`, `/usr/bin/testparm`,
  `/usr/bin/smbcontrol`, `/usr/bin/nmblookup`, `/usr/bin/dbwrap_tool`,
  `/usr/bin/winexe`, `/usr/bin/samba-regedit`, `/usr/bin/samba-log-parser`
- **Help snapshots:** [`help-net.txt`](help-net.txt) plus
  `help-net-<subcommand>.txt` and `help-<tool>.txt` in this directory
- **Upstream:** https://www.samba.org/
- **Licence:** GPL-3.0-or-later

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it provides

`samba-common-bin` is the client and administration half of Samba. It ships the
multi-transport `net` tool, the password changer, the configuration checker,
the daemon message bus client, a NetBIOS name lookup tool, a tdb inspector, a
remote Windows command runner, a registry editor, and a debug-log parser.

| Tool | Purpose |
|------|---------|
| `net` | Local and remote administration over RPC, RAP, ADS, and local tdb |
| `smbpasswd` | Change local or remote SMB passwords |
| `testparm` | Validate `smb.conf` and print effective parameters |
| `smbcontrol` | Send control messages to `smbd`, `nmbd`, `winbindd`, or a PID |
| `nmblookup` | Resolve NetBIOS names and node status |
| `dbwrap_tool` | Read and write Samba tdb databases |
| `winexe` | Run a command on a remote Windows host over SMB |
| `samba-regedit` | Browse and edit a remote registry over SMB |
| `samba-log-parser` | Parse Samba debug logs into traces and flows |

## `net` — multi-transport administration

`net` is the main administration interface. Its grammar is:

```text
net [<method>] <command> [<subcommand>] [options] [target]
```

`<method>` is one of `rpc`, `rap`, `ads`, or omitted. When omitted, `net`
chooses a transport based on the command: local tdb operations run in-process,
while most account and share operations default to RPC.

### Local command families

These operate on the local host, its tdbs, or its configuration:

| Family | Subcommands |
|--------|-------------|
| `status` | `sessions [parseable]`, `shares [parseable]` |
| `conf` | `list`, `import`, `listshares`, `drop`, `showshare`, `addshare`, `delshare`, `setparm`, `getparm`, `delparm`, `getincludes`, `setincludes`, `delincludes` |
| `registry` | `enumerate`, `enumerate_recursive`, `createkey`, `deletekey`, `deletekey_recursive`, `getvalue`, `getvalueraw`, `getvaluesraw`, `setvalue`, `increment`, `deletevalue`, `getsd`, `getsd_sddl`, `setsd_sddl`, `import`, `export`, `convert`, `check` |
| `serverid` | `wipedbs`, `exists` |
| `tdb` | `locking` |
| `cache` | `add`, `del`, `get`, `search`, `list`, `flush`, `samlogon` |
| SID/RID | `getlocalsid`, `setlocalsid`, `getdomainsid`, `setdomainsid`, `maxrid` |
| `time` | `system`, `set`, `zone` |
| `lookup` | `host`, `ldap`, `kdc`, `pdc`, `dc`, `master`, `name`, `sid`, `dsgetdcname` |
| `printing` | `dump`, `migrate` |
| `eventlog` | `dump`, `import`, `export` |
| `usershare` | `list`, `add`, `delete`, `info` |
| `groupmap` | `add`, `modify`, `delete`, `set`, `cleanup`, `addmem`, `delmem`, `listmem`, `memberships`, `list` |
| `sam` | `createbuiltingroup`, `createlocalgroup`, `createdomaingroup`, `deletelocalgroup`, `deletedomaingroup`, `mapunixgroup`, `unmapunixgroup`, `addmem`, `delmem`, `listmem`, `list`, `show`, `set`, `policy`, `rights`, `privison` |
| `idmap` | `dump`, `restore`, `get`, `set`, `delete`, `check` |
| `g_lock` | `do`, `locks`, `dump`, `dumpall` |
| `notify` | `listen`, `trigger` |
| `witness` | `list`, `client-move`, `share-move`, `force-unregister`, `force-response` |
| `vfs` | `getntacl`, `stream2adouble` |

### Remote command families

| Family | Subcommands |
|--------|-------------|
| `rpc` | `audit`, `info`, `join`, `oldjoin`, `testjoin`, `user`, `password`, `group`, `share`, `file`, `printer`, `changetrustpw`, `trustdom`, `abortshutdown`, `shutdown`, `vampire`, `getsid`, `rights`, `service`, `registry`, `shell`, `trust`, `conf` |
| `rap` | `file`, `share`, `session`, `server`, `domain`, `printq`, `user`, `group`, `validate`, `groupmember`, `admin`, `service`, `password` |
| `ads` | `info`, `join`, `testjoin`, `leave`, `status`, `user`, `group`, `dns`, `password`, `changetrustpw`, `printer`, `search`, `dn`, `sid`, `workgroup`, `lookup`, `keytab`, `spnset`, `gpo`, `kerberos`, `enctypes` |
| `dom` | `join`, `unjoin`, `renamecomputer` |
| `offlinejoin` | `provision`, `requestodj`, `composeodj` |
| top level | `join`, `primarytrust dumpinfo`, `changetrustpw` |

### Help snapshots

| Command | Snapshot |
|---------|----------|
| `net` | [`help-net.txt`](help-net.txt) |
| `net rpc` | [`help-net-rpc.txt`](help-net-rpc.txt) |
| `net rap` | [`help-net-rap.txt`](help-net-rap.txt) |
| `net ads` | [`help-net-ads.txt`](help-net-ads.txt) |
| `net file` | [`help-net-file.txt`](help-net-file.txt) |
| `net share` | [`help-net-share.txt`](help-net-share.txt) |
| `net session` | [`help-net-session.txt`](help-net-session.txt) |
| `net server` | [`help-net-server.txt`](help-net-server.txt) |
| `net domain` | [`help-net-domain.txt`](help-net-domain.txt) |
| `net printq` | [`help-net-printq.txt`](help-net-printq.txt) |
| `net user` | [`help-net-user.txt`](help-net-user.txt) |
| `net group` | [`help-net-group.txt`](help-net-group.txt) |
| `net groupmap` | [`help-net-groupmap.txt`](help-net-groupmap.txt) |
| `net sam` | [`help-net-sam.txt`](help-net-sam.txt) |
| `net validate` | [`help-net-validate.txt`](help-net-validate.txt) |
| `net groupmember` | [`help-net-groupmember.txt`](help-net-groupmember.txt) |
| `net admin` | [`help-net-admin.txt`](help-net-admin.txt) |
| `net service` | [`help-net-service.txt`](help-net-service.txt) |
| `net password` | [`help-net-password.txt`](help-net-password.txt) |
| `net primarytrust` | [`help-net-primarytrust.txt`](help-net-primarytrust.txt) |
| `net changetrustpw` | [`help-net-changetrustpw.txt`](help-net-changetrustpw.txt) |
| `net time` | [`help-net-time.txt`](help-net-time.txt) |
| `net lookup` | [`help-net-lookup.txt`](help-net-lookup.txt) |
| `net g_lock` | [`help-net-g_lock.txt`](help-net-g_lock.txt) |
| `net join` | [`help-net-join.txt`](help-net-join.txt) |
| `net offlinejoin` | [`help-net-offlinejoin.txt`](help-net-offlinejoin.txt) |
| `net dom` | [`help-net-dom.txt`](help-net-dom.txt) |
| `net cache` | [`help-net-cache.txt`](help-net-cache.txt) |
| `net idmap` | [`help-net-idmap.txt`](help-net-idmap.txt) |
| `net status` | [`help-net-status.txt`](help-net-status.txt) |
| `net usershare` | [`help-net-usershare.txt`](help-net-usershare.txt) |
| `net usersidlist` | [`help-net-usersidlist.txt`](help-net-usersidlist.txt) |
| `net conf` | [`help-net-conf.txt`](help-net-conf.txt) |
| `net registry` | [`help-net-registry.txt`](help-net-registry.txt) |
| `net eventlog` | [`help-net-eventlog.txt`](help-net-eventlog.txt) |
| `net printing` | [`help-net-printing.txt`](help-net-printing.txt) |
| `net serverid` | [`help-net-serverid.txt`](help-net-serverid.txt) |
| `net notify` | [`help-net-notify.txt`](help-net-notify.txt) |
| `net tdb` | [`help-net-tdb.txt`](help-net-tdb.txt) |
| `net vfs` | [`help-net-vfs.txt`](help-net-vfs.txt) |
| `net witness` | [`help-net-witness.txt`](help-net-witness.txt) |

### Common options

Availability varies by command; `net help <command>` shows the accepted set.

| Option | Meaning |
|--------|---------|
| `-S, --server=SERVER` | Target server |
| `-I, --ipaddress=ADDR` | Target IP address |
| `-w, --target-workgroup=WG` | Target workgroup or domain |
| `-p, --port=PORT` | Connection port |
| `--myname=NAME` | Client name |
| `--long` | Full information |
| `-U, --user=[DOMAIN/]USER[%PASS]` | Credentials |
| `-N, --no-pass` | Do not prompt for a password |
| `--password=STRING` | Password on the command line |
| `--pw-nt-hash` | Treat the password as an NT hash |
| `-A, --authentication-file=FILE` | Credentials file |
| `-P, --machine-pass` | Machine account password |
| `-W, --workgroup=WG` | Workgroup or domain |
| `--realm=REALM` | Kerberos realm |
| `-R, --name-resolve=ORDER` | Name resolution order |
| `-O, --socket-options=OPTS` | Socket options |
| `-m, --max-protocol=PROTO` | Maximum SMB protocol |
| `-n, --netbiosname=NAME` | Local NetBIOS name |
| `--netbios-scope=SCOPE` | NetBIOS scope |
| `-s, --configfile=FILE` | Alternative `smb.conf` |
| `--option=name=value` | Override one `smb.conf` option |
| `-l, --log-basename=BASE` | Log basename |
| `-d, --debuglevel=LEVEL` | Debug level |
| `--debug-stdout` | Debug to stdout |
| `--leak-report`, `--leak-report-full` | Talloc leak report |
| `-V, --version` | Print version |

### Recipes

**1. Show local sessions and shares.**

```bash
net status sessions
net status shares parseable
```

**2. Show the local SID and domain SID.**

```bash
net getlocalsid
net getdomainsid
```

**3. Enumerate the local Samba registry.**

```bash
net registry enumerate 'HKLM\Software\Samba'
net registry enumerate_recursive 'HKLM\Software\Samba'
```

**4. Dump effective configuration and one parameter.**

```bash
net conf list
net conf getparm <share> 'path'
```

**5. Look up a host or domain controller.**

```bash
net lookup host fileserver.corp.example
net lookup dc corp.example
net lookup pdc corp.example
```

**6. List local SAM users and groups.**

```bash
net sam list users
net sam list groups
net sam list localgroups
```

**7. Remote user and group enumeration over RPC.**

```bash
net rpc user -S dc01.corp.example -U 'CORP/alice%Password123'
net rpc group list -S dc01.corp.example -U 'CORP/alice%Password123'
```

**8. Remote share enumeration over RAP.**

```bash
net rap share -S fileserver.corp.example -U 'CORP/alice%Password123'
```

**9. ADS information and LDAP search.**

```bash
net ads info -S dc01.corp.example -U 'CORP/alice%Password123'
net ads search '(sAMAccountName=alice)' -S dc01.corp.example \
  -U 'CORP/alice%Password123'
```

**10. Remote service control over RPC.**

```bash
net rpc service list -S win01.corp.example -U 'CORP/alice%Password123'
net rpc service status -S win01.corp.example -U 'CORP/alice%Password123' Spooler
```

## `smbpasswd`

`smbpasswd` changes SMB passwords. In local mode it edits Samba's password
database; in remote mode it speaks the SMB password-change protocol to another
host. `-L` must be the first option.

```text
smbpasswd [options] [username]
```

| Option | Meaning |
|--------|---------|
| `-L` | Local mode (must be first) |
| `-h` | Print usage |
| `-s` | Read the password from stdin |
| `-c FILE` | Alternative `smb.conf` |
| `-D LEVEL` | Debug level |
| `-r MACHINE` | Remote machine |
| `-U USER` | Remote username |
| `-a` | Add a user (root or local mode) |
| `-d` | Disable a user |
| `-e` | Enable a user |
| `-i` | Interdomain trust account |
| `-m` | Machine trust account |
| `-n` | Set no password |
| `-W` | Read the LDAP admin password from stdin |
| `-w PASSWORD` | LDAP admin password |
| `-x` | Delete a user |
| `-R ORDER` | Name resolution order |

**Add a local user non-interactively.**

```bash
printf 'Password123\nPassword123\n' | smbpasswd -L -s -a alice
```

**Disable, enable, then delete a local user.**

```bash
smbpasswd -L -d alice
smbpasswd -L -e alice
smbpasswd -L -x alice
```

**Change a remote password.**

```bash
smbpasswd -r fileserver.corp.example -U 'CORP/alice'
```

## `testparm`

`testparm` parses `smb.conf`, reports unknown parameters, and prints the
effective configuration. It is the first command to run after any Samba
configuration change.

```text
testparm [options] <config-file> [host-name] [host-ip]
```

| Option | Meaning |
|--------|---------|
| `-s, --suppress-prompt` | Do not wait for Enter |
| `-v, --verbose` | Show default values too |
| `-l, --skip-logic-checks` | Skip global logic checks |
| `--show-all-parameters` | List every parameter with its type |
| `--parameter-name=NAME` | Print only one parameter |
| `--section-name=SECTION` | Print only one section |
| `--option=name=value` | Override one option |
| `-d, --debuglevel=LEVEL` | Debug level |
| `--debug-stdout` | Debug to stdout |
| `-V, --version` | Print version |

**Validate and print.**

```bash
testparm -s
testparm -s /etc/samba/smb.conf
```

**Show one parameter or one section.**

```bash
testparm --parameter-name='server role' -s
testparm --section-name=homes -s
```

**List every parameter and its type.**

```bash
testparm --show-all-parameters | head -40
```

## `smbcontrol`

`smbcontrol` sends a message to a running Samba daemon by name (`smbd`,
`nmbd`, `winbindd`, `ldap_server`) or by PID. The daemon must be running; on a
host with no Samba services every destination fails with
`Can't find pid for destination`.

```text
smbcontrol [options] <destination> <message-type> <parameters>
```

| Message | Meaning |
|---------|---------|
| `ping` | Elicit a response |
| `debug LEVEL` | Set the debug level |
| `debuglevel` | Show current debug levels |
| `force-election` | Force a browse election |
| `profile` / `profilelevel` | Profiling control |
| `inject` | Inject a fatal signal into `smbd` |
| `stacktrace` | Print a daemon stack trace |
| `pool-usage` | Show talloc memory usage |
| `ringbuf-log` | Dump the ring buffer log |
| `close-share SHARE` | Disconnect a share |
| `close-denied-share` | Disconnect users from now-denied shares |
| `kill-client-ip IP` | Disconnect a client by IP |
| `printnotify` | Send a print notify message |
| `rpc-dump-status` | Dump RPC status |

```bash
smbcontrol smbd ping
smbcontrol smbd debug 3
smbcontrol -t 2 smbd pool-usage
smbcontrol 12345 ping
```

## `nmblookup`

`nmblookup` queries NetBIOS name service on UDP 137. It resolves names to IP
addresses, looks up node status, and translates addresses back to names.

```text
nmblookup [options] <NODE> ...
```

| Option | Meaning |
|--------|---------|
| `-B, --broadcast=ADDR` | Broadcast address |
| `-f, --flags` | Show NMB flags |
| `-U, --unicast=ADDR` | Unicast address |
| `-M, --master-browser` | Search for a master browser |
| `--recursion` | Set recursion desired |
| `-S, --status` | Look up node status too |
| `-T, --translate` | Translate an IP address to a name |
| `-r, --root-port` | Use source port 137 |
| `-A, --lookup-by-ip` | Node status on an IP address |

```bash
nmblookup -A 192.0.2.10
nmblookup -S WORKGROUP
nmblookup -M WORKGROUP
nmblookup -U 192.0.2.10 WORKGROUP
nmblookup -T 192.0.2.10
```

## `dbwrap_tool`

`dbwrap_tool` reads and writes Samba's tdb databases. It requires exactly one
of `--persistent` or `--non-persistent`. `--non-persistent` opens an in-memory
copy and can wipe the database; use it only on a copy or a scratch file.

```text
dbwrap_tool [options] <database> <op> [<key> [<type> [<value>]]]
```

Operations: `fetch`, `store`, `delete`, `exists`, `erase`, `listkeys`.
Types: `int32`, `uint32`, `string`, `hex`.

```bash
dbwrap_tool --persistent /var/lib/samba/registry.tdb listkeys
dbwrap_tool --persistent /var/lib/samba/registry.tdb exists 'HKLM\Software\Samba'
dbwrap_tool --persistent /tmp/scratch.tdb store key string value
dbwrap_tool --persistent /tmp/scratch.tdb fetch key string
dbwrap_tool --persistent /tmp/scratch.tdb delete key
```

## `winexe`

`winexe` installs a temporary service on a remote Windows host and runs a
command as `SYSTEM` or as a supplied user. It needs local administrator
credentials on the target.

```text
winexe [OPTION]... //HOST[:PORT] COMMAND
```

| Option | Meaning |
|--------|---------|
| `--uninstall` | Uninstall the service after execution |
| `--reinstall` | Reinstall the service before execution |
| `--runas=[DOMAIN\]USER%PASSWORD` | Run as a user (cleartext over the network) |
| `--runas-file=FILE` | Read run-as options from a file |
| `--interactive=0\|1` | Allow desktop interaction |
| `--ostype=0\|1\|2` | 32-bit, 64-bit, or auto |
| `--system` | Run as `SYSTEM` |

```bash
winexe -U 'CORP/alice%Password123' //win01.corp.example 'cmd /c whoami'
winexe --runas='CORP/alice%Password123' //win01.corp.example 'cmd /c hostname'
winexe --system //win01.corp.example 'cmd /c whoami'
```

## `samba-regedit`

`samba-regedit` is an interactive curses registry editor. It connects to a
remote host over SMB and lets you browse, create, and delete keys and values.

```bash
samba-regedit -S win01.corp.example -U 'CORP/alice%Password123'
```

It needs a real terminal; redirecting stdin or stdout prevents the curses UI
from starting.

## `samba-log-parser`

`samba-log-parser` groups Samba debug logs by trace ID or PID and can print
request/sub-request flows.

```text
samba-log-parser [-h] [--traceid ID] [--pid PID] [--breakdown]
                 [--merge-by-timestamp] [--flow] [--flow-compact] path
```

| Option | Meaning |
|--------|---------|
| `--traceid ID` | Select one trace ID |
| `--pid PID` | Select one winbind client PID |
| `--breakdown` | Write per-trace files |
| `--merge-by-timestamp` | Merge logs by timestamp |
| `--flow` | Show request/sub-request flow |
| `--flow-compact` | Flow without DCE/RPC detail |

```bash
samba-log-parser --traceid 1234 /var/log/samba
samba-log-parser --pid 5678 /var/log/samba
samba-log-parser --breakdown /var/log/samba
samba-log-parser --flow /var/log/samba/log.smbd
```

## Exit status

| Tool | Success | Failure |
|------|---------|---------|
| `net` | 0 for most successful commands and `--help` | 255 for usage errors and many remote failures; 1 for some local lookups |
| `smbpasswd` | 0 on a completed change | 1 for usage, including `-h` |
| `testparm` | 0 even with unknown parameters | 1 for a missing file or a malformed `--option` |
| `smbcontrol` | 0 when the message is delivered | 1 for usage or an unknown destination |
| `nmblookup` | 0 when a name is resolved | 1 when the name is not found |
| `dbwrap_tool` | 0 on a successful operation | 1 or 255 on a missing key or bad arguments |
| `winexe` | 0 when the remote command succeeds | 1 for usage or connection failure |
| `samba-regedit` | 0 on a clean exit | 1 on connection or terminal failure |
| `samba-log-parser` | 0 when parsing succeeds | 1 for a missing selector or unreadable path; 2 for a missing argument |

## Quirks and limits

- **`net` has several different usage exit codes.** `net` with no arguments
  and `net help` exit 255. `net --help` and `net -h` exit 0. `net --usage` is
  an unknown option and exits 1.
- **`net help <command>` does not always mean help.** `net help usershare` and
  `net help usersidlist` execute the command. `net help bogus` exits 0 with
  the general usage. `net help file`, `user`, `group`, `domain`, `lookup`,
  `offlinejoin`, and `password` print usage and exit 255.
- **`net time` needs a target.** Without `-S` it exits 255 with
  `Could not locate a time server`. `net time system`, `net time zone`, and
  `net time set` without `-S` print usage and exit 0 instead of acting.
- **`net status` needs a subcommand.** `net status` alone exits 255;
  `net status sessions` and `net status shares` work locally and print empty
  tables on an idle host.
- **`net getlocalsid` works on a standalone host.** `net getdomainsid` exits 1
  with `Could not fetch domain SID`, and `net maxrid` exits 1 with
  `can't get current maximum rid`.
- **`net conf` is registry-backed.** `net conf list` was empty on this host
  even though `/etc/samba/smb.conf` exists. `net conf getparm global ...`
  fails with `service 'global' does not exist`; pass a share name instead.
- **`net registry` can work without a running daemon.** Enumerating
  `HKLM\Software\Samba` reads the local registry tdb directly.
- **`net usershare` needs a usershare directory.** It fails with
  `cannot open usershare directory /var/lib/samba/usershares` when the
  directory does not exist.
- **`net tdb locking` needs the locking database.** It fails with
  `Failed to open byte range locking database /run/samba/brlock.tdb` when
  Samba has not created it.
- **`net idmap dump` needs the idmap database.** It fails with
  `Could not open idmap db (/var/lib/samba/winbindd_idmap.tdb)` when winbind
  has not run.
- **`net witness` requires clustering.** `net witness list` reports
  `Only supported with clustering=yes!`.
- **`net sam list` is quiet on a standalone host.** `net sam list users` and
  `net sam list groups` exit 0 with no output.
- **`net join` is not a standalone operation.** It exits 255 with
  `cannot join as standalone machine`.
- **`net rpc user` and `net rpc share` prompt for a password.** Supply `-U`
  and `-N`, or an authentication file, in non-interactive runs.
- **`net ads` needs a reachable DC.** `net ads info` fails with
  `Didn't find the ldap server!` when no logon server is available.
- **`net rap` defaults to localhost.** Without `-S` it tries `127.0.0.1` and
  reports `NT_STATUS_CONNECTION_REFUSED` when no SMB server is listening.
- **`smbpasswd -h` exits 1.** `--help` and `-V` are invalid options; there is
  no version flag. `-L` must be the first option.
- **`smbpasswd -s` reads the password from stdin.** It is the only practical
  non-interactive form; the password is still sent to the local or remote
  password database.
- **`testparm` tolerates unknown parameters.** It warns
  `Unknown parameter encountered` and still exits 0. A missing config file or
  a malformed `--option` exits 1.
- **`testparm` without `-s` waits for Enter.** In a script it reads EOF and
  continues; use `-s` to suppress the prompt.
- **`smbcontrol -?` prints help to stderr and exits 1.** `smbcontrol -V` exits
  0. With no running daemon, every named destination fails with
  `Can't find pid for destination`.
- **`nmblookup -?` exits 0 but no arguments exit 1.** `-A 127.0.0.1` blocked
  until the timeout in testing; `-S localhost` and `-T 127.0.0.1` returned
  `name_query failed to find name`. NetBIOS name service is UDP 137 and is
  often filtered.
- **`dbwrap_tool` requires a persistence mode.** Without exactly one of
  `--persistent` and `--non-persistent` it exits 1 with
  `ERROR: you must specify exactly one of --persistent and --non-persistent`.
- **`--non-persistent` does not persist between invocations.** A `store`
  followed by a separate `fetch` did not find the key. Use `--persistent` on a
  scratch file for a real round trip, and never point it at a live database
  you have not backed up.
- **`winexe` sends `--runas` passwords in cleartext.** The help text warns
  about this explicitly. Prefer a domain credential over an untrusted network,
  or use `--runas-file` with restrictive permissions.
- **`samba-regedit` is interactive.** It needs a terminal and a reachable
  remote registry service; it is not scriptable.
- **`samba-log-parser` needs a selector.** A bare path exits 1 with
  `One of --traceid or --pid is needed or --breakdown or --merge-by-timestamp.`
  No arguments exits 2.

## Related tools

- [`smbclient`](../smbclient/) — interactive SMB client and share access.
- [`smbget`](../smbget/) — non-interactive SMB downloader.
- [`impacket`](../impacket/) — SMB, Kerberos, and AD attack tooling.
- [`netexec`](../netexec/) — multi-protocol SMB/AD enumeration and spraying.
- [`enum4linux-ng`](../enum4linux-ng/) — SMB and NetBIOS enumeration.
- [`ldapsearch`](../ldapsearch/) — raw LDAP queries against AD.
- [`ldapdomaindump`](../ldapdomaindump/) — bulk AD LDAP dump.
- [`nmap`](../nmap/) — SMB and NetBIOS NSE scripts.

## References

- `man net`, `man smbpasswd`, `man testparm`, `man smbcontrol`,
  `man nmblookup`, `man dbwrap_tool`, `man winexe`, and `man samba-regedit`.
- The help snapshots in this directory.
- Samba project documentation: https://www.samba.org/samba/docs/
- `smb.conf(5)` for the parameters accepted by `--option`.
