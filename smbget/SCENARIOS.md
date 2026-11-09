# smbget — command scenarios for deep research

Installed build: Samba 4.22.11-Debian-4.22.11+dfsg-0+deb13u1, binary
`/usr/bin/smbget`, from the Debian `smbclient` package. Every option below
appears in [`help.txt`](help.txt).

The installed host has no SMB server of its own, so the successful-transfer
behaviour in this file was observed against a local impacket
`SimpleSMBServer` listening on `127.0.0.1:4455`, exposing a read-only share
named `share` from `~/smbget-demo/share`, with a small text file, a nested
file, and a 200,000-byte binary. That server accepts any credentials, so
wrong-password and authorisation-denied paths are documented as lab plans
rather than observed results.

> **Scope:** downloading from a share reads other people's files. Only run
> these scenarios against a server you own or are explicitly authorised to
> assess. Guest access and credential reuse are not authorised by a network
> scan alone.

> **Credential handling:** prefer an environment variable or an authentication
> file over a command line, because a password in `-U` or `--password` appears
> in shell history and process listings. Never write real credentials into this
> repository.

## How to invoke

```text
@deep-research For an authorised SMB assessment with smbget 4.22.11, explain
the URL grammar, how guest, NTLM, hash, authentication-file, and Kerberos
credentials are supplied, how `--recursive` maps remote paths to the local
directory, how `-r` and `-u` decide whether to transfer, how `--limit-rate` and
`-b` affect throughput, why `-O` and `-R` in the man-page synopsis do not mean
`--stdout` and `--recursive`, and which failures still exit 0. Cite the
installed help text, the man page, and observed output. Separate confirmed
behaviour from inference and state confidence.
```

Create a working directory, build a local share, and start the lab server:

```bash
mkdir -p ~/smbget-demo/evidence ~/smbget-demo/share/sub
cd ~/smbget-demo
printf 'router config\n' > share/file.txt
printf 'nested file\n' > share/sub/nested.txt
head -c 200000 /dev/urandom > share/big.bin
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
smbget --help > evidence/help.txt
dpkg -s smbclient | grep -E 'Version|Depends' > evidence/package.txt
```

```bash
# Background SMB server on an unprivileged port; stop it with `kill %1`.
python3 - <<'PY' &
from impacket.smbserver import SimpleSMBServer
s = SimpleSMBServer(listenPort=4455, listenAddress='127.0.0.1')
s.addShare('share', '/home/user/smbget-demo/share')
s.setSMB2Support(True)
s.start()
PY
sleep 1
ss -ltn | grep 4455
```

## Scenario 1 — version, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `smbget -V` | Print the Samba version |
| 2 | `smbget --version` | Long form of the version flag |
| 3 | `smbget '-?'` | Short help flag |
| 4 | `smbget --help` | Full option list |
| 5 | `smbget --usage` | Brief usage message |
| 6 | `smbget --help 2>&1 \| head -20` | Inspect the download options |
| 7 | `smbget --help 2>&1 \| grep -c '^  -'` | Count option lines |
| 8 | `command -v smbget` | Resolve the binary |
| 9 | `readlink -f "$(command -v smbget)"` | Real path |
| 10 | `file "$(command -v smbget)"` | ELF binary check |
| 11 | `ldd "$(command -v smbget)" \| head` | Linked Samba libraries |
| 12 | `dpkg -S /usr/bin/smbget` | Owning package |
| 13 | `dpkg -s smbclient \| grep -E 'Version\|Depends'` | Package provenance |
| 14 | `smbget --bogus` | Unknown option handling |
| 15 | `smbget -o` | Missing option argument |
| 16 | `smbget --limit-rate` | Missing numeric argument |
| 17 | `smbget --help > evidence/help.txt` | Save the help text |
| 18 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which package owns the binary?
- Which short options are real, and which exist only in the man-page synopsis?
- What exit code does an unknown option produce?
- Does `--version` print a Debian revision suffix?

**Sources:** `help.txt`; `man smbget`; `dpkg -S`; observed output.
**Output:** a provenance record and an option inventory.

## Scenario 2 — URL forms and authentication

