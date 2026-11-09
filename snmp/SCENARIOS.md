# snmp — command scenarios for deep research

Installed build: Net-SNMP 5.9.4.pre2. Flags come from the five help snapshots
in this directory; all five commands share one option parser.

## How to invoke

```text
@deep-research For Net-SNMP 5.9.4, which OID subtrees give the highest-value
device inventory on mixed Cisco/Juniper/HP/printers/BMC estates with the least
traffic, and how should snmpbulkwalk windows (-Cn/-Cr) be tuned for fragile
agents? Cite Net-SNMP docs, vendor MIB notes, and field reports; return an
ordered OID checklist and safe walk settings.
```

---

## Scenario 1 — validate access cheaply

| # | Command | How to use |
|---|---------|------------|
| 1 | `snmpget -v2c -c public -Oqv 10.10.10.5 SNMPv2-MIB::sysDescr.0` | One OID, value only |
| 2 | `snmpget -v2c -c public -t 2 -r 1 -Oqv 10.10.10.5 1.3.6.1.2.1.1.1.0` | Numeric OID, tight timeout |
| 3 | `snmpwalk -v2c -c public -Oq 10.10.10.5 1.3.6.1.2.1.1` | Small system subtree |
| 4 | `snmpwalk -v3 -l authPriv -u monitor -a SHA-256 -A 'auth' -x AES -X 'priv' 10.10.10.5 1.3.6.1.2.1.1` | v3 authPriv check |
| 5 | `onesixtyone -c communities.txt 10.10.10.0/24` | Fast community sweep first |

**Research questions**
- Which single OID is least likely to be filtered by hardened agents?
- Does v3 `authNoPriv` ever succeed where `authPriv` fails due to engine IDs?

**Sources:** Net-SNMP man pages, vendor hardening guides.
**Output:** a confirmed agent and credential.

---

## Scenario 2 — inventory subtrees

| # | Command | How to use |
|---|---------|------------|
| 6 | `snmpwalk -v2c -c public -Oqn 10.10.10.5 1.3.6.1.2.1.1 > system.txt` | System group |
| 7 | `snmpbulkwalk -v2c -c public -Cn0 -Cr25 -Oqn 10.10.10.5 1.3.6.1.2.1.2.2 > ifaces.txt` | Interface table |
| 8 | `snmpwalk -v2c -c public -Oqn 10.10.10.5 1.3.6.1.2.1.4.22 > arp.txt` | ARP table |
| 9 | `snmpwalk -v2c -c public -Oqn 10.10.10.5 1.3.6.1.2.1.4.21 > routes.txt` | Routing table |
| 10 | `snmpwalk -v2c -c public -Oqn 10.10.10.5 1.3.6.1.2.1.25.4.2 > processes.txt` | Process table (host resources) |
| 11 | `snmpwalk -v2c -c public -Oqn 10.10.10.5 1.3.6.1.2.1.25.6.3 > software.txt` | Installed software |

**Research questions**
- Which subtrees are commonly disabled or ACL-filtered?
- How much traffic does the process/software subtree generate on a server?

**Sources:** Net-SNMP OID references, vendor MIB documentation.
**Output:** CSV/JSON-ready inventory files.

---

## Scenario 3 — bulk tuning and slow agents

| # | Command | How to use |
|---|---------|------------|
| 12 | `snmpbulkwalk -v2c -c public -Cn0 -Cr10 -t 3 -r 1 10.10.10.5 1.3.6.1.2.1.2.2` | Small window for fragile agents |
| 13 | `snmpbulkwalk -v2c -c public -Cn0 -Cr50 -t 1 -r 0 10.10.10.5 1.3.6.1.2.1.4.22` | Aggressive window on fast LAN |
| 14 | `snmpwalk -v2c -c public -Cc 10.10.10.5 1.3.6.1.2.1.2.2` | Skip increasing-OID check |
| 15 | `snmpwalk -v2c -c public -Ct -t 5 -r 2 10.10.10.5 1.3.6.1.2.1.4` | Time the walk |
| 16 | `snmpwalk -v2c -c public -CE 1.3.6.1.2.1.4.22.0 10.10.10.5 1.3.6.1.2.1.4` | Stop at an OID |

**Research questions**
- What window size causes drops on this agent model?
- Does `-Cc` hide a real agent bug?

**Sources:** Net-SNMP docs, packet loss measurements, agent logs.
**Output:** a tuned walk profile per device class.

---

## Scenario 4 — MIB translation and output shaping

| # | Command | How to use |
|---|---------|------------|
| 17 | `snmptranslate -On SNMPv2-MIB::sysDescr.0` | Name to numeric OID |
| 18 | `snmptranslate -Td SNMPv2-MIB::sysUpTime.0` | Full object detail |
| 19 | `snmptranslate -Tp -m ALL SNMPv2-MIB::system` | Subtree tree view |
| 20 | `snmpwalk -v2c -c public -Oqv 10.10.10.5 1.3.6.1.2.1.1` | Values only |
| 21 | `snmpwalk -v2c -c public -OQn 10.10.10.5 1.3.6.1.2.1.1` | `OID = value` for parsers |
| 22 | `snmpwalk -v2c -c public -Ox 10.10.10.5 1.3.6.1.2.1.1.5` | Hex strings |

**Research questions**
- Which output mode is most stable for awk/jq pipelines?
- Why do vendor MIBs conflict, and does `-Pe` resolve it safely?

**Sources:** snmptranslate help, MIB files, parser tests.
**Output:** machine-readable or human-readable output.

---

## Scenario 5 — controlled writes (high risk)

| # | Command | How to use |
|---|---------|------------|
| 23 | `snmpset -v2c -c private 10.10.10.5 SNMPv2-MIB::sysLocation.0 s "lab rack 4"` | Change a benign string |
| 24 | `snmpset -v2c -c private 10.10.10.5 SNMPv2-MIB::sysContact.0 s "noc@example.net"` | Change contact |
| 25 | `snmpset -v2c -c private -C q 10.10.10.5 1.3.6.1.2.1.1.6.0 s "noc@example.net"` | Quiet write |
| 26 | `snmpget -v2c -c private -Oqv 10.10.10.5 1.3.6.1.2.1.1.6.0` | Re-read to verify |
| 27 | `snmptranslate -On SNMPv2-MIB::sysLocation.0` | Confirm the OID before writing |

**Research questions**
- Which OIDs are safe to use for a write test without operational impact?
- Does the device log SNMP SETs, and where?

**Sources:** vendor configuration guide, change policy, device logs.
**Output:** a verified write with a change record.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Shared | `-h -H -v -V -c -a -A -e -E -l -n -u -x -X -Z -r -t -d -D -m -M -P -O -I -L -C` | 1–5 |
| snmpwalk `-C` | `p i I c t T E` | 3 |
| snmpbulkwalk `-C` | `c i n p r` | 3 |
| snmpset `-C` | `q` | 5 |
| snmptranslate `-T/-w` | `B d p a l o s z t` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — validate access | 5 |
| 2 — inventory | 6 |
| 3 — bulk tuning | 5 |
| 4 — MIB/output | 6 |
| 5 — controlled writes | 5 |
