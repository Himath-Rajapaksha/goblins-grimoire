# smbclient — command scenarios for deep research

Installed build: Samba 4.22.11-Debian. Flags come from [`help.txt`](help.txt).
SMB1 is disabled by default; legacy servers may need `-m NT1`.

> **Sensitive:** share access can expose personal and business data. Use only
> within written scope, and minimize what you download.

## How to invoke

```text
@deep-research Which smbclient checks distinguish a misconfigured share from a
properly restricted one, and how should null-session, guest, and authenticated
access be tested in order without locking accounts or copying unnecessary
data? Cite the Samba smbclient man page, SMB signing documentation, and field
reports; return a test matrix with a stop condition.
```

Prepare the demo directory:

```bash
mkdir -p ~/smbclient-demo/evidence
printf 'username=ACME/alice\npassword=Password123\ndomain=ACME\n' > ~/smbclient-demo/auth.txt
chmod 600 ~/smbclient-demo/auth.txt
```

Use lab credentials only; do not commit the auth file.

## Scenario 1 — share inventory

Start with a list, not a connection. The share table tells you what exists and
which shares are IPC or printer endpoints.

| # | Command | How to use |
|---|---------|------------|
| 1 | `smbclient -L //10.10.10.5 -N` | Null-session list |
| 2 | `smbclient -L //10.10.10.5 -U 'ACME/alice%Password123'` | Authenticated list |
| 3 | `smbclient -L //10.10.10.5 -A ~/smbclient-demo/auth.txt` | Auth file (no password in `ps`) |
| 4 | `smbclient -L //10.10.10.5 -N -g` | Grepable output |
| 5 | `smbclient -L //10.10.10.5 -N -d 3` | Debug dialect negotiation |

**Research questions**
- Which shares are visible anonymously versus authenticated?
- Which shares are `Disk`, `IPC`, or `Printer`?
- Does the server negotiate SMB2/3, or require SMB1?

**Sources:** smbclient man page; SMB dialect docs; server config (if provided).
**Output:** a share table with access level and type.

## Scenario 2 — confirm access and browse

A share name in a list does not prove read access. Connect and list the root,
then check a subdirectory.

| # | Command | How to use |
|---|---------|------------|
| 6 | `smbclient //10.10.10.5/Finance -U 'ACME/alice%Password123'` | Interactive session |
| 7 | `smbclient //10.10.10.5/Finance -U alice -c 'ls'` | Non-interactive list |
| 8 | `smbclient //10.10.10.5/Finance -U alice -c 'cd reports; ls'` | Subdirectory |
| 9 | `smbclient //10.10.10.5/Finance -U alice -c 'allinfo q4.pdf'` | Metadata |
| 10 | `smbclient //10.10.10.5/Finance -U alice -c 'showacls'` | ACL view |

**Research questions**
- Does the share allow read, write, or both?
- Are ACLs inherited or explicitly set?
- Does the share expose more than the application intends?

**Sources:** smbclient man page; ACL output; share permissions.
**Output:** an access matrix (read/write/execute) per share.

## Scenario 3 — credential variants

The same share may be reachable with a password, an NT hash, Kerberos, or a
machine account. Test the variants the engagement provides, not guesses.

| # | Command | How to use |
|---|---------|------------|
| 11 | `smbclient //10.10.10.5/share -U 'ACME/alice%Password123'` | Plaintext |
| 12 | `smbclient //10.10.10.5/share -U 'ACME/alice' --pw-nt-hash <nthash>` | Pass-the-hash |
| 13 | `KRB5CCNAME=~/acme.ccache smbclient //dc.acme.local/SYSVOL --use-kerberos=required -U alice` | Kerberos |
| 14 | `smbclient //10.10.10.5/share -P` | Machine account |
| 15 | `smbclient //10.10.10.5/share -A ~/smbclient-demo/auth.txt` | Auth file |

**Research questions**
- Which authentication methods does the server accept?
- Does pass-the-hash work where plaintext is blocked?
- Is signing/encryption required, and does it change access?

**Sources:** Samba docs; netexec comparison; Kerberos ticket inspection.
**Output:** an auth-method matrix with the result of each.

## Scenario 4 — controlled file transfer

Download only what the finding requires. Use `mask`, `prompt`, and a local
directory to avoid copying an entire share.

| # | Command | How to use |
|---|---------|------------|
| 16 | `smbclient //10.10.10.5/share -U alice -c 'get reports/q4.pdf ~/smbclient-demo/evidence/q4.pdf'` | One file |
| 17 | `smbclient //10.10.10.5/share -U alice -c 'mask *.txt; prompt off; mget *'` | Patterned batch |
| 18 | `smbclient //10.10.10.5/share -U alice -c 'du'` | Size before copying |
| 19 | `smbclient //10.10.10.5/share -U alice -c 'recurse on; prompt off; mget *'` | Recursive (careful) |
| 20 | `smbclient //10.10.10.5/share -U alice -T x backup.tar` | Tar extract |

**Research questions**
- How large is the share, and what is the smallest evidence sample?
- Does the download preserve timestamps and metadata?
- Are there hidden or system files that `mask` skips?

**Sources:** smbclient man page; `du` output; file hashes.
**Output:** a minimal evidence set with hashes and a data-handling note.

## Scenario 5 — write access and impact

Write tests change state. Only run them when the rules of engagement allow it,
and clean up immediately.

| # | Command | How to use |
|---|---------|------------|
| 21 | `smbclient //10.10.10.5/share -U alice -c 'put /tmp/probe.txt probe.txt'` | Upload a probe |
| 22 | `smbclient //10.10.10.5/share -U alice -c 'mkdir _probe_dir'` | Create a directory |
| 23 | `smbclient //10.10.10.5/share -U alice -c 'del probe.txt'` | Delete the probe |
| 24 | `smbclient //10.10.10.5/share -U alice -c 'rmdir _probe_dir'` | Remove the directory |
| 25 | `smbclient //10.10.10.5/share -U alice -c 'ls'` | Verify cleanup |

**Research questions**
- Does the share allow write, or only read?
- Can a low-privileged user create files that others execute?
- Is the probe visible to the application, and could it be served?

**Sources:** share permissions; application behavior; cleanup verification.
**Output:** a write-access finding with before/after listings.

## Scenario 6 — evidence and cleanup

Record the share, credentials used (redacted), commands, and file hashes.
Remove downloaded data and auth files at close.

| # | Command | How to use |
|---|---------|------------|
| 26 | `smbclient //10.10.10.5/share -U alice -c 'ls' > ~/smbclient-demo/evidence/ls.txt` | Listing evidence |
| 27 | `sha256sum ~/smbclient-demo/evidence/* > ~/smbclient-demo/evidence/hashes.txt` | Integrity |
| 28 | `smbclient -L //10.10.10.5 -A ~/smbclient-demo/auth.txt -g > ~/smbclient-demo/evidence/shares.txt` | Share evidence |
| 29 | `shred -u ~/smbclient-demo/auth.txt` | Remove credentials |
| 30 | `rm -rf ~/smbclient-demo/evidence/downloads` | Remove copied data |

**Research questions**
- Does the evidence identify the exact share and path?
- Are credentials redacted in the report?
- Is downloaded data removed or archived per policy?

**Sources:** engagement retention policy; file hashes; report template.
**Output:** an evidence bundle and a cleanup checklist.
