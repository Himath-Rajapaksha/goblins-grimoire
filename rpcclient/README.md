# rpcclient — MS-RPC enumeration and administration

- **Version:** Samba 4.22.11-Debian (`rpcclient --version`)
- **Binary:** `/usr/bin/rpcclient` (Debian package `smbclient`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html
- **Docs:** `man rpcclient`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

rpcclient speaks MS-RPC over SMB. It binds to the SAMR, LSA, SRVSVC, and other
named pipes and runs commands against them: enumerate users, groups, and
shares; translate SIDs; query domain and server information; and, with
sufficient privilege, create or modify accounts and groups. It is the
command-line equivalent of the old `net rpc` tooling and a useful cross-check
for `netexec` and `enum4linux-ng`.

It is not a scanner and not a relay tool. It needs a reachable SMB service and,
for most useful commands, valid credentials.

## Use cases

| Situation | Why rpcclient |
|-----------|---------------|
| Domain inventory | `enumdomusers`, `enumdomgroups`, `enumdomains` |
| User detail | `queryuser <rid>`, `queryusergroups` |
| SID translation | `lookupsids`, `lookupnames`, `getsid` |
| Share enumeration | `netshareenum`, `netshareenumall` |
| Server fingerprint | `srvinfo`, `dsroledominfo`, `getdcname` |
| Policy review | `getdompwinfo`, `getusrdompwinfo` |
| Admin actions (authorized) | `createdomuser`, `setpasswd`, `addaliasmem` |

## Requirements & install layout

- Debian package `smbclient`; binary `/usr/bin/rpcclient`. Samba 4.22.11.
- No root needed; network access to TCP 445 (and 139 for legacy).
- Config from `/etc/samba/smb.conf`; `-s` and `--option` override it.
- Credentials can be a password, NT hash, Kerberos ccache, or auth file.
- Many commands require domain-admin or local-admin rights and return
  `ACCESS_DENIED` otherwise.

## Basic syntax

```bash
rpcclient [OPTION...] BINDING-STRING|HOST

rpcclient -U '' -N 10.10.10.5
rpcclient -U 'ACME/alice%Password123' 10.10.10.5
rpcclient -U 'ACME/alice%Password123' -c 'enumdomusers' 10.10.10.5
rpcclient -U 'ACME/alice%<nthash>' --pw-nt-hash 10.10.10.5 -c 'getdompwinfo'
```

Without `-c`, rpcclient opens an interactive prompt. Commands are
semicolon-separated with `-c`.

## Flag reference

### Connection

| Flag | Meaning |
|------|---------|
| `HOST` / `BINDING-STRING` | Target host or SMB binding string |
| `-I, --dest-ip <ip>` | Destination IP when the name does not resolve |
| `-p, --port <port>` | SMB port |
| `-R, --name-resolve <order>` | Name resolution order |
| `-O, --socket-options <opts>` | Socket options |
| `-m, --max-protocol <proto>` | Maximum SMB protocol |
| `-n, --netbiosname <name>` | Local NetBIOS name |
| `--netbios-scope <scope>` | NetBIOS scope |
| `-W, --workgroup <wg>` | Workgroup/domain |
| `--realm <realm>` | Kerberos realm |

### Credentials

| Flag | Meaning |
|------|---------|
| `-U, --user=[DOMAIN/]USER[%PASS]` | Username, optionally with password |
| `-N, --no-pass` | Do not prompt for a password |
| `--password <string>` | Password |
| `--pw-nt-hash` | The supplied password is an NT hash |
| `-A, --authentication-file <file>` | Read credentials from a file |
| `-P, --machine-pass` | Use the stored machine account password |
| `--use-kerberos=desired\|required\|off` | Kerberos mode |
| `--use-krb5-ccache <ccache>` | Kerberos credential cache |
| `--use-winbind-ccache` | Use the winbind cache |
| `--client-protection=sign\|encrypt\|off` | SMB signing/encryption |
| `-k, --kerberos` | Deprecated; use `--use-kerberos` |

### Execution and debugging

| Flag | Meaning |
|------|---------|
| `-c, --command <cmds>` | Semicolon-separated commands |
| `-d, --debuglevel <n>` | Debug level |
| `--debug-stdout` | Debug output to stdout |
| `-s, --configfile <file>` | Alternate smb.conf |
| `--option=name=value` | Override one smb.conf setting |
| `-l, --log-basename <base>` | Log file basename |
| `--leak-report` / `--leak-report-full` | Talloc leak reporting |
| `-V, --version` | Print version |

## Common commands

Run `help` at the interactive prompt for the full list. These are the ones used
most in enumeration:

### Domain and server

| Command | Meaning |
|---------|---------|
| `srvinfo` | Server OS/version information |
| `lsaquery` | Domain name and SID |
| `enumdomains` | Enumerate domains |
| `querydominfo` | Domain information |
| `dsroledominfo` | Domain controller role |
| `dsinfo` | Directory service information |
| `getdcname` | DC name for a domain |
| `lsaenumtrustdom` | Trust relationships |
| `enumprivs` | Privileges |
| `lsaenumerateaccountrights <sid>` | Account rights |

### Users and groups

| Command | Meaning |
|---------|---------|
| `enumdomusers` | Enumerate domain users (RID + name) |
| `queryuser <rid>` | Detailed user information |
| `queryusergroups <rid>` | Groups for a user |
| `querydispinfo` | Display information for users |
| `enumdomgroups` | Enumerate domain groups |
| `querygroup <rid>` | Group details |
| `querygroupmem <rid>` | Group members |
| `enumalsgroups` | Alias (local) groups |
| `queryaliasmem <rid>` | Alias members |
| `lookupnames <name>` | Name to SID |
| `lookupsids <sid>` | SID to name |
| `getsid <name>` | SID for a name |
| `getusername` | Current username |
| `getdompwinfo` | Domain password policy |
| `getusrdompwinfo <rid>` | Per-user password policy |

### Shares and sessions

| Command | Meaning |
|---------|---------|
| `netshareenum` | Enumerate shares |
| `netshareenumall` | Enumerate all shares |
| `netsharegetinfo <name>` | Share details |
| `netsessionenum` | Active sessions |
| `netsessiongetinfo <id>` | Session details |
| `netfileenum` | Open files |
| `netnameenum` | Named pipes |
| `netremotetod` | Remote time of day |

### Administrative (authorized only)

| Command | Meaning |
|---------|---------|
| `createdomuser <name>` | Create a domain user |
| `deletedomuser <name>` | Delete a domain user |
| `setuserinfo <rid> <field> <value>` | Modify user fields |
| `setuserinfo2 <name> <field> <value>` | Modify user fields by name |
| `chgpasswd <user> <old> <new>` | Change a password |
| `setpasswd <user> <new>` | Set a password |
| `addaliasm <name>` | Create an alias |
| `addaliasmem <alias> <sid>` | Add a member to an alias |
| `netshareadd` / `netsharedel` | Add/remove a share |
| `reg` | Registry operations |

## Recipes

Authorized targets only.

**1. Null-session domain inventory.** Modern AD often blocks this; a failure is
not proof of misconfiguration.

```bash
rpcclient -U '' -N 10.10.10.5 -c 'srvinfo; lsaquery; enumdomusers'
```

**2. Authenticated user enumeration.**

```bash
rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'enumdomusers'
```

**3. User detail and group membership.** Use the RID from `enumdomusers`.

```bash
rpcclient -U 'ACME/alice%Password123' 10.10.10.5 \
  -c 'queryuser 0x1f4; queryusergroups 0x1f4'
```

**4. SID translation.**

```bash
rpcclient -U 'ACME/alice%Password123' 10.10.10.5 \
  -c 'lookupsids S-1-5-21-1111111111-2222222222-3333333333-1105'
```

**5. Share enumeration.**

```bash
rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'netshareenumall'
```

**6. Password policy.**

```bash
rpcclient -U 'ACME/alice%Password123' 10.10.10.5 -c 'getdompwinfo'
```

**7. Pass-the-hash.**

```bash
rpcclient -U 'ACME/alice%8846f7eaee8fb117ad06bdd830b7586c' --pw-nt-hash \
  10.10.10.5 -c 'enumdomusers'
```

**8. Kerberos.**

```bash
export KRB5CCNAME=~/acme.ccache
rpcclient --use-kerberos=required -U alice dc.acme.local -c 'lsaquery'
```

## Output & pipelines

- `enumdomusers` prints `user:[name] rid:[0x...]`; parse the RID for follow-up
  commands.
- `lookupsids` prints `S-1-5-... domain\name (type)`.
- `netshareenum` prints share name, type, and comment.
- `-c` output goes to stdout; debug and errors go to stderr.
- Interactive `help` lists every command the build supports; `help <command>`
  may show usage.
- Exit status reflects connection/bind failure, not individual command success;
  parse the output for `ACCESS_DENIED` or `NT_STATUS_*`.

## Gotchas & operational security

- **Modern AD restricts anonymous SAMR.** `enumdomusers` with `-N` often returns
  nothing; use authenticated enumeration.
- **RIDs are hexadecimal in output.** `queryuser` accepts `0x...`; convert
  carefully.
- **Many commands need admin.** `createdomuser`, `setpasswd`, and share changes
  will fail for a normal user; do not retry blindly.
- **`rpcclient` is not a relay.** It binds and executes; it does not coerce or
  relay authentication. Use `impacket-ntlmrelayx` for that.
- **SMB signing/encryption may be required.** If the bind fails, check
  `--client-protection` and the server's policy before disabling anything.
- **Passwords on the command line leak.** Use `-A` or the interactive prompt.
- **Enumeration output is sensitive.** It contains account names and SIDs;
  store it in the engagement directory.
- **Write commands change the domain.** Treat them as post-exploitation and
  require explicit authorization.

## Legal & safety

Use rpcclient only against systems you own or have **explicit written
authorization** to test. Domain enumeration exposes account and policy data;
administrative commands can create, modify, or delete accounts. Never run write
commands without explicit permission and a rollback plan.

## Related tools

- [`smbclient/`](../smbclient/README.md) — share access and file transfer.
- [`netexec/`](../netexec/README.md) — SMB/LDAP/WinRM enumeration and spraying.
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — broad SMB/AD enumeration.
- [`impacket/`](../impacket/README.md) — `rpcdump`, `samrdump`, `secretsdump`,
  and relay tooling.
- [`ldapsearch/`](../ldapsearch/README.md) — LDAP-based AD enumeration.
- [`nmap/`](../nmap/README.md) — `msrpc-*` and `smb-*` NSE scripts.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Samba rpcclient man page: https://www.samba.org/samba/docs/current/man-html/rpcclient.1.html
- MS-SAMR: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-samr/
- MS-LSAD: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-lsad/
