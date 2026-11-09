# macchanger — inspect and change a network interface MAC address

- **Version:** GNU MAC Changer 1.7.0 (Debian package `macchanger` `1.7.0-6`)
- **Binary:** `/usr/bin/macchanger`
- **Help snapshot:** [`help.txt`](help.txt) — `macchanger --help` (17 lines)
- **Upstream:** https://github.com/alobbs/macchanger
- **Docs:** `man macchanger`, `info macchanger`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`macchanger` reads and writes the hardware address of a network interface. It
can set an arbitrary address, generate a random one, generate a random address
that still looks like the original vendor, keep the vendor prefix and change
only the last three bytes, or restore the permanent address.

Changing a MAC is a normal part of wireless testing (defeating client
tracking, joining a network that filters by MAC, reproducing a client's view),
but it is also trivially detectable and can break drivers. `macchanger` does
not change the interface state; you must bring the interface down and back up
around the change, or use `-m` on an interface that the driver allows to change
while up.

```text
macchanger [options] device
```

## Options

| Option | Meaning |
|--------|---------|
| `-h, --help` | Print help |
| `-V, --version` | Print version and exit |
| `-s, --show` | Print the MAC address and exit |
| `-e, --ending` | Do not change the vendor bytes |
| `-a, --another` | Set a random vendor MAC of the same kind |
| `-A` | Set a random vendor MAC of any kind |
| `-p, --permanent` | Reset to the original, permanent hardware MAC |
| `-r, --random` | Set a fully random MAC |
| `-l, --list[=keyword]` | Print known vendors |
| `-b, --bia` | Pretend to be a burned-in address |
| `-m, --mac=XX:XX:XX:XX:XX:XX` | Set the MAC explicitly |

## How the modes differ

| Mode | Flag | Vendor bytes | NIC bytes | Notes |
|------|------|--------------|-----------|-------|
| Show | `-s` | — | — | Read-only |
| Explicit | `-m` | as given | as given | Full control |
| Random | `-r` | random | random | May set the locally-administered bit |
| Same vendor | `-a` | preserved | random | Looks like the same vendor |
| Any vendor | `-A` | random known vendor | random | Looks like a different vendor |
| Ending only | `-e` | preserved | random ending | Keeps the vendor prefix |
| Permanent | `-p` | original | original | Restores the burned-in address |
| BIA | `-b` | — | — | Marks the address as burned-in |

The `-b`/`--bia` flag is used together with a change so the new address is
presented as a burned-in address rather than a locally administered one. That
matters because many drivers and APs treat the locally-administered bit
differently.

## Recipes

**1. Show the current and permanent MAC.**

```bash
macchanger -s eth0
```

**2. List known vendors.**

```bash
macchanger -l
```

**3. Search vendors by keyword.**

```bash
macchanger -l intel
```

**4. Set an explicit MAC.**

```bash
sudo ip link set eth0 down
sudo macchanger -m 02:00:00:00:00:01 eth0
sudo ip link set eth0 up
```

**5. Set a fully random MAC.**

```bash
sudo ip link set eth0 down
sudo macchanger -r eth0
sudo ip link set eth0 up
```

**6. Random MAC from the same vendor.**

```bash
sudo ip link set eth0 down
sudo macchanger -a eth0
sudo ip link set eth0 up
```

**7. Random MAC from any known vendor.**

```bash
sudo ip link set eth0 down
sudo macchanger -A eth0
sudo ip link set eth0 up
```

**8. Change only the ending.**

```bash
sudo ip link set eth0 down
sudo macchanger -e eth0
sudo ip link set eth0 up
```

**9. Present the new address as burned-in.**

```bash
sudo ip link set eth0 down
sudo macchanger -b -r eth0
sudo ip link set eth0 up
```

**10. Restore the permanent MAC.**

```bash
sudo ip link set eth0 down
sudo macchanger -p eth0
sudo ip link set eth0 up
```

**11. Change a wireless interface.**

```bash
sudo ip link set wlan0 down
sudo macchanger -r wlan0
sudo ip link set wlan0 up
```

**12. Verify the change.**

```bash
ip link show eth0
macchanger -s eth0
```

## Gotchas

- **The interface must usually be down.** Most drivers reject a MAC change
  while the interface is up. Use `ip link set DEV down` first and bring it
  back afterwards.
- **`-m` and `-r` can conflict.** Pick one mode; do not stack them.
- **`-a` preserves the vendor prefix.** That makes the address look like the
  same vendor, which is useful when the network filters on OUI.
- **`-A` picks a random known vendor.** It is more likely to collide with a
  real device on the same network.
- **`-e` is not "random ending".** It keeps the vendor bytes and changes only
  the ending; check the man page for the exact byte count on your version.
- **`-p` restores the permanent address.** That is the correct cleanup step
  after a test.
- **`-b` does not change the address by itself.** It marks the address as
  burned-in when combined with a change.
- **Random MACs may be blocked.** Some APs, captive portals, and enterprise
  networks reject locally-administered addresses.
- **The change is not persistent across reboots.** It applies to the running
  interface only.
- **Some drivers ignore MAC changes entirely.** Virtual interfaces and
  certain firmware-backed adapters may silently keep the old address; always
  verify with `ip link show`.
- **Changing the MAC does not change the DHCP lease identity.** The DHCP
  client may still use a client identifier derived from the old address.
- **`macchanger` is not a privacy tool by itself.** It changes the layer-2
  address, not the IP, hostname, or traffic patterns.
- **Root is required.** Writing a hardware address needs `CAP_NET_ADMIN`.
- **`-l` output is large.** Pipe it through `grep` or `head`.
- **Monitor-mode interfaces may need to be recreated.** Changing the MAC of a
  monitor interface can reset its channel or type.

## Related docs

- [`iw`](../iw/README.md) — inspect and configure wireless interfaces
- [`aircrack-ng`](../aircrack-ng/README.md) — monitor mode and wireless capture
- [`wifite`](../wifite/README.md) — automated wireless auditing with MAC
  randomisation
- [`bettercap`](../bettercap/README.md) — `mac.changer` module
- [`reaver`](../reaver/README.md) — WPS attacks
- [`wash`](../wash/README.md) — WPS survey
- [`tcpdump`](../tcpdump/README.md) — verify traffic after a change
