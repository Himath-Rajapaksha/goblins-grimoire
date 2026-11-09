# wpa_supplicant — command scenarios for deep research

Installed build: wpasupplicant 2.10-24, with `/usr/sbin/wpa_supplicant`,
`/usr/sbin/wpa_cli`, and `/usr/bin/wpa_passphrase`. Every option below appears
in one of the three help snapshots; behaviour was observed on this host, which
has a live supplicant on `wlp2s0`.

> **Scope:** starting a supplicant changes interface state and can disconnect
> the machine you are working on. Scanning and associating are active radio
> operations. Only run these scenarios against networks and adapters you own or
> are explicitly authorised to test, and use a spare interface where possible.

> **Substitute the interface.** The examples use `wlp2s0`, the interface with a
> running supplicant on this host. Replace it with your own `iw dev` name. A
> wrong name produces exit 255, not a helpful message.

> **Privileges:** starting `wpa_supplicant`, changing interface state, and
> opening raw 802.11 operations need root or `CAP_NET_ADMIN`. `wpa_cli` needs
> access to the control socket, normally via `GROUP=netdev`.

## How to invoke

```text
@deep-research For an authorised Wi-Fi assessment with wpa_supplicant 2.10,
explain the daemon's startup path (`-D`, `-i`, `-c`, `-C`, `-g`, `-N`, `-b`),
how the configuration file expresses PSK, SAE, OWE, and EAP networks, how
`wpa_cli` speaks the control protocol over a UNIX socket, and how
`wpa_passphrase` derives a PBKDF2 PSK. Show which startup failures exit 255,
which `wpa_cli` commands return 0 even when the answer is `FAIL`, how the
control socket's group permissions form a local privilege boundary, and how
WPS, PMF, and SAE change the association. Cite the installed help text, the
upstream `wpa_supplicant.conf` documentation, and observed output. Separate
confirmed behaviour from inference and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/wpa-demo/evidence
cd ~/wpa-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
wpa_supplicant -v > evidence/version.txt
ip -brief link > evidence/links.txt
```

## Scenario 1 — version, licence, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `wpa_supplicant -v` | Daemon version |
| 2 | `wpa_supplicant -L` | BSD licence text |
| 3 | `wpa_supplicant -h` | Option and driver list |
| 4 | `wpa_supplicant -h \| grep -A6 '^drivers:'` | Supported drivers |
| 5 | `wpa_supplicant -h \| grep -c '^  -'` | Count short options |
| 6 | `wpa_cli -v` | Client version |
| 7 | `wpa_cli -h` | Client usage and command list |
| 8 | `wpa_cli -h \| grep -c '^  [a-z].* = '` | Count documented commands |
| 9 | `wpa_passphrase` | Usage when SSID is missing |
| 10 | `command -v wpa_supplicant wpa_cli wpa_passphrase` | Resolve all three |
| 11 | `readlink -f "$(command -v wpa_supplicant)"` | Real daemon path |
| 12 | `file "$(command -v wpa_supplicant)"` | Confirm a native ELF |
| 13 | `dpkg -S /usr/sbin/wpa_supplicant` | Owning package is `wpasupplicant` |
| 14 | `dpkg -s wpasupplicant \| grep -E 'Version\|Depends'` | Package provenance |
| 15 | `dpkg -s wpa_supplicant 2>&1 \| head -2` | The other name is not installed |
| 16 | `dpkg -L wpasupplicant \| grep -E 'bin/'` | Shipped executables |
| 17 | `ls -l /etc/wpa_supplicant/` | Shipped helper scripts |
| 18 | `ls -l /etc/wpa_supplicant/wpa_supplicant.conf 2>&1` | Default config may be absent |
| 19 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which package actually owns the binaries?
- Does `-L` exit 0 or print and fail?
- How many commands does `wpa_cli` advertise?
- Where does Debian put the default configuration?

**Sources:** `wpa_supplicant -h`; `wpa_cli -h`; `dpkg -S`; observed output.
**Output:** a provenance record and a command-surface count.

## Scenario 2 — configuration files and driver selection

