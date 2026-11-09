# macchanger — command scenarios for deep research

Installed build: GNU MAC Changer 1.7.0 (Debian package `macchanger`
`1.7.0-6`). Every option below appears in [`help.txt`](help.txt). Behaviour
notes come from `man macchanger` and from observing `ip link` before and
after each change.

> **Scope:** Changing a MAC address can break connectivity, bypass a network's
> MAC filter, and reset the interface's driver state. Use it only on
> interfaces and networks you are authorized to test, and always record the
> permanent address so you can restore it.

## How to invoke

```text
@deep-research For an authorized wireless assessment, compare macchanger
1.7.0 with the MAC randomisation built into wifite 2.7.0, bettercap 2.33.0,
and NetworkManager. How do they differ in when the change is applied, which
address bits are set, whether the change survives a driver reset, and how
they interact with DHCP client identifiers? Cite the macchanger manual, the
IEEE 802 address format, systemd/NetworkManager documentation, and the
bettercap mac.changer module docs; return a comparison with confidence notes.
```

Create an evidence workspace and record the permanent address:

```bash
mkdir -p ~/macchanger-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/macchanger-demo/evidence/run-start.txt
ip -brief link > ~/macchanger-demo/evidence/links-before.txt
macchanger -s eth0 > ~/macchanger-demo/evidence/mac-before.txt 2>&1
```

## Scenario 1 — inspect before changing

Read the permanent address and the vendor database before touching anything.
The permanent address is the only reliable way back.

| # | Command | How to use |
|---|---------|------------|
| 1 | `macchanger --help` | All options |
| 2 | `macchanger --version` | Version banner |
| 3 | `macchanger -s eth0` | Current and permanent MAC |
| 4 | `ip link show eth0` | Kernel view of the address |
| 5 | `macchanger -l \| head` | Vendor database preview |
| 6 | `macchanger -l intel` | Search vendors by keyword |
| 7 | `macchanger -l \| wc -l` | Size of the vendor database |
| 8 | `man macchanger \| col -b \| sed -n '1,80p'` | Full manual |

**Research questions**
- What does `-s` print that `ip link` does not?
- Where does the vendor database come from, and how current is it?
- Which byte in a MAC identifies the vendor?
- What do the locally-administered and multicast bits mean?
- Why is the permanent address the only reliable restore point?

**Sources:** `macchanger --help`; `man macchanger`; IEEE 802 address format;
the OUI database.
**Output:** a before-change record with the permanent address.

## Scenario 2 — explicit and random changes

Explicit changes are reproducible; random changes are not. Pick based on
whether the test needs to be repeatable.

| # | Command | How to use |
|---|---------|------------|
| 9 | `sudo ip link set eth0 down` | Take the interface down |
| 10 | `sudo macchanger -m 02:00:00:00:00:01 eth0` | Set an explicit address |
| 11 | `sudo ip link set eth0 up` | Bring it back up |
| 12 | `ip link show eth0` | Verify the change |
| 13 | `sudo ip link set eth0 down && sudo macchanger -r eth0 && sudo ip link set eth0 up` | Fully random |
| 14 | `sudo ip link set eth0 down && sudo macchanger -a eth0 && sudo ip link set eth0 up` | Same-vendor random |
| 15 | `sudo ip link set eth0 down && sudo macchanger -A eth0 && sudo ip link set eth0 up` | Any-vendor random |
| 16 | `sudo ip link set eth0 down && sudo macchanger -e eth0 && sudo ip link set eth0 up` | Change only the ending |
| 17 | `sudo ip link set eth0 down && sudo macchanger -b -r eth0 && sudo ip link set eth0 up` | Present as burned-in |

**Research questions**
- Does this driver accept a MAC change while the interface is up?
- What is the difference between `-r`, `-a`, and `-A` in the resulting OUI?
- Which bits does `-r` set, and how does `-b` change them?
- How many bytes does `-e` actually change?
- What happens if the new address collides with another device?

**Sources:** `macchanger --help`; `man macchanger`; IEEE 802; driver
documentation for the interface in use.
**Output:** a change log with the exact address used and the verification
command.

## Scenario 3 — wireless interfaces and monitor mode

Wireless is where MAC changes matter most, and where they are most likely to
reset the interface state.

