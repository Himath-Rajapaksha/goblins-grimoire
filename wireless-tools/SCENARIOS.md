# wireless-tools — command scenarios for deep research

Installed build: wireless-tools 30~pre9-18+b1, with `iwconfig`, `iwlist`,
`iwpriv`, `iwspy`, `iwgetid`, and `iwevent`. Every option below appears in the
per-tool help snapshots; behaviour was observed on this host against `wlp2s0`
and non-wireless interfaces.

> **Scope:** read-only queries are safe, but `iwconfig` writes, `iwlist
> scanning`, and `iwevent` change or use the radio. Only run them on adapters
> and networks you own or are explicitly authorised to test. Use a spare
> interface for the set-parameter scenarios.

> **WEXT is legacy.** On a modern nl80211 driver many operations still work,
> but features such as SAE, PMF, HE/EHT, and multi-link are invisible. Use
> [`iw`](../iw/) for configuration where possible; these scenarios document
> what the legacy path exposes.

## How to invoke

```text
@deep-research For an authorised Wi-Fi assessment with wireless-tools 30 on a
modern nl80211 host, explain which Wireless Extensions operations still work,
how `iwconfig`, `iwlist`, `iwpriv`, `iwspy`, `iwgetid`, and `iwevent` map to
their nl80211/`iw` equivalents, which queries report "not supported" with exit
0, and how scan output is structured. Show which writes change interface state,
how `iwgetid -r` differs from its labelled modes, and why `iwspy` and `iwpriv`
are driver-dependent. Cite the installed help text, the kernel WEXT
documentation, and observed output. Separate confirmed behaviour from inference
and state confidence.
```

Create a working directory and record the environment:

```bash
mkdir -p ~/wext-demo/evidence
cd ~/wext-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
iwconfig --version > evidence/version.txt 2>&1
ip -brief link > evidence/links.txt
```

## Scenario 1 — version, help, binaries, package provenance

| # | Command | Purpose |
|---|---------|---------|
| 1 | `iwconfig --version` | Suite version |
| 2 | `iwconfig` | Usage and interface list |
| 3 | `iwlist` | Usage |
| 4 | `iwpriv` | Usage |
| 5 | `iwspy` | Usage |
| 6 | `iwgetid -h` | Options |
| 7 | `iwevent -h` | Options |
| 8 | `iwevent -v` | Version |
| 9 | `command -v iwconfig iwlist iwpriv iwspy iwgetid iwevent` | Resolve all six |
| 10 | `readlink -f "$(command -v iwconfig)"` | Real path |
| 11 | `file "$(command -v iwconfig)"` | Confirm a native ELF |
| 12 | `dpkg -S /sbin/iwconfig` | Owning package |
| 13 | `dpkg -s wireless-tools \| grep -E 'Version\|Depends'` | Package provenance |
| 14 | `dpkg -L wireless-tools \| grep -E 'bin/'` | Shipped executables |
| 15 | `iwgetid --version 2>&1 \| head -2` | Not a supported flag |
| 16 | `man iwconfig \| col -b \| head -20` | Manual page |
| 17 | `ls -l /usr/share/doc/wireless-tools/` | Shipped documentation |
| 18 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which package owns all six binaries?
- Does every tool share one version string?
- Does `iwgetid` accept `--version`?
- What documentation ships with the package?

**Sources:** per-tool help; `dpkg -S`; observed output.
**Output:** a provenance record and a tool inventory.

## Scenario 2 — interface inventory and WEXT state