| # | Command | Purpose |
|---|---------|---------|
| 20 | `printf 'ctrl_interface=DIR=/tmp/wpa-demo/ctrl\n' > minimal.conf` | Smallest valid config |
| 21 | `cat minimal.conf` | Inspect it |
| 22 | `timeout 2 wpa_supplicant -D nl80211 -i nosuch0 -c minimal.conf` | Missing interface |
| 23 | `timeout 2 wpa_supplicant -D bogus -i lo -c minimal.conf` | Unsupported driver |
| 24 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c /nonexistent.conf` | Missing config |
| 25 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c /dev/null` | Empty config is accepted |
| 26 | `timeout 2 wpa_supplicant -D wired -i lo -c minimal.conf` | Wired driver blocks |
| 27 | `timeout 2 wpa_supplicant -D nl80211,wext -i lo -c minimal.conf` | Driver fallback list |
| 28 | `timeout 2 wpa_supplicant -D nl80211 -i lo -C 'DIR=/tmp/wpa-demo/ctrl'` | `-C` instead of `-c` |
| 29 | `timeout 2 wpa_supplicant -c minimal.conf` | No interface |
| 30 | `timeout 2 wpa_supplicant -i lo` | No config |
| 31 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -p bogus` | Bad driver parameter |
| 32 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -b br0` | Missing bridge |
| 33 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -P /tmp/wpa-demo/pid` | PID file |
| 34 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -f /tmp/wpa-demo/log` | Log file |
| 35 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -s` | Syslog output |
| 36 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -t -d` | Timestamps and debug |
| 37 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -K` | Keys in debug output |
| 38 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -W` | Wait for a monitor |
| 39 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c minimal.conf -u` | D-Bus control |

**Research questions**
- Which failures happen before the interface is touched?
- Does `-C` replace `-c` completely?
- What does `-W` wait for?
- Does `-K` change the log even without `-d`?

**Sources:** `wpa_supplicant -h`; observed exit codes and messages.
**Output:** a startup-failure matrix with exit codes.

## Scenario 3 — starting, stopping, and logging the daemon

| # | Command | Purpose |
|---|---------|---------|
| 40 | `sudo wpa_supplicant -B -P /tmp/wpa-demo/wpa.pid -D nl80211 -i wlp2s0 -c minimal.conf` | Background start |
| 41 | `cat /tmp/wpa-demo/wpa.pid` | PID file |
| 42 | `pgrep -af 'wpa_supplicant.*wlp2s0'` | Confirm the process |
| 43 | `sudo wpa_supplicant -B -f /tmp/wpa-demo/wpa.log -D nl80211 -i wlp2s0 -c minimal.conf` | Log to a file |
| 44 | `tail -20 /tmp/wpa-demo/wpa.log` | Startup log |
| 45 | `sudo wpa_supplicant -B -s -D nl80211 -i wlp2s0 -c minimal.conf` | Syslog instead |
| 46 | `journalctl -t wpa_supplicant -n 20 --no-pager` | Read the syslog path |
| 47 | `sudo wpa_supplicant -B -t -d -D nl80211 -i wlp2s0 -c minimal.conf` | Timestamps and debug |
| 48 | `sudo wpa_supplicant -B -q -D nl80211 -i wlp2s0 -c minimal.conf` | Quieter |
| 49 | `sudo wpa_supplicant -B -qq -D nl80211 -i wlp2s0 -c minimal.conf` | Quietest |
| 50 | `sudo wpa_supplicant -g /tmp/wpa-demo/global -D nl80211 -i wlp2s0 -c minimal.conf` | Global control socket |
| 51 | `sudo wpa_supplicant -g /tmp/wpa-demo/global -G netdev -D nl80211 -i wlp2s0 -c minimal.conf` | Group ownership |
| 52 | `sudo wpa_supplicant -g /tmp/wpa-demo/global -N -i wlp2s0 -c minimal.conf -N -i lo -c minimal.conf` | Two interface descriptions |
| 53 | `sudo wpa_supplicant -B -D nl80211 -i wlp2s0 -c minimal.conf -I extra.conf` | Additional config |
| 54 | `sudo wpa_supplicant -B -D nl80211 -i wlp2s0 -c minimal.conf -e /dev/urandom` | Entropy source |
| 55 | `sudo wpa_supplicant -B -D nl80211 -i wlp2s0 -c minimal.conf -T` | Linux tracing |
| 56 | `wpa_cli -i wlp2s0 terminate` | Stop through the control socket |
| 57 | `kill "$(cat /tmp/wpa-demo/wpa.pid)"` | Stop by PID |
| 58 | `pgrep -af wpa_supplicant \| head -5` | Confirm what is still running |

**Research questions**
- Does `-B` return before association completes?
- Which log destination wins when `-f`, `-s`, and stdout are all available?
- Does `-N` require a global control socket?
- Does `terminate` remove the PID file?

**Sources:** `wpa_supplicant -h`; observed output.
**Output:** a start/stop runbook with log destinations.

## Scenario 4 — status and link introspection

| # | Command | Purpose |
|---|---------|---------|
| 59 | `wpa_cli -i wlp2s0 status` | Current association |
| 60 | `wpa_cli -i wlp2s0 status verbose` | Verbose status |
| 61 | `wpa_cli -i wlp2s0 ifname` | Interface name |
| 62 | `wpa_cli -i wlp2s0 ping` | Control-socket liveness |
| 63 | `wpa_cli -i wlp2s0 signal_poll` | RSSI, noise, width |
| 64 | `wpa_cli -i wlp2s0 pktcnt_poll` | TX/RX counters |
| 65 | `wpa_cli -i wlp2s0 mib` | 802.11 MIB variables |
| 66 | `wpa_cli -i wlp2s0 driver_flags` | Driver capability bits |
| 67 | `wpa_cli -i wlp2s0 get country` | Regulatory country |
| 68 | `wpa_cli -i wlp2s0 get_capability modes` | Supported modes |
| 69 | `wpa_cli -i wlp2s0 get_capability eap` | Supported EAP methods |
| 70 | `wpa_cli -i wlp2s0 get_capability pairwise` | Pairwise ciphers |
| 71 | `wpa_cli -i wlp2s0 get_capability key_mgmt` | Key-management methods |
| 72 | `wpa_cli -i wlp2s0 pmksa` | PMKSA cache |
| 73 | `wpa_cli -i wlp2s0 list_creds` | Stored credentials |
| 74 | `wpa_cli -i wlp2s0 log_level` | Current debug level |
| 75 | `wpa_cli -i wlp2s0 raw STATUS` | Unprocessed command path |
| 76 | `wpa_cli -i wlp2s0 help status` | Per-command help |
| 77 | `wpa_cli -i wlp2s0 license` | Client licence |
| 78 | `wpa_cli -i wlp2s0 interface` | Interfaces known to the daemon |
| 79 | `wpa_cli -i wlp2s0 interface p2p-dev-wlp2s0` | Select another interface |
| 80 | `wpa_cli -i wlp2s0 note audit-run` | Write to the debug log |
| 81 | `wpa_cli -i wlp2s0 relog` | Reopen the log file |

**Research questions**
- Which values come from the driver and which from the daemon?
- Does `status` work before association completes?
- Does `get_capability` return `FAIL` or an empty list for an unknown class?
- Does `note` appear in the daemon log?

**Sources:** `wpa_cli -h`; observed output.
**Output:** a status-field reference with example values.

## Scenario 5 — scanning and BSS inspection

| # | Command | Purpose |
|---|---------|---------|
| 82 | `wpa_cli -i wlp2s0 scan` | Request a scan |
| 83 | `sleep 3` | Let the scan finish |
| 84 | `wpa_cli -i wlp2s0 scan_results` | Tabular results |
| 85 | `wpa_cli -i wlp2s0 all_bss` | All BSS entries |
| 86 | `wpa_cli -i wlp2s0 bss 0` | Detail for BSS index 0 |
| 87 | `wpa_cli -i wlp2s0 bss 36:d9:b5:cb:88:80` | Detail by BSSID |
| 88 | `wpa_cli -i wlp2s0 abort_scan` | Abort an in-flight scan |
| 89 | `wpa_cli -i wlp2s0 scan_interval 10` | Set the scan interval |
| 90 | `wpa_cli -i wlp2s0 bss_expire_age 180` | BSS expiry age |
| 91 | `wpa_cli -i wlp2s0 bss_expire_count 2` | BSS expiry count |
| 92 | `wpa_cli -i wlp2s0 bss_flush 0` | Flush BSS entries |
| 93 | `wpa_cli -i wlp2s0 get_capability channels` | Supported channels |
| 94 | `wpa_cli -i wlp2s0 get_capability freq` | Supported frequencies |
| 95 | `wpa_cli -i wlp2s0 signal_monitor 100 10` | Signal monitor parameters |
| 96 | `wpa_cli -i wlp2s0 autoscan periodic:10` | Autoscan policy |
| 97 | `wpa_cli -i wlp2s0 autoscan` | Clear autoscan |
| 98 | `wpa_cli -i wlp2s0 mac_rand_scan scan enable=1` | Scan MAC randomisation |
| 99 | `wpa_cli -i wlp2s0 mac_rand_scan scan enable=0` | Disable it |
| 100 | `wpa_cli -i wlp2s0 neighbor_rep_request` | Neighbour report request |

**Research questions**
- Does `scan` return `OK` before results exist?
- How long does a full scan take on this adapter?
- Does `bss 0` index into `scan_results` or `all_bss`?
- Does `mac_rand_scan` need driver support?

**Sources:** `wpa_cli -h`; observed output.
**Output:** a scan-result schema and a BSS-detail field list.

## Scenario 6 — network, credential, and WPS management

| # | Command | Purpose |
|---|---------|---------|
| 101 | `wpa_cli -i wlp2s0 list_networks` | Configured networks |
| 102 | `wpa_cli -i wlp2s0 add_network` | Add a network block |
| 103 | `wpa_cli -i wlp2s0 set_network 0 ssid '"lab-ap"'` | Set SSID |
| 104 | `wpa_cli -i wlp2s0 set_network 0 psk '"correct horse battery staple"'` | Set PSK |
| 105 | `wpa_cli -i wlp2s0 set_network 0 key_mgmt WPA-PSK` | Set key management |
| 106 | `wpa_cli -i wlp2s0 set_network 0 priority 5` | Set priority |
| 107 | `wpa_cli -i wlp2s0 get_network 0 ssid` | Read a variable |
| 108 | `wpa_cli -i wlp2s0 set_network 0` | List variables |
| 109 | `wpa_cli -i wlp2s0 enable_network 0` | Enable |
| 110 | `wpa_cli -i wlp2s0 disable_network 0` | Disable |
| 111 | `wpa_cli -i wlp2s0 select_network 0` | Select and disable others |
| 112 | `wpa_cli -i wlp2s0 reassociate` | Force reassociation |
| 113 | `wpa_cli -i wlp2s0 reattach` | Reattach to the same BSS |
| 114 | `wpa_cli -i wlp2s0 reconnect` | Reconnect if disconnected |
| 115 | `wpa_cli -i wlp2s0 disconnect` | Disconnect |
| 116 | `wpa_cli -i wlp2s0 save_config` | Persist the config |
| 117 | `wpa_cli -i wlp2s0 remove_network 0` | Remove the block |
| 118 | `wpa_cli -i wlp2s0 add_cred` | Add an EAP credential |
| 119 | `wpa_cli -i wlp2s0 set_cred 0 realm example.org` | Set credential realm |
| 120 | `wpa_cli -i wlp2s0 get_cred 0 realm` | Read it back |
| 121 | `wpa_cli -i wlp2s0 remove_cred 0` | Remove it |
| 122 | `wpa_cli -i wlp2s0 identity 0 alice` | EAP identity |
| 123 | `wpa_cli -i wlp2s0 password 0 secret` | EAP password |
| 124 | `wpa_cli -i wlp2s0 psk_passphrase 0 'correct horse battery staple'` | Set a PSK passphrase |
| 125 | `wpa_cli -i wlp2s0 bssid 0 36:d9:b5:cb:88:80` | Pin a BSSID |
| 126 | `wpa_cli -i wlp2s0 bssid_ignore 36:d9:b5:cb:88:80` | Temporarily ignore a BSS |
| 127 | `wpa_cli -i wlp2s0 bssid_ignore` | List ignored BSSIDs |
| 128 | `wpa_cli -i wlp2s0 wps_pbc` | WPS push-button |
| 129 | `wpa_cli -i wlp2s0 wps_pin 36:d9:b5:cb:88:80` | WPS PIN |
| 130 | `wpa_cli -i wlp2s0 wps_check_pin 12345670` | Check PIN checksum |
| 131 | `wpa_cli -i wlp2s0 wps_cancel` | Cancel WPS |
| 132 | `wpa_cli -i wlp2s0 flush` | Flush state |
| 133 | `wpa_cli -i wlp2s0 reconfigure` | Re-read the config |
| 134 | `wpa_cli -i wlp2s0 terminate` | Stop the daemon |

**Research questions**
- Does `add_network` return the new id on stdout?
- Does `set_network` need the SSID value quoted twice?
- Does `save_config` require `update_config=1`?
- Does `wps_pin` return a generated PIN when none is supplied?

**Sources:** `wpa_cli -h`; observed output.
**Output:** a network-management command sequence and a WPS warning list.

## Scenario 7 — wpa_passphrase and PSK derivation

| # | Command | Purpose |
|---|---------|---------|
| 135 | `wpa_passphrase TestSSID 'correct horse battery staple'` | Full network block |
| 136 | `wpa_passphrase TestSSID 'correct horse battery staple' \| grep ssid` | SSID line |
| 137 | `wpa_passphrase TestSSID 'correct horse battery staple' \| grep '#psk='` | Plaintext comment |
| 138 | `wpa_passphrase TestSSID 'correct horse battery staple' \| grep '^	psk='` | 64-hex PSK |
| 139 | `wpa_passphrase TestSSID 'correct horse battery staple' \| grep -c '^network='` | One block |
| 140 | `printf 'correct horse battery staple\n' \| wpa_passphrase TestSSID` | Passphrase from stdin |
| 141 | `wpa_passphrase TestSSID short` | Length error |
| 142 | `wpa_passphrase TestSSID short >/dev/null 2>&1; echo "short=$?"` | Exit code 1 |
| 143 | `wpa_passphrase TestSSID` | No passphrase, no stdin |
| 144 | `wpa_passphrase` | Missing SSID |
| 145 | `wpa_passphrase TestSSID 'aaaaaaaa'` | Eight-character minimum |
| 146 | `wpa_passphrase TestSSID 'aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa'` | 64 hex chars are treated as a PSK |
| 147 | `wpa_passphrase 'SSID with spaces' 'correct horse battery staple'` | SSID quoting |
| 148 | `wpa_passphrase 'quote"inside' 'correct horse battery staple'` | Quote handling |
| 149 | `wpa_passphrase TestSSID 'correct horse battery staple' \| sed '/#psk=/d'` | Safe block for storage |
| 150 | `wpa_passphrase TestSSID 'correct horse battery staple' \| sha256sum` | Determinism check |
| 151 | `wpa_passphrase TestSSID 'correct horse battery staple' \| grep -o 'psk=[0-9a-f]*' \| wc -c` | PSK field length |

**Research questions**
- Is the PSK deterministic for a given SSID and passphrase?
- Does the plaintext comment appear on stdout or stderr?
- What exactly counts as a 64-hex PSK?
- How are quotes and spaces in the SSID encoded?

**Sources:** `wpa_passphrase`; observed output; IEEE 802.11i PBKDF2 definition.
**Output:** a PSK derivation record and a config-safety checklist.

## Scenario 8 — failure modes, control socket, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 152 | `wpa_cli -p /tmp/wpa-demo/nosuch status` | Missing socket |
| 153 | `wpa_cli -p /tmp/wpa-demo/nosuch status >/dev/null 2>&1; echo "nosock=$?"` | Exit 255 |
| 154 | `wpa_cli -i nosuch0 status` | Unknown interface |
| 155 | `wpa_cli -i wlp2s0 frobnicate` | Unknown command |
| 156 | `wpa_cli -i wlp2s0 get_network 99 ssid` | Unknown network id |
| 157 | `wpa_cli -i wlp2s0 set_network 99 ssid '"x"'` | Unknown network id on write |
| 158 | `wpa_cli -i wlp2s0 remove_network 99` | Remove a missing network |
| 159 | `wpa_cli -i wlp2s0 bss 99` | Unknown BSS index |
| 160 | `wpa_cli -i wlp2s0 wps_check_pin 00000000` | Bad PIN checksum |
| 161 | `wpa_cli -i wlp2s0 get_capability bogus` | Unknown capability class |
| 162 | `timeout 2 wpa_supplicant -D nl80211 -i nosuch0 -c minimal.conf 2>&1 \| tail -3` | Startup error text |
| 163 | `timeout 2 wpa_supplicant -D bogus -i lo -c minimal.conf 2>&1 \| grep -i unsupported` | Driver error |
| 164 | `timeout 2 wpa_supplicant -D nl80211 -i lo -c /nonexistent.conf 2>&1 \| grep -i 'failed'` | Config error |
| 165 | `ls -l /tmp/wpa-demo/ctrl 2>&1` | Control socket permissions |
| 166 | `stat -c '%a %U %G %n' /tmp/wpa-demo/ctrl 2>&1` | Numeric mode |
| 167 | `ss -xl \| grep -i wpa \| head -3` | UNIX sockets |
| 168 | `pgrep -af wpa_supplicant \| head -5` | Surviving processes |
| 169 | `sudo pkill -f 'wpa_supplicant.*wlp2s0'` | Stop leftovers |
| 170 | `rm -rf ~/wpa-demo` | Remove the demo tree |
| 171 | `test ! -e ~/wpa-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which errors are daemon-side and which are client-side?
- Does `wpa_cli` distinguish "command failed" from "socket failed" in `$?`?
- What mode does the control socket get by default?
- Does killing the daemon leave a stale socket behind?

