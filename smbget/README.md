# smbget — wget-like SMB downloader

- **Version:** 4.22.11-Debian-4.22.11+dfsg-0+deb13u1 (Debian package `smbclient`)
- **Binary:** `/usr/bin/smbget`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://www.samba.org/
- **Licence:** GPL-3.0-or-later

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`smbget` downloads files and directory trees from SMB/CIFS shares with
`wget`-style semantics. It is the quickest way to pull a share, a backup
directory, or a single file when you already have SMB credentials and do not
need an interactive client. It can resume partial downloads, skip files that
are already current, throttle bandwidth, and write the file to stdout for
piping.

It is a client only: it reads from the remote share and writes to the local
filesystem or stdout. It does not modify the remote server.

Typical assessment flow:

1. Obtain a credential, guest access, or a machine account.
2. List the share with `smbclient` or `netexec`.
3. Download the interesting tree with `smbget --recursive`.
4. Search the local copy for configuration files, scripts, backups, and
   credentials.

## URL forms

```text
smb://host/share/path/to/file
smb://host:port/share/path/to/file
smb://DOMAIN;user:password@host/share/path
smb://user:password@host/share/path
```

The share name is the first path component after the host. A trailing slash on
a directory is optional; both `smb://host/share` and `smb://host/share/` work
with `--recursive`.

Credentials can also come from `-U`, `--password`, `-A`, the `PASSWD`
environment variable, or an interactive prompt. `-N` suppresses the prompt.

## Options

```text
smbget [-a] [-r] [--recursive] [-D] [-o FILE] [-q] [-v] [-b INT]
       [--stdout] [-u] [-e] [--limit-rate=INT] [-?|--help] [--usage]
       [-d LEVEL] [--debug-stdout] [-s FILE] [--option=name=value]
       [-l BASENAME] [--leak-report] [--leak-report-full]
       [-R ORDER] [-O OPTIONS] [-m PROTOCOL] [-n NAME] [--netbios-scope=SCOPE]
       [-W WORKGROUP] [--realm=REALM]
       [-U [DOMAIN/]USER[%PASS]] [-N] [--password=STRING] [--pw-nt-hash]
       [-A FILE] [-P] [--simple-bind-dn=DN]
       [--use-kerberos=desired|required|off] [--use-krb5-ccache=CCACHE]
       [--use-winbind-ccache] [--client-protection=sign|encrypt|off]
       [-V] smb://host/share/path [...]
```

### Download behaviour

| Option | Meaning |
|--------|---------|
| `-a, --guest` | Authenticate as guest |
| `-r, --resume` | Resume an aborted or partial local file |
| `-u, --update` | Download only if the remote file is newer or local is missing |
| `--recursive` | Recurse into directories |
| `-b, --blocksize=INT` | Block size in bytes |
| `-o, --outputfile=FILE` | Write to a specific local file |
| `--stdout` | Write file data to stdout |
| `--limit-rate=INT` | Throttle to `INT` KB/s |
| `-D, --dots` | Print dots as progress |
| `-q, --quiet` | Suppress progress and success output |
| `-v, --verbose` | Verbose progress |
| `-e, --encrypt` | Require SMB transport encryption |

### Credentials

| Option | Meaning |
|--------|---------|
| `-U, --user=[DOMAIN/]USER[%PASS]` | Username, optionally with domain and password |
| `-N, --no-pass` | Do not prompt for a password |
| `--password=STRING` | Password on the command line |
| `--pw-nt-hash` | Treat the supplied password as an NT hash |
| `-A, --authentication-file=FILE` | Read credentials from a file |
| `-P, --machine-pass` | Use the stored machine account password |
| `--simple-bind-dn=DN` | DN for a simple bind |
| `--use-kerberos=MODE` | `desired`, `required`, or `off` |
| `--use-krb5-ccache=FILE` | Kerberos credential cache |
| `--use-winbind-ccache` | Use the winbind credential cache |
| `--client-protection=MODE` | `sign`, `encrypt`, or `off` |

