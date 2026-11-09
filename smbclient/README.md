# smbclient — SMB/CIFS share access and file transfer

- **Version:** Samba 4.22.11-Debian (`smbclient --version`)
- **Binary:** `/usr/bin/smbclient` (Debian package `smbclient`)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://www.samba.org/samba/docs/current/man-html/smbclient.1.html
- **Docs:** `man smbclient`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

smbclient is an FTP-like client for SMB/CIFS shares. It lists shares, connects
to one, and then supports interactive commands to browse directories, download
and upload files, and inspect metadata. It is the standard way to confirm what
an SMB share actually exposes after `nmap`, `netexec`, or `enum4linux-ng` finds
one.

It is a client, not a scanner. It does not enumerate users or brute-force
passwords; use `netexec`, `rpcclient`, or `enum4linux-ng` for that. Its value is
direct, scriptable access to a share with the exact credentials you have.

## Use cases

| Situation | Why smbclient |
|-----------|---------------|
| Share inventory | `-L host` lists shares and comments |
| Confirm access | Connect with a user/hash and list the root |
| File review | Download a specific file for offline analysis |
| Evidence capture | `-c 'ls; get ...'` in a script |
| Null-session check | `-N -L host` tests anonymous access |
| Backup/restore | `-T` tar mode for bulk transfer |
| Kerberos environments | `--use-kerberos=required` with a ccache |

## Requirements & install layout

- Debian package `smbclient`; binary `/usr/bin/smbclient`. Samba 4.22.11.
- No root needed for client access.
- Config is read from `/etc/samba/smb.conf` unless `-s` points elsewhere;
  `--option=name=value` overrides one setting.
- **SMB1 is disabled by default** in modern Samba clients. Legacy servers may
  need `-m NT1` and a matching `client min protocol` setting.
- Password on the command line is visible in the process list; prefer `-A` or
  the interactive prompt.

## Basic syntax

```bash
smbclient [OPTIONS] service <password>

smbclient -L //10.10.10.5 -N
smbclient //10.10.10.5/share -U 'DOMAIN/user%password'
smbclient //10.10.10.5/share -U user --pw-nt-hash <nthash>
smbclient //10.10.10.5/share -U user -c 'ls; get file.txt'
```

The service is `//server/share` (or `\\server\share`). If the password is
omitted, smbclient prompts for it.

## Flag reference

### Connection and listing

| Flag | Meaning |
|------|---------|
| `-L, --list <host>` | List shares on a host |
| `-I, --ip-address <ip>` | Connect to this IP instead of resolving the name |
| `-p, --port <port>` | SMB port (default 445) |
| `-m, --max-protocol <proto>` | Maximum protocol (e.g. `SMB3`, `NT1`) |
| `-R, --name-resolve <order>` | Name resolution order |
| `-n, --netbiosname <name>` | Local NetBIOS name |
| `--netbios-scope <scope>` | NetBIOS scope |
| `-W, --workgroup <wg>` | Workgroup/domain |
| `--realm <realm>` | Kerberos realm |
| `-O, --socket-options <opts>` | Socket options |
| `-B, --browse` | Browse SMB servers via DNS |

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

### Commands and transfer

| Flag | Meaning |
|------|---------|
| `-c, --command <string>` | Semicolon-separated commands |
| `-D, --directory <dir>` | Start in this directory |
| `-T, --tar <c\|x>IXFvgbNan` | Create/extract a tar archive |
| `-M, --message <host>` | Send a WinPopup message |
| `-b, --send-buffer <bytes>` | Transmit buffer size |
| `-t, --timeout <seconds>` | Per-operation timeout |
| `-g, --grepable` | Grepable output |
| `-q, --quiet` | Suppress the help banner |

### Debugging

| Flag | Meaning |
|------|---------|
| `-d, --debuglevel <n>` | Debug level |
| `--debug-stdout` | Send debug output to stdout |
| `-s, --configfile <file>` | Alternate smb.conf |
| `--option=name=value` | Override one smb.conf setting |
| `-l, --log-basename <base>` | Log file basename |
| `--leak-report` / `--leak-report-full` | Talloc leak reporting |
| `-V, --version` | Print version |

## Interactive commands

Run `help` inside a session for the full list. The common ones:

| Command | Meaning |
|---------|---------|
| `ls` / `dir` | List the current directory |
| `cd <dir>` / `lcd <dir>` | Change remote / local directory |
| `pwd` | Print the remote directory |
| `get <file> [local]` | Download one file |
| `mget <pattern>` | Download matching files |
| `put <file> [remote]` | Upload one file |
| `mput <pattern>` | Upload matching files |
| `prompt` | Toggle prompting for `mget`/`mput` |
| `recurse` | Toggle recursive `mget`/`mput` |
| `mask <pattern>` | Set the transfer mask |
| `del` / `rm` | Delete a remote file |
| `mkdir` / `rmdir` | Create/remove a remote directory |
| `rename <old> <new>` | Rename a remote file |
| `more <file>` | Page through a remote file |
| `allinfo <file>` | Show all metadata |
| `showacls` | Show ACLs |
| `posix` | Toggle POSIX path handling |
| `chmod` / `chown` | Change permissions/owner (server support required) |
| `symlink` / `readlink` | Symlink operations |
| `tarmode` | Toggle tar mode |
| `du` | Disk usage |
| `volume` | Volume information |
| `!<cmd>` | Run a local shell command |
| `quit` / `exit` / `q` | Leave the session |

## Recipes

Authorized targets only.

**1. List shares with a null session.**

```bash
smbclient -L //10.10.10.5 -N
```

**2. List shares as a domain user.**

```bash
smbclient -L //10.10.10.5 -U 'ACME/alice%Password123' -W ACME
```

**3. Connect and list the root of a share.**

```bash
smbclient //10.10.10.5/Finance -U 'ACME/alice%Password123'
```

**4. Pass-the-hash.** The `--pw-nt-hash` flag tells smbclient the password
argument is an NT hash, not plaintext.

```bash
smbclient //10.10.10.5/C$ -U 'ACME/alice' --pw-nt-hash \
  8846f7eaee8fb117ad06bdd830b7586c
```

**5. Script a download with `-c`.** Commands are semicolon-separated.

```bash
smbclient //10.10.10.5/share -U user%pass -c 'cd reports; ls; get q4.pdf'
```

**6. Recursive download.**

```bash
smbclient //10.10.10.5/share -U user%pass -c 'prompt off; recurse on; mget *'
```

**7. Kerberos with a ccache.**

```bash
export KRB5CCNAME=~/acme.ccache
smbclient //dc.acme.local/SYSVOL --use-kerberos=required -U alice
```

**8. Legacy SMB1 server.** Only if the server truly requires it.

```bash
smbclient -L //10.10.10.5 -N -m NT1 --option='client min protocol=NT1'
```

**9. Grepable share list for scripts.**

```bash
smbclient -L //10.10.10.5 -N -g
```

## Output & pipelines

- `-L` prints a share table: name, type (Disk/IPC/Printer), and comment.
- `-g` emits `type|name|comment` lines that are easy to `awk`/`grep`.
- `-c` returns command output on stdout; errors go to stderr (use `-E` to merge
  them).
- `-T` tar mode writes or reads a tar file; `tarmode` controls inclusion of
  hidden/system files.
- `-d 3` or higher shows the SMB dialect negotiation, which is useful when a
  connection fails.
- Exit status is `0` on success and non-zero on connection/auth failure.

## Gotchas & operational security

- **SMB1 is off by default.** If a legacy server fails, check the dialect with
  `-d 3` before forcing `-m NT1`; forcing SMB1 weakens the connection.
- **Passwords on the command line leak.** They appear in `ps` and shell
  history; use `-A` or let smbclient prompt.
- **`--client-protection=off` disables signing/encryption.** Only use it when
  the server requires it and the engagement allows it; it removes integrity
  protection.
- **Null sessions are rare on modern systems.** A failed `-N` list is not proof
  that SMB is misconfigured.
- **Share names are case-insensitive, paths are not always.** Use the exact
  case the server reports.
- **Recursive `mget` can copy far more than intended.** Set `mask` and
  `prompt`, and check `du` first.
- **Downloaded files may contain malware or personal data.** Handle them in an
  isolated directory and follow the engagement's data-handling rules.
- **`C$`/`ADMIN$` access is highly privileged.** Treat it as a finding, not a
  convenience.

## Legal & safety

Access only shares you own or have **explicit written authorization** to
access. Reading files outside the agreed scope, even on an in-scope host, can
exceed authorization. Do not download personal data unless it is necessary for
the finding, and delete it at engagement close.

## Related tools

- [`netexec/`](../netexec/README.md) — SMB enumeration, spraying, and command
  execution across hosts.
- [`rpcclient/`](../rpcclient/README.md) — MS-RPC enumeration (users, groups,
  shares).
- [`enum4linux-ng/`](../enum4linux-ng/README.md) — broad SMB/AD enumeration.
- [`impacket/`](../impacket/README.md) — `smbclient.py`, `secretsdump`, and
  `psexec` alternatives.
- [`nmap/`](../nmap/README.md) — `smb-*` NSE scripts for share discovery.
- [`nbtscan/`](../nbtscan/README.md) — NetBIOS name inventory before SMB access.

## References

- Installed help snapshot: [`help.txt`](help.txt)
- Samba smbclient man page: https://www.samba.org/samba/docs/current/man-html/smbclient.1.html
- Samba project: https://www.samba.org/
