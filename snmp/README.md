# snmp — Net-SNMP command-line suite (get, walk, bulkwalk, set, translate)

- **Version:** 5.9.4.pre2 (`snmpwalk -V` reports `NET-SNMP version: 5.9.4.pre2`)
- **Binaries:** `/usr/bin/snmpget`, `/usr/bin/snmpwalk`, `/usr/bin/snmpbulkwalk`,
  `/usr/bin/snmpset`, `/usr/bin/snmptranslate`; the same package also installs
  `snmpgetnext`, `snmpbulkget`, `snmptable`, `snmptrap`, `snmpdf`, `snmpstatus`,
  `snmpnetstat`, `snmpusm`, `snmpvacm`, `snmpconf`, `snmptls`, `snmpps`,
  `snmpdelta`, and `snmptest`.
- **Full help snapshots:** [`help-snmpget.txt`](help-snmpget.txt),
  [`help-snmpwalk.txt`](help-snmpwalk.txt), [`help-snmpbulkwalk.txt`](help-snmpbulkwalk.txt),
  [`help-snmpset.txt`](help-snmpset.txt), [`help-snmptranslate.txt`](help-snmptranslate.txt)
- **Upstream:** https://github.com/net-snmp/net-snmp
- **Docs:** http://www.net-snmp.org/docs/man/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Net-SNMP is the reference open-source SNMP implementation. These five commands are
the client half of it, and they share one option parser, so version, authentication,
timeout, MIB, and output flags behave identically across them:

| Command | Operation |
|---------|-----------|
| `snmpget` | Retrieve one or more OIDs (`GET`) |
| `snmpwalk` | Walk a subtree with repeated `GETNEXT` |
| `snmpbulkwalk` | Walk a subtree with `GETBULK` (v2c/v3, far fewer round trips) |
| `snmpset` | Write one or more OIDs (`SET`) |
| `snmptranslate` | Convert between numeric OIDs and MIB names; print MIB structure |

SNMP is the management protocol of routers, switches, firewalls, printers, UPSes,
NAS boxes, BMCs/IPMI controllers, and monitoring agents. Read access commonly leaks
hostname, uptime, interface counters, ARP and route tables, installed software,
process lists (through extensible agents), and occasionally credentials or community
strings stored in vendor OIDs. It runs on **UDP 161**; traps go to **UDP 162**.

## Use cases

| Situation | Why this suite |
|-----------|----------------|
| Network inventory | `snmpwalk` the system and interface subtrees of each device |
| Fast table reads on slow gear | `snmpbulkwalk` with `-Cn`/`-Cr` window tuning |
| Verifying a community string | `snmpget` one OID instead of a full walk |
| Mapping numeric OIDs from a capture | `snmptranslate -On` / `-Td` |
| v3 migration checks | Compare `noAuthNoPriv`, `authNoPriv`, `authPriv` results |
| Monitoring/parsing scripts | `-Oq` / `-OQ` / `-On` machine-readable output |

## Requirements & install layout

- Installed from the Debian `snmp` package; binaries live in `/usr/bin`, and no root
  privileges are needed for client operations.
- Base MIBs come from `libsnmp-base` under `/usr/share/snmp/mibs` (with `iana/` and
  `ietf/`). Default search path per help: `$HOME/.snmp/mibs:/usr/share/snmp/mibs:`
  `/usr/share/snmp/mibs/iana:/usr/share/snmp/mibs/ietf`; vendor MIBs go in `~/.snmp/mibs`.
- Config: system `/etc/snmp/snmp.conf`, per-user `$HOME/.snmp/snmp.conf`.
  `snmpwalk -H` prints every configuration directive the build understands.
- Network: outbound UDP/161 to the agent; no listening port is opened.
- Timeouts/retries are not stated in the help snapshots — set `-t` and `-r` explicitly.

## Basic syntax