| # | Command | Purpose |
|---|---------|---------|
| 19 | `smbget -N smb://127.0.0.1:4455/share/file.txt` | Anonymous-style download with no prompt |
| 20 | `smbget -a smb://127.0.0.1:4455/share/file.txt` | Explicit guest login |
| 21 | `smbget -N -U 'alice%Password123' smb://127.0.0.1:4455/share/file.txt` | User and password in `-U` |
| 22 | `smbget -N -U 'WORKGROUP/alice%Password123' smb://127.0.0.1:4455/share/file.txt` | Domain with a slash |
| 23 | `smbget -N -U 'WORKGROUP\alice%Password123' smb://127.0.0.1:4455/share/file.txt` | Domain with a backslash |
| 24 | `smbget -N --password=Password123 -U alice smb://127.0.0.1:4455/share/file.txt` | Password in a separate option |
| 25 | `smbget -N 'smb://alice:Password123@127.0.0.1:4455/share/file.txt'` | Credentials in the URL |
| 26 | `smbget -N 'smb://WORKGROUP;alice:Password123@127.0.0.1:4455/share/file.txt'` | Domain, user, and password in the URL |
| 27 | `smbget -N -W WORKGROUP smb://127.0.0.1:4455/share/file.txt` | Workgroup option |
| 28 | `printf 'username = alice\npassword = Password123\ndomain = WORKGROUP\n' > auth.txt` | Create an authentication file |
| 29 | `chmod 600 auth.txt` | Restrict the credential file |
| 30 | `smbget -A auth.txt smb://127.0.0.1:4455/share/file.txt` | Read credentials from the file |
| 31 | `chmod 644 auth.txt` | Show why permissions matter |
| 32 | `smbget -A auth.txt smb://127.0.0.1:4455/share/file.txt` | Re-run with a world-readable file |
| 33 | `PASSWD=Password123 smbget -N -U alice smb://127.0.0.1:4455/share/file.txt` | Password from the environment |
| 34 | `printf 'Password123' > pw.txt` | Create a password file |
| 35 | `PASSWD_FILE="$PWD/pw.txt" smbget -N -U alice smb://127.0.0.1:4455/share/file.txt` | Password from `PASSWD_FILE` |
| 36 | `smbget -N --pw-nt-hash -U 'alice%31d6cfe0d16ae931b73c59d7e0c089c0' smb://127.0.0.1:4455/share/file.txt` | Treat the password as an NT hash |
| 37 | `smbget -N --use-kerberos=required smb://127.0.0.1:4455/share/file.txt` | Kerberos required without a cache |
| 38 | `smbget -N --use-krb5-ccache=/tmp/nosuch.ccache smb://127.0.0.1:4455/share/file.txt` | Missing Kerberos cache |
| 39 | `smbget -P smb://127.0.0.1:4455/share/file.txt` | Machine account on a standalone host |
| 40 | `smbget -N --simple-bind-dn='CN=alice,DC=corp' smb://127.0.0.1:4455/share/file.txt` | Simple-bind DN |
| 41 | `env SMBGET_PASS='Password123' sh -c 'smbget -N -U "alice%$SMBGET_PASS" smb://127.0.0.1:4455/share/file.txt'` | Keep the secret out of history |

**Research questions**
- Which credential forms put the password in the process list?
- Does `-U` accept `DOMAIN/user`, `DOMAIN\user`, and `user` equally?
- Does the URL form need a domain prefix, and what separator does it use?
- What happens when Kerberos is required but no cache exists?
- Does `-P` work on a host that is not domain-joined?

**Sources:** `help.txt`; `man smbget`; observed output.
**Output:** an authentication matrix with the exposure of each form.

## Scenario 3 — single-file downloads and output naming