**Sources:** `wpa_cli -h`; `wpa_supplicant -h`; observed output.
**Output:** an error matrix, a socket-permission record, and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-v`, `-L`, `-h`, binaries, package `wpasupplicant`, config directory | 1 |
| `-D`, `-i`, `-c`, `-C`, `-p`, `-b`, `-P`, `-f`, `-s`, `-t`, `-d`, `-K`, `-W`, `-u` | 2 |
| `-B`, `-g`, `-G`, `-N`, `-I`, `-e`, `-T`, `-q`, `-qq`, `terminate`, PID handling | 3 |
| `status`, `status verbose`, `ifname`, `ping`, `signal_poll`, `pktcnt_poll`, `mib`, `driver_flags`, `get`, `get_capability`, `pmksa`, `list_creds`, `log_level`, `raw`, `help`, `license`, `interface`, `note`, `relog` | 4 |
| `scan`, `scan_results`, `all_bss`, `bss`, `abort_scan`, `scan_interval`, `bss_expire_*`, `bss_flush`, `signal_monitor`, `autoscan`, `mac_rand_scan`, `neighbor_rep_request` | 5 |
| `list_networks`, `add_network`, `set_network`, `get_network`, `enable_network`, `disable_network`, `select_network`, `reassociate`, `reattach`, `reconnect`, `disconnect`, `save_config`, `remove_network`, `*_cred`, `identity`, `password`, `psk_passphrase`, `bssid`, `bssid_ignore`, `wps_*`, `flush`, `reconfigure`, `terminate` | 6 |
| `wpa_passphrase` block, stdin, length errors, 64-hex PSK, quoting, determinism | 7 |
| bad socket, bad interface, bad ids, bad PIN, startup errors, socket mode, cleanup | 8 |

Full text: [`help.txt`](help.txt), [`help-wpa_cli.txt`](help-wpa_cli.txt),
[`help-wpa_passphrase.txt`](help-wpa_passphrase.txt); authoritative
configuration reference in the upstream `wpa_supplicant.conf` manual.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, licence, help, binaries, package provenance | 19 |
| 2 | Configuration files and driver selection | 20 |
| 3 | Starting, stopping, and logging the daemon | 19 |
| 4 | Status and link introspection | 23 |
| 5 | Scanning and BSS inspection | 19 |
| 6 | Network, credential, and WPS management | 34 |
| 7 | wpa_passphrase and PSK derivation | 17 |
| 8 | Failure modes, control socket, and cleanup | 20 |
| **Total** | | **171** |

## Legal and safety

`wpa_supplicant` changes radio and interface state, can disconnect the host you
are working on, and can join networks you do not control. Only associate with
networks and adapters you own or are explicitly authorised to test. Scanning
and association are active radio operations; respect spectrum rules and
neighbour privacy. `wps_pbc`, `wps_pin`, and `wps_reg` interact with an AP's
WPS implementation and can alter its configuration — never run them against an
AP you do not own. `-K` writes passwords and keys to logs; keep those logs
protected and delete them after the run. The control socket is a local
privilege boundary: a group-writable socket lets any member drive the daemon,
read the SSID and BSSID, and change networks. Delete the demo tree and any
captured credentials when finished.