```bash
snmpget  -v2c -c public 10.10.10.5 SNMPv2-MIB::sysDescr.0
snmpwalk -v2c -c public 10.10.10.5 1.3.6.1.2.1.1
snmpbulkwalk -v2c -c public -Cn0 -Cr25 10.10.10.5 1.3.6.1.2.1.2.2
snmptranslate -On SNMPv2-MIB::sysUpTime.0
snmpset -v2c -c private 10.10.10.5 SNMPv2-MIB::sysLocation.0 s "lab rack 4"
```

## Flag reference

### Shared options (all five commands)

| Flag | Meaning |
|------|---------|
| `-h, --help` | Display the help message |
| `-H` | Display the configuration-file directives understood |
| `-v 1\|2c\|3` | SNMP version to use |
| `-V, --version` | Display the package version |
| `-c COMMUNITY` | Community string (v1/v2c only) |
| `-a PROTOCOL` | v3 authentication protocol: `MD5`, `SHA`, `SHA-224`, `SHA-256`, `SHA-384`, `SHA-512` |
| `-A PASSPHRASE` | v3 authentication pass phrase |
| `-e ENGINE-ID` / `-E ENGINE-ID` | Security engine ID / context engine ID (`800000020109840301`) |
| `-l LEVEL` | Security level: `noAuthNoPriv`, `authNoPriv`, `authPriv` |
| `-n CONTEXT` | Context name, e.g. `bridge1` |
| `-u USER-NAME` | Security name (v3 user) |
| `-x PROTOCOL` / `-X PASSPHRASE` | Privacy protocol (`DES`, `AES`, `AES-192`, `AES-256`) / privacy pass phrase |
| `-Z BOOTS,TIME` | Destination engine boots/time |
| `-r RETRIES` / `-t TIMEOUT` | Retries / request timeout in seconds |
| `-d` / `-D[TOKEN[,...]]` | Hex packet dump / debug tokens (`ALL` is extremely verbose) |
| `-m MIB[:...]` / `-M DIR[:...]` | Load MIBs (`ALL` = everything) / MIB search directories |
| `-P MIBOPTS` | MIB-parsing toggles (below) |
| `-O OUTOPTS` | Output-display toggles (below) |
| `-I INOPTS` | Input-parsing toggles (below) |
| `-L LOGOPTS` | Logging toggles (below) |
| `-C APPOPTS` | Application-specific behaviour (per command, below) |

### MIB parsing (`-P`)

| Flag | Meaning |
|------|---------|
| `-Pu` / `-Pc` / `-Pd` | Allow underscores in symbols / allow `--` comments / save DESCRIPTIONs |
| `-Pe` / `-Pw` / `-PW` / `-PR` | Suppress conflict errors / warn / detailed warn / replace symbols from the latest module |

### Output (`-O`)

| Flag | Meaning |
|------|---------|
| `-O0` / `-Oa` / `-Ob` | Pad single-digit hex / strings as ASCII / do not break OID indexes down |
| `-Oe` / `-OE` / `-Of` | Enums numerically / escape quotes in string indices / full OIDs |
| `-On` / `-Op PREC` / `-Oq` | Numeric OIDs / float precision / quick print for parsing |
| `-OQ` / `-Os` / `-OS` | Quick print with `=` / last symbolic element / module-id plus last element |
| `-Ot` / `-OT` / `-Ou` | Timeticks as integers / readable text beside hex / UCD-style prefix suppression |
| `-OU` / `-Ov` / `-Ox` / `-OX` | No units / values only / strings as hex / extended index format |

### Input parsing (`-I`)

| Flag | Meaning |
|------|---------|
| `-Ib` / `-Ih` / `-Ir` | Best/regex MIB match / ignore DISPLAY-HINTs / skip range and type checks |
| `-IR` / `-Iu` | Random access to OID labels / require a leading `.` on top-level OIDs |
| `-Is SUFFIX` / `-IS PREFIX` | Append / prepend text to every textual OID before parsing |