| # | Command | Purpose |
|---|---------|---------|
| 19 | `iwconfig` | Every interface and WEXT presence |
| 20 | `iwconfig 2>&1 \| grep -c 'no wireless extensions'` | Count non-Wi-Fi interfaces |
| 21 | `iwconfig 2>&1 \| grep -v 'no wireless extensions'` | Wireless interfaces only |
| 22 | `iwconfig wlp2s0` | Full WEXT link state |
| 23 | `iwconfig lo` | Loopback has no WEXT |
| 24 | `iwconfig tun0` | Tunnel has no WEXT |
| 25 | `iwconfig nosuch0 2>&1` | Missing interface |
| 26 | `iwconfig wlp2s0 \| grep ESSID` | SSID line |
| 27 | `iwconfig wlp2s0 \| grep 'Access Point'` | BSSID line |
| 28 | `iwconfig wlp2s0 \| grep Frequency` | Frequency and channel |
| 29 | `iwconfig wlp2s0 \| grep 'Tx-Power'` | TX power |
| 30 | `iwconfig wlp2s0 \| grep 'Link Quality'` | Quality and signal |
| 31 | `iwconfig wlp2s0 \| grep 'Retry'` | Retry, RTS, fragment |
| 32 | `iwconfig wlp2s0 \| grep Encryption` | Encryption state |
| 33 | `iwconfig wlp2s0 \| grep 'Power Management'` | Power management |
| 34 | `iwconfig wlp2s0 \| grep 'invalid'` | Error counters |
| 35 | `iw dev` | nl80211 view for comparison |
| 36 | `ip -brief link` | Kernel link view |
| 37 | `iwconfig wlp2s0 > evidence/iwconfig-wlp2s0.txt` | Save the state |
| 38 | `diff <(iwconfig wlp2s0) evidence/iwconfig-wlp2s0.txt` | Stability check |

**Research questions**
- Which fields does WEXT expose that `iw` does not, and vice versa?
- Does `iwconfig` exit non-zero for a non-wireless interface?
- Which counters are driver-maintained?
- Does the output change between two reads?

**Sources:** `iwconfig`; `iw dev`; observed output.
**Output:** a WEXT field map with a saved baseline.

## Scenario 3 — iwgetid script fields

| # | Command | Purpose |
|---|---------|---------|
| 39 | `iwgetid` | Default interface ESSID |
| 40 | `iwgetid wlp2s0` | Explicit interface |
| 41 | `iwgetid -r wlp2s0` | Raw ESSID |
| 42 | `iwgetid -s wlp2s0` | Scheme identifier |
| 43 | `iwgetid -a wlp2s0` | Access-point address |
| 44 | `iwgetid -c wlp2s0` | Channel |
| 45 | `iwgetid -f wlp2s0` | Frequency |
| 46 | `iwgetid -m wlp2s0` | Mode |
| 47 | `iwgetid -p wlp2s0` | Protocol name |
| 48 | `iwgetid --ap wlp2s0` | Long form |
| 49 | `iwgetid --channel wlp2s0` | Long form |
| 50 | `iwgetid --freq wlp2s0` | Long form |
| 51 | `iwgetid --mode wlp2s0` | Long form |
| 52 | `iwgetid --protocol wlp2s0` | Long form |
| 53 | `iwgetid --raw wlp2s0` | Long form |
| 54 | `iwgetid lo` | No WEXT |
| 55 | `iwgetid nosuch0` | Missing interface |
| 56 | `iwgetid -r wlp2s0 \| wc -c` | Raw value length |
| 57 | `iwgetid -a wlp2s0 \| awk '{print $NF}'` | Parse the address |
| 58 | `iwgetid -r wlp2s0 > evidence/essid.txt` | Save for comparison |

**Research questions**
- Which options print a label and which print only the value?
- Does `-r` differ from `-s` on this driver?
- What exit code does a non-wireless interface produce?
- Is the default interface chosen the same way as `iwconfig`?

**Sources:** `iwgetid -h`; observed output.
**Output:** a field/format table and an exit-code record.

## Scenario 4 — iwlist capability queries