| # | Command | Purpose |
|---|---------|---------|
| 42 | `mkdir -p single && cd single` | Fresh download directory |
| 43 | `smbget -N smb://127.0.0.1:4455/share/file.txt` | Download by basename |
| 44 | `ls -l file.txt` | Check name, size, and mode |
| 45 | `cat file.txt` | Verify contents |
| 46 | `sha256sum file.txt` | Record the hash |
| 47 | `rm -f file.txt` | Clear the destination |
| 48 | `smbget -N -o renamed.txt smb://127.0.0.1:4455/share/file.txt` | Rename on download |
| 49 | `ls -l renamed.txt` | Verify the chosen name |
| 50 | `smbget -N smb://127.0.0.1:4455/share/sub/nested.txt` | Nested source still lands by basename |
| 51 | `ls -l nested.txt` | Verify flattening |
| 52 | `smbget -N smb://127.0.0.1:4455/share/file.txt smb://127.0.0.1:4455/share/sub/nested.txt` | Multiple URLs in one run |
| 53 | `ls -l file.txt nested.txt` | Both files present |
| 54 | `smbget -N -o out.bin smb://127.0.0.1:4455/share/big.bin` | Binary download to a chosen name |
| 55 | `cmp out.bin ~/smbget-demo/share/big.bin` | Byte-for-byte comparison |
| 56 | `smbget -N smb://127.0.0.1:4455/share/file.txt` | Re-run without overwrite permission |
| 57 | `smbget -N --stdout smb://127.0.0.1:4455/share/file.txt > stdout.txt` | Redirect stdout data to a file |
| 58 | `smbget -N --stdout smb://127.0.0.1:4455/share/file.txt 2>/dev/null` | Data only, progress discarded |
| 59 | `smbget -N --stdout -q smb://127.0.0.1:4455/share/file.txt > clean.txt` | Quiet stdout for pipelines |
| 60 | `smbget -N -q smb://127.0.0.1:4455/share/sub/nested.txt` | Quiet normal download |
| 61 | `smbget -N -v smb://127.0.0.1:4455/share/sub/nested.txt` | Verbose download |
| 62 | `smbget -N -D smb://127.0.0.1:4455/share/big.bin` | Dot progress |
| 63 | `stat -c '%a %s %n' file.txt nested.txt out.bin` | Compare modes and sizes |

**Research questions**
- Does a nested remote path create a local directory, or only the basename?
- Are downloaded files created with remote execute bits?
- Which stream carries progress, and which carries file data?
- Does `-q` suppress errors as well as progress?
- Does a second download overwrite an existing file?

**Sources:** observed output; local impacket share.
**Output:** a naming and stream-separation table.

## Scenario 4 — recursive downloads and layout

| # | Command | Purpose |
|---|---------|---------|
| 64 | `mkdir -p recursive && cd recursive` | Fresh tree |
| 65 | `smbget -N --recursive smb://127.0.0.1:4455/share/` | Recurse from the share root |
| 66 | `find . -type f \| sort` | Show the local layout |
| 67 | `ls -la` | Confirm no share-name directory |
| 68 | `rm -rf ./*` | Clear the tree |
| 69 | `smbget -N --recursive smb://127.0.0.1:4455/share` | Recurse without a trailing slash |
| 70 | `find . -type f \| sort` | Compare the layout |
| 71 | `rm -rf ./*` | Clear the tree |
| 72 | `smbget -N --recursive smb://127.0.0.1:4455/share/sub` | Recurse on a subdirectory |
| 73 | `find . -type f \| sort` | Check whether `sub/` is preserved |
| 74 | `rm -rf ./*` | Clear the tree |
| 75 | `smbget -N --recursive smb://127.0.0.1:4455/share/sub/` | Same subdirectory with a slash |
| 76 | `find . -type f \| sort` | Compare flattening |
| 77 | `smbget -N --recursive -q smb://127.0.0.1:4455/share/` | Quiet recursive run |
| 78 | `smbget -N --recursive -D smb://127.0.0.1:4455/share/` | Dot progress for a tree |
| 79 | `smbget -N --recursive -o out.bin smb://127.0.0.1:4455/share/` | Rejected option combination |
| 80 | `smbget -N --stdout --recursive smb://127.0.0.1:4455/share/` | Rejected stdout combination |
| 81 | `smbget -N --recursive smb://127.0.0.1:4455/share/sub/nested.txt` | Recurse on a single file |
| 82 | `smbget -N smb://127.0.0.1:4455/share/` | Directory without recursion |
| 83 | `smbget -N --recursive --limit-rate=10 smb://127.0.0.1:4455/share/` | Throttled tree download |
| 84 | `smbget -N --recursive -r smb://127.0.0.1:4455/share/` | Resume mode over a tree |

**Research questions**
- Is the share name included in the local path?
- Does a trailing slash change the destination?
- Are subdirectories preserved below the requested root?
- Which combinations are refused, and do they exit non-zero?
- What does `--recursive` do when given a file URL?

**Sources:** observed output; local impacket share.
**Output:** a path-mapping table and a list of rejected combinations.

## Scenario 5 — resume, update, progress, and throttling