### Logging (`-L`)

| Flag | Meaning |
|------|---------|
| `-Le` / `-Lo` / `-Ln` | Log to stderr / stdout / nowhere |
| `-Lf FILE` / `-Ls FACILITY` | Log to a file / syslog facility |
| `-L[EON]pri` / `-L[EON]p1-p2` | Threshold logging to stderr, output, or `/dev/null` |
| `-L[FS]pri token` / `-L[FS]p1-p2 token` | Threshold logging to file or syslog |

### Per-command usage and extra options

| Command | Usage | Extra flags |
|---------|-------|-------------|
| `snmpwalk` | `snmpwalk [OPTIONS] AGENT [OID]` | `-C p` variable count; `-C i` include OID in range; `-C I` don't; `-C c` skip increasing-OID check; `-C t` total wall-clock time; `-C T` per-request time; `-C E {OID}` end the walk at OID |
| `snmpget` | `snmpget [OPTIONS] AGENT OID [OID]...` | `-C f` do not fix errors and retry |
| `snmpbulkwalk` | `snmpbulkwalk [OPTIONS] AGENT [OID]` | `-C c` skip increasing-OID check; `-C i` include OIDs in range; `-C n<NUM>` non-repeaters; `-C p` variable count; `-C r<NUM>` max-repeaters |
| `snmpset` | `snmpset [OPTIONS] AGENT OID TYPE VALUE [...]` | `-C q` don't print results on success |
| `snmptranslate` | `snmptranslate [OPTIONS] OID [OID]...` | `-w WIDTH` tree/detail width; `-T B` all regex matches; `-T d` full details; `-T p` tree; `-T a` ASCII table; `-T l` labelled report; `-T o` OID report; `-T s` dotted symbolic; `-T z` MIB child report; `-T t` alternate suffix report |

`snmpset` TYPE codes: `i` INTEGER, `u` unsigned, `t` TIMETICKS, `a` IPADDRESS,
`o` OBJID, `s` STRING, `x` HEX STRING, `d` DECIMAL STRING, `b` BITS,
`U` unsigned int64, `I` signed int64, `F` float, `D` double.

`snmptranslate` is an offline MIB tool: it takes no agent, no `-v`, and no community.

## Community strings and SNMP versions (security context)

- **SNMPv1 and v2c** authenticate with a single shared **community string** sent in
  cleartext in every datagram. There is no per-user identity and no integrity or
  encryption. Anyone who can sniff one packet, read a device backup, or guess the
  string gets exactly the access the agent grants that community — read-only or
  read-write. The distinction between `-c public` and `-c private` is convention and
  agent configuration, not protocol enforcement.
- **Default strings are common.** `onesixtyone` ships `public private` as its built-in
  list for precisely this reason; treat any default-community device as unauthenticated.
- **SNMPv3** adds the User-based Security Model. `-l noAuthNoPriv` sends a user name
  with no protection; `-l authNoPriv` adds message authentication and integrity using
  `-a`/`-A`; `-l authPriv` adds encryption using `-x`/`-X`. Only `authPriv` resists
  passive capture. Engine ID (`-e`/`-E`), context (`-n`), and boots/time (`-Z`) matter
  for v3 interoperability and for reproducing a specific engine state.
- **Practical rule:** if a v2c community works, you have the same authority as that
  community string. Do not assume read-only because the string looks like `public`.

## Recipes