| # | Command | Purpose |
|---|---------|---------|
| 59 | `iwlist wlp2s0 frequency` | Supported channels and frequencies |
| 60 | `iwlist wlp2s0 channel` | Current channel |
| 61 | `iwlist wlp2s0 bitrate` | Supported bit rates |
| 62 | `iwlist wlp2s0 rate` | Current rate |
| 63 | `iwlist wlp2s0 encryption` | Encryption capability |
| 64 | `iwlist wlp2s0 keys` | Key state |
| 65 | `iwlist wlp2s0 wpakeys` | WPA key state |
| 66 | `iwlist wlp2s0 power` | Power management |
| 67 | `iwlist wlp2s0 txpower` | TX power |
| 68 | `iwlist wlp2s0 retry` | Retry parameters |
| 69 | `iwlist wlp2s0 ap` | Peer/AP table |
| 70 | `iwlist wlp2s0 accesspoints` | Peer/AP table alias |
| 71 | `iwlist wlp2s0 peers` | Peer table |
| 72 | `iwlist wlp2s0 event` | Event capability |
| 73 | `iwlist wlp2s0 auth` | Authentication parameters |
| 74 | `iwlist wlp2s0 genie` | Raw information elements |
| 75 | `iwlist wlp2s0 modulation` | Modulation capabilities |
| 76 | `iwlist wlp2s0 frequency \| head -8` | First channels |
| 77 | `iwlist wlp2s0 frequency \| grep -c 'Channel'` | Channel count |
| 78 | `iwlist wlp2s0 frequency > evidence/freq.txt` | Save the list |
| 79 | `iwlist lo scanning` | Non-wireless scan attempt |
| 80 | `iwlist` | No arguments exits 1 |

**Research questions**
- Which queries return data and which return "not supported"?
- Does `genie` expose raw IEs that `iw` also shows?
- Is the channel list regulatory-domain dependent?
- Does `iwlist` validate the query name?

**Sources:** `iwlist` help; observed output.
**Output:** a query/capability matrix with exit codes.

## Scenario 5 — scanning and BSS parsing

| # | Command | Purpose |
|---|---------|---------|
| 81 | `sudo iwlist wlp2s0 scanning` | Full scan |
| 82 | `sudo iwlist wlp2s0 scanning \| grep -c 'Cell '` | BSS count |
| 83 | `sudo iwlist wlp2s0 scanning \| grep ESSID` | SSIDs |
| 84 | `sudo iwlist wlp2s0 scanning \| grep -E 'Cell\|ESSID'` | BSSID and SSID pairs |
| 85 | `sudo iwlist wlp2s0 scanning \| grep 'Quality'` | Signal quality |
| 86 | `sudo iwlist wlp2s0 scanning \| grep 'Encryption key'` | Encryption state |
| 87 | `sudo iwlist wlp2s0 scanning \| grep 'IE:'` | Information elements |
| 88 | `sudo iwlist wlp2s0 scanning essid a99` | Filter by ESSID |
| 89 | `sudo iwlist wlp2s0 scanning last` | Reuse previous results |
| 90 | `sudo iwlist wlp2s0 scanning > evidence/scan.txt` | Save the scan |
| 91 | `wc -l evidence/scan.txt` | Scan size |
| 92 | `awk '/Cell /{bssid=$5} /ESSID/{print bssid, $0}' evidence/scan.txt \| head` | Parse pairs |
| 93 | `grep -c 'WPA2' evidence/scan.txt` | WPA2 networks |
| 94 | `grep -c 'WPA3' evidence/scan.txt` | WPA3 networks |
| 95 | `grep -c 'WEP' evidence/scan.txt` | WEP networks |
| 96 | `time sudo iwlist wlp2s0 scanning >/dev/null` | Scan duration |
| 97 | `iw dev wlp2s0 scan \| grep -c '^BSS'` | Compare with `iw` |
| 98 | `sudo iwlist wlp2s0 scanning \| grep -i 'hidden'` | Hidden SSID markers |
| 99 | `sudo iwlist wlp2s0 scanning \| tail -5` | End of output |

**Research questions**
- How long does a full scan take, and does it block?
- Which fields are stable enough to parse?
- Does `essid` filtering work on this driver?
- Does `scanning last` return the previous scan or rescan?

**Sources:** `iwlist` help; observed output; `iw` comparison.
**Output:** a scan schema and a parser recipe.

## Scenario 6 — iwconfig parameters and writes

Run the writes only on a spare interface you control.

