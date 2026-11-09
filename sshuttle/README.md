# sshuttle — VPN over SSH without a remote VPN server

- **Version:** sshuttle 1.3.1
- **Binary:** `/usr/bin/sshuttle`
- **Help snapshot:** [`help.txt`](help.txt) (full `--help`)
- **Upstream:** https://github.com/sshuttle/sshuttle
- **Docs:** https://sshuttle.readthedocs.io/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

sshuttle turns an ordinary SSH login into a transparent TCP (and optionally
DNS) tunnel. A local firewall rule intercepts traffic for the routed subnets
and forwards it over the SSH connection to a small Python helper on the remote
host, which opens the real connections. No remote VPN daemon, tun device, or
root access on the server is required — only Python and SSH.

## Use cases

| Situation | Why sshuttle |
|-----------|--------------|
| Pivot into an internal network | Reach RFC1918 ranges through one SSH box |
| Avoid installing a VPN server | Nothing to deploy beyond Python |
| DNS-aware tunneling | `--dns` resolves names on the remote side |
| Selective routing | Include/exclude subnets per host or port |
| Per-user traffic capture | `--user`/`--group` scope the rules |
| Quick lab access | One command from a laptop |

## Requirements & install layout

- **Root locally** (firewall/NAT rules); the remote account does **not** need
  root.
- Python 3 on the remote host; `--python` selects an explicit interpreter.
- Debian package 1.3.1; binary at `/usr/bin/sshuttle`.
- `--method auto` picks `nft` on modern Debian (nftables), falling back to
  `nat`/`tproxy`. Check with `--method nft`.
- The remote shell is assumed POSIX; use `--remote-shell` for Windows targets.

## Basic syntax

```bash
# route one subnet through an SSH host
sshuttle -r user@jump.example.net 10.10.0.0/16

# route several subnets and remote DNS
sshuttle -r user@jump --dns 10.10.0.0/16 172.16.0.0/12

# exclude the jump host's own subnet from the tunnel
sshuttle -r user@jump 0.0.0.0/1 -x 10.10.5.0/24

# run in the background with a pidfile and syslog
sshuttle -D --pidfile /run/sshuttle.pid -r user@jump 10.10.0.0/16
```

## Flag reference

### Routing
| Flag | Meaning |
|------|---------|
| `<subnets...>` | Networks to capture: `IP/MASK[:PORT[-PORT]]` |
| `-x, --exclude <subnet>` | Exclude a subnet/port range (repeatable) |
| `-X, --exclude-from <file>` | Read exclusions from a whitespace-separated file |
| `-s, --subnets <file>` | Read the routed subnets from a file |
| `-N, --auto-nets` | Ask the remote side which subnets to route |
| `-H, --auto-hosts` | Continuously discover remote hostnames into `/etc/hosts` |
| `--seed-hosts <names>` | Seed the host scan without full `--auto-hosts` |
| `--dns` | Capture local DNS and forward to the remote resolver |
| `--ns-hosts <ips>` | Only capture DNS sent to these servers |
| `--to-ns <ip[:port]>` | Forward captured DNS to a specific server |

### Connection
| Flag | Meaning |
|------|---------|
| `-r, --remote [user[:pass]@]host[:port]` | SSH server and optional credentials |
| `-e, --ssh-cmd <cmd>` | SSH command to use (default `ssh`) |
| `--python <path>` | Remote Python interpreter |
| `--remote-shell <prog>` | Remote shell (`cmd`, `powershell`, ...) |
| `--no-cmd-delimiter` | Do not add `--` before the remote command |
| `-l, --listen [ip:]port` | Local transproxy listen address |
| `--method <type>` | `auto`, `nft`, `nat`, `tproxy`, `pf`, `ipfw` |
| `-t, --tmark <hex>` | tproxy packet mark (default `0x01`) |
| `--namespace <name>` / `--namespace-pid <pid>` | Run inside a net namespace |
| `--disable-ipv6` | Disable IPv6 handling |
| `--no-latency-control` | Trade latency control for throughput |
| `--latency-buffer-size <size>` | Tune the latency buffer |
| `--wrap <n>` | Testing aid for channel numbers |

### Process control
| Flag | Meaning |
|------|---------|
| `-D, --daemon` | Run in the background |
| `--pidfile <path>` | Pidfile when daemonised (default `./sshuttle.pid`) |
| `--syslog` | Log to syslog (default with `--daemon`) |
| `--user <user>` / `--group <group>` | Apply rules only to that user/group |
| `-v, --verbose` | Increase verbosity (repeatable) |
| `-V, --version` | Print version |
| `--sudoers-no-modify` | Print a passwordless sudoers stanza (insecure) |
| `--sudoers-user <user>` | User/group for that stanza |
| `--no-sudo-pythonpath` | Do not set `PYTHONPATH` under sudo |

## Recipes

```bash
# 1. Reach an internal /16 through a DMZ host
sshuttle -r analyst@dmz.example.net 10.10.0.0/16

# 2. Add remote DNS and a second subnet
sshuttle -r analyst@dmz --dns 10.10.0.0/16 172.16.0.0/12

# 3. Exclude a noisy management range
sshuttle -r analyst@dmz 10.0.0.0/8 -x 10.10.99.0/24

# 4. Route everything except the local LAN
sshuttle -r analyst@dmz 0.0.0.0/1 -x 192.168.1.0/24

# 5. Auto-discover remote routes and hostnames
sshuttle -r analyst@dmz -N -H --dns

# 6. Daemonise with logging
sudo sshuttle -D --pidfile /run/sshuttle.pid --syslog -r analyst@dmz 10.10.0.0/16

# 7. Only capture traffic from one local user
sudo sshuttle --user tester -r analyst@dmz 10.10.0.0/16

# 8. Verify the tunnel
ip route get 10.10.1.5
sudo nft list ruleset | grep -i sshuttle
```

## Output & pipelines

- Traffic is transparent: applications need no proxy settings.
- `--dns` rewrites `/etc/resolv.conf` handling internally; verify with
  `resolvectl query` or `dig`.
- Logs go to stderr (or syslog with `-D`); `-vv` shows firewall rules and
  channel setup.
- Forwarded connections appear on the remote host as ordinary outbound TCP
  from the SSH user, useful for egress logging.

## Gotchas & operational security

- **Local root is mandatory**; the firewall/NAT rules are added and removed by
  sshuttle itself.
- Only **TCP** is tunnelled by default (plus DNS with `--dns`); UDP and ICMP
  are not.
- `--auto-nets` can pull in huge ranges; always review before accepting.
- `--sudoers-no-modify` output grants more than sshuttle — never install it
  blindly.
- If the session dies uncleanly, stale nft/NAT rules can persist; restart
  sshuttle or flush its chains.
- `-x` exclusions must cover the SSH server itself or the tunnel can loop.
- Corporate SSH gateways may forbid port forwarding; `AllowTcpForwarding no`
  breaks sshuttle.

## Legal & safety

Tunnelling changes network paths and may violate monitoring or acceptable-use
policy. Use only on hosts and networks you are authorized to access, and
respect egress controls.

## Related tools

- [`gophish/`](../gophish/README.md) — separate phishing infrastructure
- [`recon-ng/`](../recon-ng/README.md) — OSINT from the tunnelled network
- [`kerbrute/`](../kerbrute/README.md) — AD enumeration behind the pivot
- [`nbtscan/`](../nbtscan/README.md), [`snmp/`](../snmp/README.md)

## References

- https://sshuttle.readthedocs.io/
- https://github.com/sshuttle/sshuttle
- `man sshuttle`