| # | Command | How to use |
|---|---------|------------|
| 18 | `iw dev` | List wireless interfaces |
| 19 | `sudo ip link set wlan0 down` | Take the radio down |
| 20 | `sudo macchanger -r wlan0` | Randomise the station MAC |
| 21 | `sudo ip link set wlan0 up` | Bring it back up |
| 22 | `iw dev wlan0 link` | Confirm association state |
| 23 | `sudo airmon-ng start wlan0` | Create a monitor interface |
| 24 | `iw dev wlan0mon info` | Verify monitor mode and channel |
| 25 | `sudo ip link set wlan0mon down && sudo macchanger -m 02:00:00:00:00:02 wlan0mon && sudo ip link set wlan0mon up` | Change the monitor MAC |
| 26 | `iw dev wlan0mon info` | Check whether channel/type survived |

**Research questions**
- Does the driver preserve monitor mode across a MAC change?
- What happens to the channel after the interface is brought down and up?
- Does the AP see the new MAC immediately, or only after reassociation?
- How does `wifite -mac` differ from a manual change?
- What does `iw dev ... link` show after a change?

**Sources:** `iw` documentation; `macchanger --help`; `airmon-ng` usage;
IEEE 802.11 association.
**Output:** a wireless change record with monitor-mode state before and after.

## Scenario 4 — restore and verify

Cleanup is the part that gets skipped. Make it explicit and verifiable.

| # | Command | How to use |
|---|---------|------------|
| 27 | `macchanger -s eth0` | Show current and permanent |
| 28 | `sudo ip link set eth0 down` | Take it down for restore |
| 29 | `sudo macchanger -p eth0` | Restore the permanent address |
| 30 | `sudo ip link set eth0 up` | Bring it back up |
| 31 | `ip link show eth0` | Verify the restore |
| 32 | `macchanger -s eth0` | Confirm current equals permanent |
| 33 | `sudo systemctl restart NetworkManager` | Rebuild the connection |
| 34 | `ip -brief address` | Confirm addressing |

**Research questions**
- Does `-p` always succeed, or can a driver reject it?
- What breaks if you restore the MAC while the interface is up?
- Does DHCP need to be restarted after a restore?
- How do you prove the permanent address is back?
- What should the evidence file contain?

**Sources:** `macchanger --help`; `man macchanger`; `ip link`; NetworkManager
documentation.
**Output:** a restore record with before/after addresses and a connectivity
check.

## Scenario 5 — interaction with other tools

`macchanger` is often used alongside wireless and MITM tools. Know which one
owns the interface.

| # | Command | How to use |
|---|---------|------------|
| 35 | `sudo wifite -mac --daemon` | Wifite randomises and restores managed mode |
| 36 | `sudo bettercap -iface eth0 -eval 'set mac.changer.address 02:00:00:00:00:03; mac.changer on; mac.changer off; quit'` | Bettercap module change and restore |
| 37 | `nmcli device show eth0 \| grep -i mac` | NetworkManager's view |
| 38 | `ip link show eth0` | Kernel's view |
| 39 | `sudo ethtool -P eth0` | Permanent address from the driver |
| 40 | `journalctl -u NetworkManager --since '10 min ago' \| grep -i mac` | Check for MAC-related errors |

**Research questions**
- Which tool owns the interface during a change?
- Does NetworkManager overwrite a manual change?
- What does `ethtool -P` report that `macchanger -s` does not?
- How do you detect a driver that silently ignored the change?
- Which logs prove what happened?

**Sources:** `wifite -h`; bettercap `mac.changer` docs; `nmcli`; `ethtool`;
`journalctl`.
**Output:** a tool-interaction matrix and a log excerpt.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-h`, `--help` | 1 |
| `-V`, `--version` | 1 |
| `-s`, `--show` | 1, 4 |
| `-e`, `--ending` | 2 |
| `-a`, `--another` | 2 |
| `-A` | 2 |
| `-p`, `--permanent` | 4 |
| `-r`, `--random` | 2, 3 |
| `-l`, `--list` | 1 |
| `-b`, `--bia` | 2 |
| `-m`, `--mac` | 2, 3 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inspect before changing | 8 |
| 2 | Explicit and random changes | 9 |
| 3 | Wireless and monitor mode | 9 |
| 4 | Restore and verify | 8 |
| 5 | Interaction with other tools | 6 |
| **Total** | | **40** |

## Legal and safety

A MAC change can bypass a network's access control and can break the
interface's driver state. Use it only on interfaces and networks you are
authorized to test. Record the permanent address before the first change,
restore it with `-p` before leaving, and verify with both `ip link` and
`macchanger -s`. On wireless interfaces, confirm that monitor mode and channel
survived the change, and restart NetworkManager if connectivity does not
return.