| # | Command | Purpose |
|---|---------|---------|
| 100 | `sudo iwconfig wlp2s0 essid lab-ap` | Set ESSID |
| 101 | `sudo iwconfig wlp2s0 essid any` | Clear ESSID preference |
| 102 | `sudo iwconfig wlp2s0 mode managed` | Station mode |
| 103 | `sudo iwconfig wlp2s0 mode ad-hoc` | Ad-hoc mode |
| 104 | `sudo iwconfig wlp2s0 mode monitor` | Monitor mode |
| 105 | `sudo iwconfig wlp2s0 channel 6` | Set channel |
| 106 | `sudo iwconfig wlp2s0 freq 2.437G` | Set frequency |
| 107 | `sudo iwconfig wlp2s0 rate 54M` | Fixed rate |
| 108 | `sudo iwconfig wlp2s0 rate auto` | Automatic rate |
| 109 | `sudo iwconfig wlp2s0 txpower 15dBm` | TX power |
| 110 | `sudo iwconfig wlp2s0 txpower auto` | Automatic TX power |
| 111 | `sudo iwconfig wlp2s0 power off` | Disable power saving |
| 112 | `sudo iwconfig wlp2s0 power period 2` | Power period |
| 113 | `sudo iwconfig wlp2s0 retry limit 4` | Retry limit |
| 114 | `sudo iwconfig wlp2s0 rts 512` | RTS threshold |
| 115 | `sudo iwconfig wlp2s0 frag 1024` | Fragment threshold |
| 116 | `sudo iwconfig wlp2s0 ap 36:D9:B5:CB:88:80` | Lock to a BSSID |
| 117 | `sudo iwconfig wlp2s0 ap auto` | Unlock |
| 118 | `sudo iwconfig wlp2s0 key off` | Disable WEP key |
| 119 | `sudo iwconfig wlp2s0 enc off` | Disable encryption |
| 120 | `sudo iwconfig wlp2s0 commit` | Apply pending changes |
| 121 | `iwconfig wlp2s0 \| head -4` | Verify the result |
| 122 | `sudo iw dev wlp2s0 set type managed` | Restore with nl80211 |
| 123 | `iwconfig nosuch0 essid lab` | Bad interface on write |
| 124 | `iwconfig wlp2s0 mode bogus` | Invalid mode |

**Research questions**
- Which writes are accepted but silently ignored by the driver?
- Does `commit` change behaviour versus immediate writes?
- Does `mode monitor` succeed on this adapter?
- How do you restore a working station after a bad write?

**Sources:** `iwconfig` help; observed output.
**Output:** a write matrix with driver-support notes and a rollback recipe.

## Scenario 7 — iwpriv, iwspy, and iwevent

| # | Command | Purpose |
|---|---------|---------|
| 125 | `iwpriv wlp2s0` | List private ioctls |
| 126 | `iwpriv lo` | Non-wireless interface |
| 127 | `iwpriv wlp2s0 get_foo` | Unknown private command |
| 128 | `iwpriv wlp2s0 set_foo 1` | Unknown private set |
| 129 | `iwspy wlp2s0` | List monitored addresses |
| 130 | `iwspy wlp2s0 + 36:D9:B5:CB:88:80` | Add an address |
| 131 | `iwspy wlp2s0 - 36:D9:B5:CB:88:80` | Remove an address |
| 132 | `iwspy lo` | Unsupported interface |
| 133 | `iwspy nosuch0` | Missing interface |
| 134 | `timeout 3 iwevent` | Watch events briefly |
| 135 | `timeout 3 iwevent > evidence/events.txt 2>&1` | Save events |
| 136 | `wc -l evidence/events.txt` | Event count |
| 137 | `timeout 3 iwevent \| head -5` | First events |
| 138 | `iwevent -v` | Version |
| 139 | `iwevent --help` | Help |
| 140 | `iwevent bogus` | Unexpected argument |
| 141 | `dmesg \| tail -20` | Driver-side messages |
| 142 | `pgrep -af iwevent` | No leftover process |

**Research questions**
- Which private ioctls does this driver expose?
- Does `iwspy` work at all on this adapter?
- What events fire during association or scan?
- Does `iwevent` need root?

**Sources:** per-tool help; observed output.
**Output:** a private-ioctl inventory, an event sample, and a cleanup check.

## Scenario 8 — failure modes, comparisons, and cleanup