### Connection and diagnostics

| Option | Meaning |
|--------|---------|
| `-R, --name-resolve=ORDER` | Name resolution order |
| `-O, --socket-options=OPTS` | Socket options |
| `-m, --max-protocol=PROTO` | Maximum SMB protocol |
| `-n, --netbiosname=NAME` | Local NetBIOS name |
| `--netbios-scope=SCOPE` | NetBIOS scope |
| `-W, --workgroup=GROUP` | Workgroup or domain |
| `--realm=REALM` | Kerberos realm |
| `-s, --configfile=FILE` | Alternative `smb.conf` |
| `--option=name=value` | Override one `smb.conf` option |
| `-l, --log-basename=BASE` | Basename for log files |
| `-d, --debuglevel=LEVEL` | Debug level |
| `--debug-stdout` | Send debug output to stdout |
| `--leak-report`, `--leak-report-full` | Talloc leak report on exit |
| `-V, --version` | Print version |

## Recipes

**1. Download one file.**

```bash
smbget -N 'smb://fileserver.corp.example/backups/router-config.txt'
```

**2. Download with a domain credential.**

```bash
smbget -N -U 'CORP/alice%Password123' \
  'smb://fileserver.corp.example/backups/router-config.txt'
```

**3. Download a whole share recursively.**

```bash
smbget -N --recursive -U 'CORP/alice%Password123' \
  'smb://fileserver.corp.example/backups/'
```

**4. Resume an interrupted ISO download.**

```bash
smbget -N -r 'smb://fileserver.corp.example/isos/tools.iso'
```

**5. Re-run a mirror and only fetch newer files.**

```bash
smbget -N -u --recursive 'smb://fileserver.corp.example/backups/'
```

**6. Pipe a file without writing it to disk.**

```bash
smbget -N --stdout -q 'smb://fileserver.corp.example/config/web.config' \
  | grep -i 'connectionString'
```

**7. Throttle a large transfer to 500 KB/s.**

```bash
smbget -N --limit-rate=500 'smb://fileserver.corp.example/isos/tools.iso'
```

**8. Use an authentication file.**

```bash
printf 'username = alice\npassword = Password123\ndomain = CORP\n' > auth.txt
chmod 600 auth.txt
smbget -A auth.txt 'smb://fileserver.corp.example/backups/'
```

**9. Guest access.**

```bash
smbget -a --recursive 'smb://fileserver.corp.example/public/'
```

**10. Pass-the-hash.**

```bash
smbget -N --pw-nt-hash -U 'CORP/alice%31d6cfe0d16ae931b73c59d7e0c089c0' \
  'smb://fileserver.corp.example/backups/'
```

## Output and naming

- A single file is written under its remote basename unless `-o` is given.
- `-o` writes the whole download to one file and cannot be combined with
  `--stdout` or `--recursive`.
- `--recursive` preserves paths below the requested directory relative to the
  current directory. It does **not** create a directory named after the share.
- Recursing on `smb://host/share` writes `file.txt` and `sub/nested.txt` into
  the current directory. Recursing on `smb://host/share/sub` flattens the
  contents of `sub` into the current directory.
- Multiple URLs on one command line are downloaded in sequence and the final
  byte total covers all of them.
- Files are created with the remote permission bits. Files from a share that
  reports execute permission commonly land as mode `0755`.
- Progress and the final `Downloaded N in N seconds` line go to stderr.
  `--stdout` puts file data on stdout, so `--stdout -q` gives clean content.
- `-q` suppresses progress and success output but not errors.

## Exit status

| Code | Meaning |
|------|---------|
| 0 | Download completed, help printed, or version printed |
| 1 | Connection, authentication, URL, or local-file failure |

Some argument conflicts are reported but still exit 0. In particular,
combining `-o` with `--stdout`, or `--recursive` with either of them, prints a
message, downloads nothing, and exits 0. Do not rely on the exit code alone;
check that the expected file exists.