| # | Command | Purpose |
|---|---------|---------|
| 85 | `mkdir -p resume && cd resume` | Fresh resume directory |
| 86 | `head -c 50000 ~/smbget-demo/share/big.bin > big.bin` | Create a partial file |
| 87 | `ls -l big.bin` | Confirm the partial size |
| 88 | `smbget -N -r smb://127.0.0.1:4455/share/big.bin` | Resume from 50,000 bytes |
| 89 | `ls -l big.bin` | Check the completed size |
| 90 | `cmp big.bin ~/smbget-demo/share/big.bin` | Verify integrity |
| 91 | `truncate -s 100000 big.bin` | Reset to a different partial size |
| 92 | `smbget -N -r smb://127.0.0.1:4455/share/big.bin` | Resume again |
| 93 | `cmp big.bin ~/smbget-demo/share/big.bin` | Verify again |
| 94 | `mkdir -p update && cd update` | Fresh update directory |
| 95 | `smbget -N -u smb://127.0.0.1:4455/share/file.txt` | First update-mode download |
| 96 | `touch -d '2020-01-01' file.txt` | Make the local copy look old |
| 97 | `smbget -N -u smb://127.0.0.1:4455/share/file.txt` | Re-download an older local file |
| 98 | `smbget -N -u -v smb://127.0.0.1:4455/share/file.txt` | Up-to-date skip message |
| 99 | `rm -f file.txt` | Remove the local copy |
| 100 | `smbget -N -u smb://127.0.0.1:4455/share/file.txt` | Missing local file is fetched |
| 101 | `smbget -N -D smb://127.0.0.1:4455/share/big.bin` | Dot progress on a large file |
| 102 | `smbget -N -v smb://127.0.0.1:4455/share/big.bin` | Verbose progress |
| 103 | `smbget -N -q smb://127.0.0.1:4455/share/big.bin` | Quiet transfer |
| 104 | `rm -f big.bin` | Clear the destination |
| 105 | `time smbget -N --limit-rate=10 smb://127.0.0.1:4455/share/big.bin` | Throttle 200 kB to about 20 s |
| 106 | `rm -f big.bin` | Clear the destination |
| 107 | `time smbget -N --limit-rate=1000 smb://127.0.0.1:4455/share/big.bin` | Looser throttle |
| 108 | `rm -f big.bin` | Clear the destination |
| 109 | `smbget -N -b 1024 smb://127.0.0.1:4455/share/big.bin` | Small block size |
| 110 | `rm -f big.bin` | Clear the destination |
| 111 | `smbget -N -b 65536 smb://127.0.0.1:4455/share/big.bin` | Large block size |
| 112 | `smbget -N -b abc smb://127.0.0.1:4455/share/big.bin` | Invalid block size |
| 113 | `smbget -N --limit-rate=abc smb://127.0.0.1:4455/share/big.bin` | Invalid rate |
| 114 | `smbget -N --limit-rate=0 smb://127.0.0.1:4455/share/big.bin` | Zero rate is parsed |
| 115 | `smbget -N --limit-rate=-5 smb://127.0.0.1:4455/share/big.bin` | Negative rate is parsed |

**Research questions**
- Does `-r` append from the current local size or restart?
- Does `-u` compare timestamps, sizes, or both?
- Is `--limit-rate` KB/s or bytes/s?
- Are `-b` and `--limit-rate` validated for zero or negative values?
- Does `-q` hide transfer errors?

**Sources:** observed output and timing; local impacket share.
**Output:** a resume/update decision table and a throughput table.

## Scenario 6 — SMB protocol, name resolution, and diagnostics