| # | Command | Purpose |
|---|---------|---------|
| 143 | `iwconfig nosuch0 2>&1 \| head -3` | Missing interface text |
| 144 | `iwlist nosuch0 scanning 2>&1 \| head -3` | Missing interface on scan |
| 145 | `iwlist wlp2s0 bogus 2>&1 \| head -3` | Unknown query |
| 146 | `iwgetid nosuch0 2>&1; echo "exit=$?"` | Missing interface exit |
| 147 | `iwgetid lo 2>&1; echo "exit=$?"` | Non-wireless exit |
| 148 | `iwpriv nosuch0 2>&1 \| head -3` | Missing interface |
| 149 | `iwspy nosuch0 2>&1 \| head -3` | Missing interface |
| 150 | `iwevent bogus 2>&1 \| head -3` | Unexpected argument |
| 151 | `diff <(iwconfig wlp2s0) <(iw dev wlp2s0 link) \| head -10` | WEXT versus nl80211 |
| 152 | `iwconfig wlp2s0 \| grep -c 'no wireless'` | Sanity check |
| 153 | `iwlist wlp2s0 frequency \| tail -3` | End of channel list |
| 154 | `stat -c '%a %U %G %n' "$(command -v iwconfig)"` | Binary permissions |
| 155 | `ls -l /sbin/iwconfig` | Symlink or real file |
| 156 | `dmesg \| grep -i 'wext\|wireless' \| tail -5` | Kernel WEXT messages |
| 157 | `pgrep -af 'iwlist\|iwevent'` | No leftovers |
| 158 | `rm -rf ~/wext-demo` | Remove the demo tree |
| 159 | `test ! -e ~/wext-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which failures print usage and which print a specific error?
- Which tools return 0 for "not supported"?
- Where does the WEXT view disagree with nl80211?
- Which artefacts does a scan leave behind?

**Sources:** observed output; `iw` comparison.
**Output:** an error matrix, a comparison table, and a clean workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `--version`, per-tool help, binaries, package provenance | 1 |
| `iwconfig` inventory, interface state, `iw` comparison | 2 |
| `iwgetid` `-a`, `-c`, `-f`, `-m`, `-p`, `-r`, `-s`, long forms | 3 |
| `iwlist` frequency, channel, bitrate, rate, encryption, keys, wpakeys, power, txpower, retry, ap, accesspoints, peers, event, auth, genie, modulation | 4 |
| `iwlist scanning`, `essid`, `last`, parsing, `iw` comparison | 5 |
| `iwconfig` essid, mode, channel, freq, rate, txpower, power, retry, rts, frag, ap, key, enc, commit | 6 |
| `iwpriv`, `iwspy`, `iwevent` | 7 |
| error paths, WEXT/nl80211 diff, permissions, cleanup | 8 |

Full text: [`help-iwconfig.txt`](help-iwconfig.txt),
[`help-iwlist.txt`](help-iwlist.txt), [`help-iwpriv.txt`](help-iwpriv.txt),
[`help-iwspy.txt`](help-iwspy.txt), [`help-iwgetid.txt`](help-iwgetid.txt),
[`help-iwevent.txt`](help-iwevent.txt); authoritative description in the
kernel Wireless Extensions documentation and `man iwconfig`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, help, binaries, package provenance | 18 |
| 2 | Interface inventory and WEXT state | 20 |
| 3 | iwgetid script fields | 20 |
| 4 | iwlist capability queries | 22 |
| 5 | Scanning and BSS parsing | 19 |
| 6 | iwconfig parameters and writes | 25 |
| 7 | iwpriv, iwspy, and iwevent | 18 |
| 8 | Failure modes, comparisons, and cleanup | 17 |
| **Total** | | **159** |

## Legal and safety

Wireless Extensions can change interface mode, channel, TX power, and
encryption state. A bad write can disconnect the host or leave the adapter in
monitor mode. Only change parameters on adapters and networks you own or are
explicitly authorised to test. Scanning is an active radio operation; respect
spectrum rules and neighbour privacy. `iwconfig key`/`enc` configure WEP, which
is cryptographically broken — document it as a finding, do not deploy it.
`iwpriv` writes driver-private settings whose effects are undocumented; avoid
them outside a lab. Delete the demo tree and any captured scan data when
finished.
