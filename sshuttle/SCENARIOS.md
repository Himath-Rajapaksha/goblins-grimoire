# sshuttle — command scenarios for deep research

Installed build: sshuttle 1.3.1. Flags come from `help.txt`.

## How to invoke

```text
@deep-research Which sshuttle 1.3.1 options and firewall method give the most
reliable transparent tunnel through a hardened jump host on Debian 13 (nftables)
without looping the SSH connection, and how do --dns, --auto-nets, and
--user/--group interact in practice? Cite sshuttle docs, nftables behaviour, and
issue reports; return a tested command set and a teardown check.
```

---

## Scenario 1 — route internal subnets

| # | Command | How to use |
|---|---------|------------|
| 1 | `sshuttle -r analyst@dmz 10.10.0.0/16` | Single subnet through SSH |
| 2 | `sshuttle -r analyst@dmz 10.10.0.0/16 172.16.0.0/12` | Multiple subnets |
| 3 | `sshuttle -r analyst@dmz 10.10.1.0/24:443` | Limit capture to a port |
| 4 | `sshuttle -r analyst@dmz -s subnets.txt` | Subnets from a file |
| 5 | `sshuttle -r analyst@dmz 0.0.0.0/1 -x 192.168.1.0/24` | Everything but LAN |

**Research questions**
- Does a `/1` route include the SSH server address, causing a loop?
- How are port-qualified subnets represented in nftables?

**Sources:** sshuttle docs, `nft list ruleset`, routing table.
**Output:** working transparent routing for the target ranges.

---

## Scenario 2 — DNS handling

| # | Command | How to use |
|---|---------|------------|
| 6 | `sshuttle -r analyst@dmz --dns 10.10.0.0/16` | Capture and forward DNS |
| 7 | `sshuttle -r analyst@dmz --dns --to-ns 10.10.0.53 10.10.0.0/16` | Force a resolver |
| 8 | `sshuttle -r analyst@dmz --ns-hosts 10.10.0.53 10.10.0.0/16` | Only capture that resolver |
| 9 | `resolvectl query internal.example.net` | Verify resolution path |
| 10 | `dig @10.10.0.53 internal.example.net` | Direct resolver test |

**Research questions**
- Which resolver does `--dns` use when `--to-ns` is omitted?
- Does systemd-resolved interfere with sshuttle's DNS capture?

**Sources:** sshuttle docs, resolv.conf, resolved logs.
**Output:** internal names resolving through the tunnel.

---

## Scenario 3 — discovery and hostnames

| # | Command | How to use |
|---|---------|------------|
| 11 | `sshuttle -r analyst@dmz -N 10.10.0.0/16` | Auto-detect remote subnets |
| 12 | `sshuttle -r analyst@dmz -H 10.10.0.0/16` | Auto-populate hosts |
| 13 | `sshuttle -r analyst@dmz --seed-hosts dc01,dc02 10.10.0.0/16` | Seed host scan |
| 14 | `sshuttle -r analyst@dmz -N -H --dns` | Full discovery mode |
| 15 | `getent hosts dc01.internal` | Confirm host entries |

**Research questions**
- How aggressive is `-H`, and can it be scoped or rate-limited?
- Does `-N` trust remote routes blindly, including default routes?

**Sources:** sshuttle source, man page, host file diffs.
**Output:** an accurate local view of remote names and routes.

---

## Scenario 4 — daemon, user scoping, namespaces

| # | Command | How to use |
|---|---------|------------|
| 16 | `sshuttle -D --pidfile /run/sshuttle.pid -r analyst@dmz 10.10.0.0/16` | Background mode |
| 17 | `sshuttle -D --syslog -r analyst@dmz 10.10.0.0/16` | Syslog logging |
| 18 | `sshuttle --user tester -r analyst@dmz 10.10.0.0/16` | Scope to one user |
| 19 | `sshuttle --group sectools -r analyst@dmz 10.10.0.0/16` | Scope to one group |
| 20 | `sshuttle --namespace lab -r analyst@dmz 10.10.0.0/16` | Run in a netns |
| 21 | `sshuttle --namespace-pid 4242 -r analyst@dmz 10.10.0.0/16` | Borrow a process netns |

**Research questions**
- Do user/group rules match on UID or on socket owner for forwarded traffic?
- What breaks when sshuttle runs inside a netns on Debian 13?

**Sources:** nftables owner match docs, sshuttle issues.
**Output:** a persistent tunnel with least-privilege capture scope.

---

## Scenario 5 — methods, troubleshooting, teardown

| # | Command | How to use |
|---|---------|------------|
| 22 | `sshuttle --method nft -r analyst@dmz 10.10.0.0/16` | Force nftables |
| 23 | `sshuttle --method tproxy -r analyst@dmz 10.10.0.0/16` | Fallback method |
| 24 | `sshuttle -vv -r analyst@dmz 10.10.0.0/16` | Verbose diagnostics |
| 25 | `sshuttle --disable-ipv6 -r analyst@dmz 10.10.0.0/16` | IPv4-only tunnel |
| 26 | `sudo nft list ruleset \| grep -i sshuttle` | Inspect installed rules |
| 27 | `pkill -f sshuttle; sudo nft flush chain ip sshuttle ...` | Clean stale rules |
| 28 | `sshuttle --sudoers-no-modify` | Inspect the suggested sudo stanza (do not install blindly) |

**Research questions**
- Which method is selected by `auto` on a hybrid nft/iptables host?
- What is the safe cleanup procedure after a killed session?

**Sources:** sshuttle docs, nftables man page, kernel logs.
**Output:** a reproducible workaround and a clean firewall afterwards.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Routing | subnets `-x -X -s -N -H --seed-hosts --dns --ns-hosts --to-ns` | 1–3 |
| Connection | `-r -e --python --remote-shell --no-cmd-delimiter -l --method -t --namespace* --disable-ipv6 --no-latency-control --latency-buffer-size --wrap` | 1, 4, 5 |
| Process | `-D --pidfile --syslog --user --group -v -V --sudoers-* --no-sudo-pythonpath` | 4, 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — subnets | 5 |
| 2 — DNS | 5 |
| 3 — discovery | 5 |
| 4 — daemon/scope | 6 |
| 5 — troubleshooting | 7 |