## Quirks and limits

- **The man-page synopsis is misleading about `-O` and `-R`.** In this build
  `-O` is `--socket-options` and `-R` is `--name-resolve`; `--stdout` and
  `--recursive` have no short forms. `smbget -R smb://host/share/file` treats
  the URL as a name-resolution order, warns, downloads nothing, and exits 0.
  `smbget -O smb://host/share/file` does the same as a socket option.
- **Existing files are not overwritten.** A second download of the same name
  fails with `Can't open <name>: File exists` and exits 1. Use `-u`, `-r`, or
  remove the local file first.
- **`-r` needs a partial local file.** It resumes from the current local size;
  it is not a synonym for "retry".
- **`-u` compares timestamps.** It skips a file whose local mtime is not older
  than the remote mtime. `-v` prints `... is up-to-date, skipping`.
- **`--limit-rate` is in KB/s, not bytes/s.** `--limit-rate=10` produced about
  9.85 kB/s in testing. Zero and negative values are accepted by the parser.
- **`-b` is in bytes and is not validated for sanity.** `-b 0` and `-b -1`
  parse without error; their transfer effect was not verified.
- **A directory without `--recursive` fails.** The message says
  `Specify -R to download recursively`, but the working flag is `--recursive`.
- **`-o` and `--stdout` are mutually exclusive.** The conflict message is
  printed, nothing is written, and the exit code is 0.
- **`--recursive` cannot be combined with `-o` or `--stdout`.**
- **`-A` requires a readable credentials file.** A missing file exits 1 with
  `Failed to set parse authentication file!` and
  `ERROR: Unable to open credentials file!`.
- **`-P` needs a domain-joined machine.** On a standalone host it exits 1 with
  `Failed to set machine account: NT_STATUS_CANT_ACCESS_DOMAIN_INFO`.
- **`--use-kerberos=required` without a cache fails.** Observed as
  `smbc_open: Invalid argument`.
- **`--client-protection=encrypt` depends on the server.** A server that does
  not support SMB encryption can fail with an SMB2 signature error.
- **Invalid URL schemes fail late.** `http://...`, a bare path, or a host
  without `smb://` produce `smbc_open: Invalid argument`.
- **`smb://` alone is a permission error.** It exits 1 with
  `You don't have enough permissions to access smb://`.
- **A missing remote file is reported by URL.** The message is
  `<url> can't be found on the remote server` and the exit code is 1.
- **`-m BOGUS` only warns.** The invalid protocol value is ignored and the
  default is used.
- **`-l` with a missing directory logs a warning and continues.** It does not
  abort the download.
- **`--bogus` is an argument error.** It exits 1 with
  `Invalid option --bogus: unknown option`.
- **Missing option arguments exit 1.** `-o` and `--limit-rate` report
  `<option>: missing argument`.
- **Invalid numbers exit 1.** `-b abc` reports `abc: invalid numeric value`;
  `--limit-rate=abc` reports `--limit-rate=abc: invalid numeric value`.

## Related tools

- [`smbclient`](../smbclient/) — interactive SMB client and share enumeration.
- [`samba-common-bin`](../samba-common-bin/) — `net`, `smbpasswd`, `testparm`,
  `nmblookup`, and related Samba administration tools.
- [`impacket`](../impacket/) — SMB, Kerberos, and AD attack tooling, including
  `smbclient.py` and `smbserver.py`.
- [`netexec`](../netexec/) — multi-protocol SMB/AD enumeration and spraying.
- [`enum4linux-ng`](../enum4linux-ng/) — SMB and NetBIOS enumeration.
- [`nmap`](../nmap/) — SMB script scanning with `smb-*` NSE scripts.

## References

- `man smbget` and [`help.txt`](help.txt) in this directory.
- Samba project documentation: https://www.samba.org/samba/docs/
- `smb.conf(5)` for the options accepted by `--option`.