| # | Command | Purpose |
|---|---------|---------|
| 116 | `smbget -N -m SMB1 smb://127.0.0.1:4455/share/file.txt` | Force SMB1 as the maximum |
| 117 | `smbget -N -m SMB2 smb://127.0.0.1:4455/share/file.txt` | Force SMB2 |
| 118 | `smbget -N -m SMB3 smb://127.0.0.1:4455/share/file.txt` | Force SMB3 |
| 119 | `smbget -N -m BOGUS smb://127.0.0.1:4455/share/file.txt` | Invalid protocol warning |
| 120 | `smbget -N -n LABCLIENT smb://127.0.0.1:4455/share/file.txt` | Set the local NetBIOS name |
| 121 | `smbget -N --netbios-scope=LAB smb://127.0.0.1:4455/share/file.txt` | Set the NetBIOS scope |
| 122 | `smbget -N --realm=LAB.EXAMPLE smb://127.0.0.1:4455/share/file.txt` | Set the Kerberos realm |
| 123 | `smbget -N -O TCP_NODELAY smb://127.0.0.1:4455/share/file.txt` | Socket option |
| 124 | `smbget -N -O smb://127.0.0.1:4455/share/file.txt` | `-O` consumes the URL |
| 125 | `smbget -N -R host smb://127.0.0.1:4455/share/file.txt` | Name-resolution order |
| 126 | `smbget -N -R smb://127.0.0.1:4455/share/file.txt` | `-R` consumes the URL |
| 127 | `smbget -N -R bogus smb://127.0.0.1:4455/share/file.txt` | Invalid order warning |
| 128 | `smbget -N -s /tmp/nosuch.conf smb://127.0.0.1:4455/share/file.txt` | Missing config file |
| 129 | `smbget -N --option='client min protocol=SMB2' smb://127.0.0.1:4455/share/file.txt` | Override one smb.conf option |
| 130 | `smbget -N --option=bogus smb://127.0.0.1:4455/share/file.txt` | Malformed option override |
| 131 | `smbget -N -l ~/smbget-demo/logs smb://127.0.0.1:4455/share/file.txt` | Log basename in a missing directory |
| 132 | `smbget -N -d 3 smb://127.0.0.1:4455/share/file.txt` | Debug level 3 |
| 133 | `smbget -N --debug-stdout -d 3 smb://127.0.0.1:4455/share/file.txt` | Debug to stdout |
| 134 | `smbget -N --leak-report smb://127.0.0.1:4455/share/file.txt` | Talloc leak report |
| 135 | `smbget -N --leak-report-full smb://127.0.0.1:4455/share/file.txt` | Full leak report |
| 136 | `smbget -N -e smb://127.0.0.1:4455/share/file.txt` | Require encryption |
| 137 | `smbget -N --client-protection=sign smb://127.0.0.1:4455/share/file.txt` | Require signing |
| 138 | `smbget -N --client-protection=encrypt smb://127.0.0.1:4455/share/file.txt` | Require encryption via protection |
| 139 | `smbget -N --client-protection=bogus smb://127.0.0.1:4455/share/file.txt` | Invalid protection mode |

**Research questions**
- Which SMB dialect is negotiated when `-m` is omitted?
- Does an invalid `-m` abort or warn?
- What does `-O` actually do in this build?
- Does `-R` ever mean recursive in this build?
- Which protection modes work against a server without encryption?

**Sources:** `help.txt`; `man smbget`; observed output.
**Output:** a protocol/option table and a warning inventory.

## Scenario 7 — failure modes and exit codes

| # | Command | Purpose |
|---|---------|---------|
| 140 | `smbget` | No arguments |
| 141 | `smbget smb://` | Empty SMB URL |
| 142 | `smbget smb://127.0.0.1` | Host without a share |
| 143 | `smbget smb://127.0.0.1/` | Host with an empty share |
| 144 | `smbget http://127.0.0.1/share/file.txt` | Wrong scheme |
| 145 | `smbget /etc/hosts` | Local path instead of a URL |
| 146 | `smbget nosuch.invalid/share/file.txt` | Host without the `smb://` scheme |
| 147 | `smbget -N smb://127.0.0.1:4455/share/nosuch.txt` | Missing remote file |
| 148 | `smbget -N smb://127.0.0.1:4455/nosuch/file.txt` | Missing share |
| 149 | `smbget -N smb://127.0.0.1:4454/share/file.txt` | Refused port |
| 150 | `smbget -N smb://10.255.255.1:4455/share/file.txt` | Unroutable host |
| 151 | `smbget -N smb://127.0.0.1:4455/share/file.txt` | Existing destination |
| 152 | `smbget -N -o existing.txt smb://127.0.0.1:4455/share/file.txt` | Existing named destination |
| 153 | `smbget -N --stdout -o out.bin smb://127.0.0.1:4455/share/file.txt` | Conflicting output modes |
| 154 | `smbget -N --recursive -o out.bin smb://127.0.0.1:4455/share/` | Recursive plus output file |
| 155 | `smbget -N --recursive --stdout smb://127.0.0.1:4455/share/` | Recursive plus stdout |
| 156 | `smbget -N -o /root/forbidden/out.bin smb://127.0.0.1:4455/share/file.txt` | Unwritable destination |
| 157 | `smbget -A /tmp/nosuch.auth smb://127.0.0.1:4455/share/file.txt` | Missing authentication file |
| 158 | `smbget -P smb://127.0.0.1:4455/share/file.txt` | Machine account without a domain |
| 159 | `smbget -N --use-kerberos=required smb://127.0.0.1:4455/share/file.txt` | Kerberos without a cache |
| 160 | `smbget --bogus` | Unknown option |
| 161 | `smbget -o` | Missing output filename |
| 162 | `smbget --limit-rate` | Missing rate value |
| 163 | `smbget -b abc smb://127.0.0.1:4455/share/file.txt` | Non-numeric block size |
| 164 | `smbget -N -q smb://127.0.0.1:4455/share/nosuch.txt` | Quiet still reports errors |
| 165 | `for u in smb:// smb://127.0.0.1 smb://127.0.0.1/ http://x /etc/hosts; do smbget "$u" >/dev/null 2>&1; printf '%s %s\n' "$?" "$u"; done` | Exit-code sweep |

