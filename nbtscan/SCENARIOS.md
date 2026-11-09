# nbtscan — command scenarios for deep research

Installed build: NBTscan 1.7.2. Flags come from `help.txt`.

## How to invoke

```text
@deep-research Which NetBIOS name suffixes and scan settings give the most
reliable host/domain inventory on modern mixed Windows/Samba networks where
UDP/137 may be filtered, and when should nbtscan be replaced by mDNS, LLMNR, or
SMB enumeration? Cite NetBIOS specifications, Samba docs, and field reports;
return an ordered discovery workflow with fallbacks.
```

---

## Scenario 1 — basic subnet inventory

| # | Command | How to use |
|---|---------|------------|
| 1 | `nbtscan 192.168.1.0/24` | Default scan of a /24 |
| 2 | `nbtscan 192.168.1.25-137` | Explicit address range |
| 3 | `nbtscan -q 192.168.1.0/24` | Suppress banners |
| 4 | `nbtscan -t 2000 -m 1 192.168.1.0/24` | Longer timeout, one retransmit |
| 5 | `nbtscan -b 64000 10.10.0.0/24` | Throttle on a slow link |

**Research questions**
- What timeout/retransmit combination suits a high-latency VPN?
- How often do hosts answer only on the second retransmit?

**Sources:** nbtscan man page, packet captures.
**Output:** a host/name/domain table.

---

## Scenario 2 — script-friendly output

| # | Command | How to use |
|---|---------|------------|
| 6 | `nbtscan -v -s : 192.168.1.0/24` | All names, colon-separated |
| 7 | `nbtscan -v -h -s , 192.168.1.0/24 > hosts.csv` | Service names, CSV |
| 8 | `nbtscan -e 192.168.1.0/24` | `/etc/hosts` format |
| 9 | `nbtscan -l 192.168.1.0/24` | `lmhosts` format |
| 10 | `nbtscan -d 192.168.1.10` | Dump raw packets for one host |

**Research questions**
- Which suffix codes distinguish domain controllers from workstations?
- Does `-e` emit aliases that would break `/etc/hosts` parsing?

**Sources:** NetBIOS suffix table, nbtscan output samples.
**Output:** CSV, hosts file, or raw packet dump.

---

## Scenario 3 — large and file-driven scopes

| # | Command | How to use |
|---|---------|------------|
| 11 | `nbtscan -f targets.txt` | Read IPs from a file |
| 12 | `cat scope.txt \| nbtscan -f -` | Read IPs from stdin |
| 13 | `nbtscan -q -s , 10.0.0.0/16 > big.csv` | Large sweep to CSV |
| 14 | `nbtscan -r 192.168.1.0/24` | Source port 137 (root) |
| 15 | `split -l 256 scope.txt part- && for p in part-*; do nbtscan -q -f "$p"; done` | Chunked scanning |

**Research questions**
- Does `-r` improve response rates on legacy hosts enough to justify root?
- What is a safe chunk size for a stateful firewall's UDP session table?

**Sources:** firewall docs, Samba behaviour notes.
**Output:** a complete inventory across a large scope.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Output | `-v -d -e -l -q -s -h` | 1, 2 |
| Timing | `-t -b -m` | 1, 3 |
| Targeting | `-f -r <range>` | 3 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — basic inventory | 5 |
| 2 — script output | 5 |
| 3 — large scopes | 5 |