```bash
# 1. Full system subtree, machine-readable (value per line)
snmpwalk -v2c -c public -Oq 10.10.10.5 1.3.6.1.2.1.1
# 2. Interface table via GETBULK, numeric OIDs, saved for parsing
snmpbulkwalk -v2c -c public -Cn0 -Cr25 -Oqn 10.10.10.5 1.3.6.1.2.1.2.2 > ifaces.txt
# 3. One OID, value only (fast check that a community is valid)
snmpget -v2c -c public -Oqv 10.10.10.5 SNMPv2-MIB::sysUpTime.0
# 4. v3 authPriv walk
snmpwalk -v3 -l authPriv -u monitor -a SHA-256 -A 'auth-pass' \
  -x AES -X 'priv-pass' 10.10.10.5 1.3.6.1.2.1.1
# 5. Translate both directions
snmptranslate -On SNMPv2-MIB::sysDescr.0        # name -> .1.3.6.1.2.1.1.1.0
snmptranslate -Td SNMPv2-MIB::sysUpTime.0       # full detail for one object
# 6. Time a slow walk and cap the request window
snmpwalk -v2c -c public -Ct -t 3 -r 1 10.10.10.5 1.3.6.1.2.1.4
# 7. INTRUSIVE: write a value (changes device state; authorization required)
snmpset -v2c -c private 10.10.10.5 SNMPv2-MIB::sysContact.0 s "noc@example.net"
```

## Output & pipelines

Default output is `OID = TYPE: VALUE`. For scripts use `-Oq` (quick), `-OQ`
(quick with `=`), `-Oqv` (value only), `-Oqn` (numeric OID and value), `-Ov`
(values only), or `-Ox` (hex). `-On` keeps OIDs numeric so downstream tools do not
need MIBs; `snmptranslate -Td` adds the names back later.

Typical handoffs: walk output to CSV/awk, numeric OIDs to a CMDB, `sysDescr` and
`sysName` to an inventory sheet, ARP tables (`1.3.6.1.2.1.4.22`) to feed host
discovery, and `-Ot`/`-Ct` timings to decide whether `snmpbulkwalk` is needed.

## Gotchas & operational security

- Missing MIBs produce `Cannot find module` warnings; numeric OIDs still work, and
  `-On` avoids the problem entirely.
- `-m ALL` only helps if the MIB files exist. Vendor MIBs belong in `~/.snmp/mibs`
  and must be found through `-M`.
- `snmpbulkwalk` depends on GETBULK, which is a v2c/v3 feature. Use `-v2c` or `-v3`,
  and tune with `-Cn`/`-Cr`; large windows can overrun fragile agents.
- Agents that return non-increasing OIDs break walks — `-Cc` disables the check.
- UDP has no delivery guarantee: raise `-t`/`-r` on lossy links, and expect printers,
  UPSes, and BMCs to be slow.
- A community string on the command line is visible in `ps` and shell history. Use
  `~/.snmp/snmp.conf` (list supported directives with `-H`) for recurring credentials.
- Walking full tables generates substantial traffic and is logged by managed switches;
  a single `snmpget` is far quieter.
- `snmpset` has no dry-run and no undo. Wrong type codes (`i` vs `u` vs `s`) are
  rejected or, worse, accepted with a different meaning.

## Legal & safety

Use only on devices you own or are **explicitly authorized** to query. Reading
`sysDescr` is low-impact but still active probing and may be logged or alerted on.
`snmpset` is a **write** to production configuration and must be treated as a change
request: authorization, maintenance window, and a re-read to confirm the value. Do
not walk networks outside the agreed scope, and never use SNMP write access to disable
logging, change communities, or alter routing as a "test".

## Related tools

- [`onesixtyone/`](../onesixtyone/README.md) — fast community-string sweeps before walking.
- [`wireshark/`](../wireshark/README.md) — decode SNMP packets and confirm community exposure.
- [`nbtscan/`](../nbtscan/README.md) — companion legacy-protocol discovery on the same LAN.

## References

- Net-SNMP manual pages: http://www.net-snmp.org/docs/man/
- SNMPv3 background shipped on this host: `/usr/share/doc/libsnmp-base/README.snmpv3`
- MIB loading notes: `/usr/share/doc/libsnmp-base/README.mibs`
- Upstream source: https://github.com/net-snmp/net-snmp