**Research questions**
- Which failures exit 1 and which exit 0?
- Is a missing remote file distinguishable from a missing share?
- Do option conflicts fail the command or silently succeed?
- Does `-q` suppress fatal errors?
- Which errors happen before any network connection?

**Sources:** observed output; `help.txt`.
**Output:** a failure matrix with exit codes and messages.

## Scenario 8 — cleanup and evidence handling

| # | Command | Purpose |
|---|---------|---------|
| 166 | `find . -type f -exec sha256sum {} + > evidence/downloads-sha256.txt` | Hash the downloaded evidence |
| 167 | `tar -czf evidence/downloads.tar.gz single recursive resume update` | Archive the tree |
| 168 | `chmod 700 evidence` | Restrict access |
| 169 | `umask 077` | Future files are private |
| 170 | `shred -u auth.txt pw.txt` | Destroy credential files |
| 171 | `unset SMBGET_PASS` | Clear the environment secret |
| 172 | `history \| grep -i smbget` | Check shell history for secrets |
| 173 | `pgrep -af smbget` | No leftover transfer process |
| 174 | `rm -rf single recursive resume update` | Remove downloaded trees |
| 175 | `test ! -e single; echo "removed=$?"` | Verify removal |
| 176 | `rm -rf ~/smbget-demo` | Remove the demo tree |
| 177 | `test ! -e ~/smbget-demo; echo "removed=$?"` | Verify cleanup |
| 178 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-end.txt` | Timestamp the end |
| 179 | `ls -l evidence` | Final evidence inventory |

**Research questions**
- Which files contain credentials or downloaded personal data?
- Does `smbget` leave temporary files outside the destination?
- What should be hashed before deleting a download?
- How do you prove the workspace is clean?

**Sources:** observed output; data-handling practice.
**Output:** an evidence-handling checklist and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-V`, `--help`, `--usage`, `--bogus`, missing arguments, package provenance | 1 |
| `-N`, `-a`, `-U`, `--password`, URL credentials, `-W`, `-A`, `PASSWD`, `--pw-nt-hash`, Kerberos, `-P` | 2 |
| single file, `-o`, basename flattening, multiple URLs, `--stdout`, `-q`, `-v`, `-D`, modes | 3 |
| `--recursive`, trailing slash, subdirectory root, `-o`/`--stdout` conflicts, file URL | 4 |
| `-r`, `-u`, `-D`, `-v`, `-q`, `--limit-rate`, `-b`, invalid numbers | 5 |
| `-m`, `-n`, `--netbios-scope`, `--realm`, `-O`, `-R`, `-s`, `--option`, `-l`, `-d`, leak reports, `-e`, protection | 6 |
| no args, bad URLs, missing file/share, refused/unroutable host, conflicts, auth failures, exit codes | 7 |
| hashing, archiving, permissions, shredding, history, cleanup | 8 |

Full text: [`help.txt`](help.txt); authoritative description in `man smbget`
and the Samba project documentation.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binaries, package provenance | 18 |
| 2 | URL forms and authentication | 23 |
| 3 | Single-file downloads and output naming | 22 |
| 4 | Recursive downloads and layout | 21 |
| 5 | Resume, update, progress, and throttling | 31 |
| 6 | SMB protocol, name resolution, and diagnostics | 24 |
| 7 | Failure modes and exit codes | 26 |
| 8 | Cleanup and evidence handling | 14 |
| **Total** | | **179** |

## Legal and safety

Downloading from a share reads other people's files. Only download from a
server you own or are explicitly authorised to assess. Guest access and
credential reuse are not authorised by permission to scan a network. Passwords
and hashes on the command line leak into shell history and process listings;
use `PASSWD`, `PASSWD_FILE`, or a mode-0600 authentication file, and clear it
afterwards. Downloaded files can contain personal data, credentials, and
regulated information: hash them before deletion if they are evidence, store
them with restrictive permissions, and destroy every copy when the engagement
ends.
