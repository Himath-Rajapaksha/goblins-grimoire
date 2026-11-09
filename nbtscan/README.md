# nbtscan — NetBIOS name and address scanner

- **Version:** NBTscan 1.7.2
- **Binary:** `/usr/bin/nbtscan`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/radiator-software/nbtscan (Debian package `nbtscan`)

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

nbtscan sends NetBIOS Name Service queries (UDP 137) to a range of addresses and
prints the NetBIOS names each host has registered: computer name, workgroup or
domain, logged-in user, and service suffixes. It is a fast way to inventory
legacy Windows/Samba hosts and to spot domain controllers and file servers on a
flat LAN.

## Use cases

| Situation | Why nbtscan |
|-----------|-----------|
| Flat internal subnet | Enumerate Windows/Samba hosts without credentials |
| Domain mapping | Identify domain/workgroup names and DC service records |
| Legacy host inventory | Find old machines that ignore modern discovery |
| Pre-SMB enumeration | Feed hostnames into `netexec`/`enum4linux-ng` |
| Rogue device check | Spot unexpected NetBIOS responders |

## Requirements & install layout

- Debian package 1.7.2; binary `/usr/bin/nbtscan`.
- No root by default; `-r` (source port 137) requires root.
- Pure UDP/137; no config or state files.
- NetBIOS is often disabled on modern networks, so silence is not proof of
  absence.

## Basic syntax

```bash
# scan a /24
nbtscan 192.168.1.0/24

# scan an explicit range
nbtscan 192.168.1.25-137

# verbose, script-friendly, colon-separated
nbtscan -v -s : 192.168.1.0/24

# read targets from a file or stdin
nbtscan -f targets.txt
cat targets.txt | nbtscan -f -
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-v` | Verbose: print every name received from each host |
| `-d` | Dump whole packet contents |
| `-e` | Output in `/etc/hosts` format |
| `-l` | Output in `lmhosts` format (cannot combine with `-v`, `-s`, `-h`) |
| `-t <ms>` | Response timeout in milliseconds (default 1000) |
| `-b <bps>` | Throttle output/query rate to this bandwidth |
| `-r` | Use local port 137 (root; needed for Win95) |
| `-q` | Suppress banners and error messages |
| `-s <sep>` | Script-friendly output with a custom field separator |
| `-h` | Human-readable service names (only with `-v`) |
| `-m <n>` | Number of retransmits (default 0) |
| `-f <file>` | Read IPs from a file; `-f -` reads stdin |
| `<range>` | Single IP, CIDR, or `start-end` range |

## Recipes

```bash
# 1. Standard /24 sweep
nbtscan 192.168.1.0/24

# 2. Parse-friendly output with service names
nbtscan -v -h -s : 192.168.1.0/24 | tee nbtscan.txt

# 3. Throttled scan over a slow VPN link
nbtscan -b 64000 -t 2000 -m 2 10.10.0.0/24

# 4. Quiet scan for scripts
nbtscan -q -s , 10.10.0.0/24 > hosts.csv

# 5. Hosts file entries
nbtscan -e 192.168.1.0/24 >> /etc/hosts

# 6. Target list from stdin
cat scope.txt | nbtscan -f -
```

## Output & pipelines

Default columns: IP, NetBIOS name, service suffix, and (with `-v`) all names.
`-s` makes it CSV/awk-friendly; `-e` and `-l` produce hosts-file formats.
Hostnames feed `netexec smb`, `enum4linux-ng`, and DNS reverse checks.

## Gotchas & operational security

- NetBIOS is broadcast/legacy; many networks block UDP/137, producing false
  negatives.
- `-r` needs root and can conflict with a local Samba/NetBIOS service bound to
  137.
- High timeouts on large ranges make scans slow; tune `-t` and `-m`.
- Names can be stale or spoofed; corroborate with SMB/DNS.
- Scanning a subnet is active probing and is usually logged.

## Legal & safety

Scan only networks you own or have **written authorization** to assess. Do not
use discovered names to access hosts outside scope.

## Related tools

- [`enum4linux-ng/`](../enum4linux-ng/README.md) — deeper SMB/AD enumeration
- [`netexec/`](../netexec/README.md) — SMB/LDAP/WinRM operations
- [`snmp/`](../snmp/README.md) — another unauthenticated inventory path
- [`onesixtyone/`](../onesixtyone/README.md) — SNMP community sweep

## References

- `man nbtscan`
- https://github.com/radiator-software/nbtscan
