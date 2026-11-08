# torsocks — command scenarios for deep research

Every torsocks 2.5.0 option and documented configuration directive, grouped by
use case. Flags from [`help.txt`](help.txt); defaults and directives from
`torsocks(1)`, `torsocks.conf(5)`, and `/etc/tor/torsocks.conf` on this host.

## How to invoke

```text
@deep-research For torsocks 2.5.0, which traffic from a CLI pentest tool can
still leak outside Tor (UDP, ICMP, raw sockets, statically linked binaries),
and how do I prove with packet captures that DNS and TCP are fully torified?
Cite Tor Project docs, torsocks source, and leak-test write-ups; return exact
commands and a verification checklist.
```

---

## Scenario 1 — prove traffic is actually going through Tor

| # | Command | How to use |
|---|---------|------------|
| 1 | `torsocks curl -s https://check.torproject.org/api/ip` | Expected: `"IsTor":true` |
| 2 | `curl -s https://check.torproject.org/api/ip` | Baseline direct exit IP |
| 3 | `torsocks curl -s https://ifconfig.me/ip` | Second opinion on exit IP |
| 4 | `torsocks wget -qO- https://check.torproject.org/api/ip` | Same check with wget |
| 5 | `torsocks dig +short example.com` | DNS resolution path |
| 6 | `torsocks getent hosts example.com` | Resolver via libc |
| 7 | `torsocks --version` | Confirm wrapper version |
| 8 | `torsocks -d curl -s https://example.com` | Debug: see SOCKS negotiation |
| 9 | `torsocks show` | Current `LD_PRELOAD` state |
| 10 | `ss -tnp \| grep 9050` | Confirm the SOCKS connection |

**Research questions**
- Does `torsocks dig` use Tor DNS or leak to the system resolver?
- What does a `check.torproject.org` false negative mean (exit relay flagged,
  not a leak)?
- Which test actually distinguishes a DNS leak from a TCP leak?

**Sources**: Tor Project leak-test docs, torsocks source, packet-capture
write-ups.

**Output**: a two-command proof pair (direct vs torified) plus a DNS check,
saved with timestamps.

---

## Scenario 2 — torify a single command

| # | Command | How to use |
|---|---------|------------|
| 11 | `torsocks curl https://example.com` | HTTP client |
| 12 | `torsocks wget https://example.com/file` | Download |
| 13 | `torsocks git clone https://github.com/org/repo` | Git over HTTPS |
| 14 | `torsocks ssh user@host.example -p 1234` | SSH (TCP only) |
| 15 | `torsocks scp file user@host:/tmp/` | SCP |
| 16 | `torsocks python3 script.py` | Any dynamically linked Python |
| 17 | `torsocks nmap -sT -Pn target.example` | TCP connect scan only |
| 18 | `torsocks sqlmap -u https://target.example --batch` | HTTP-based tool |
| 19 | `torsocks ffuf -u https://target.example/FUZZ -w words.txt` | Web fuzzing |
| 20 | `torsocks gobuster dir -u https://target.example -w words.txt` | Content discovery |
| 21 | `torsocks nuclei -u https://target.example` | Template scan |
| 22 | `torsocks nikto -h https://target.example` | Web server scan |
| 23 | `torsocks subfinder -d example.com -silent` | Passive subdomain lookup |

**Research questions**
- Which of these tools spawn helpers or use raw sockets that escape the
  `LD_PRELOAD`?
- Is scanning through Tor acceptable under the target's program policy, and
  which exit relays block it?
- For tools with native proxy support, is `--proxy socks5h://127.0.0.1:9050`
  more reliable than `torsocks`?

**Sources**: each tool's proxy docs, Tor abuse FAQ, bug-bounty program rules,
torsocks issues.

**Output**: a per-tool note: torified cleanly, needs native SOCKS, or cannot
be torified.

---

## Scenario 3 — isolate circuits per task

| # | Command | How to use |
|---|---------|------------|
| 24 | `torsocks -i curl https://example.com` | New circuit per process |
| 25 | `torsocks -i -i curl https://example.com` | Re-run for a fresh circuit |
| 26 | `torsocks -u taskA -p secret curl https://example.com` | Explicit SOCKS5 auth |
| 27 | `TORSOCKS_USERNAME=taskA TORSOCKS_PASSWORD=secret torsocks curl https://example.com` | Credentials via env |
| 28 | `TORSOCKS_ISOLATE_PID=1 torsocks curl https://example.com` | PID-based isolation |
| 29 | `torsocks --isolate curl https://example.com` | Long form of `-i` |
| 30 | `torsocks -i nmap -sT -Pn target.example` | Isolated scan circuit |
| 31 | `torsocks -i nuclei -u https://target.example` | Isolated scan circuit |

**Research questions**
- Does `-i` isolate by process only, or per connection within a process?
- What does Tor's `IsolateSOCKSAuth` default imply for `-u/-p`?
- Can two `-i` runs accidentally share a circuit through connection reuse?

**Sources**: Tor `IsolateSOCKSAuth` docs, torsocks `IsolatePID` docs, Tor
control-spec stream isolation notes.

**Output**: an isolation scheme (per task, per host, per account) and the
matching command prefix.

---

## Scenario 4 — choose the Tor instance and port

| # | Command | How to use |
|---|---------|------------|
| 32 | `torsocks curl https://example.com` | Default system Tor `127.0.0.1:9050` |
| 33 | `torsocks -P 9150 curl https://example.com` | Tor Browser's bundled Tor |
| 34 | `torsocks -a 127.0.0.1 -P 9150 curl https://example.com` | Explicit address + port |
| 35 | `TORSOCKS_TOR_PORT=9150 torsocks curl https://example.com` | Port via env |
| 36 | `TORSOCKS_TOR_ADDRESS=127.0.0.1 TORSOCKS_TOR_PORT=9150 torsocks curl https://example.com` | Address + port via env |
| 37 | `ss -ltnp \| grep -E '9050\|9150'` | See which SOCKS ports are live |
| 38 | `torsocks -P 9150 -d curl https://example.com` | Debug a wrong-port failure |

**Research questions**
- When should tools use the system Tor versus Tor Browser's Tor?
- Does Tor Browser's Tor allow SOCKS auth/isolation, or is it restricted?
- What happens if both Tor instances are running and you pick the wrong port?

**Sources**: Tor Browser design docs, torsocks man page, Tor control port docs.

**Output**: one documented default port per workflow, with a check command.

---

## Scenario 5 — shell-wide torification

| # | Command | How to use |
|---|---------|------------|
| 39 | `. torsocks on` | Add libtorsocks to `LD_PRELOAD` in zsh |
| 40 | `torsocks on` | Show usage; must be sourced to take effect |
| 41 | `. torsocks off` | Remove from `LD_PRELOAD` |
| 42 | `torsocks show` | Print current `LD_PRELOAD` |
| 43 | `torsocks --shell` | Start a new torified shell |
| 44 | `torsocks --shell` then `curl https://check.torproject.org/api/ip` | Verify inside the shell |
| 45 | `echo $LD_PRELOAD` | Confirm the library path |

**Research questions**
- Which child processes inherit `LD_PRELOAD`, and which drop it (setuid,
  statically linked, containers)?
- Does `--shell` read `.zshrc` and re-enable plugins that unset `LD_PRELOAD`?
- How do you scope torification to one shell without affecting the desktop?

**Sources**: `ld.so(8)`, torsocks docs, zsh startup docs.

**Output**: a shell-scoped torification recipe with an explicit exit step.

---

## Scenario 6 — authentication, IPv6, config file, local exceptions

| # | Command | How to use |
|---|---------|------------|
| 46 | `torsocks -u alice -p s3cret curl https://example.com` | SOCKS5 username/password |
| 47 | `torsocks -6 curl -6 https://[2606:4700:4700::1111]/` | Enable IPv6 sockets |
| 48 | `TORSOCKS_ENABLE_IPV6=1 torsocks curl -6 https://example.com` | IPv6 via env |
| 49 | `TORSOCKS_CONF_FILE=./torsocks.conf torsocks curl https://example.com` | Custom config |
| 50 | `TORSOCKS_LOG_LEVEL=5 TORSOCKS_LOG_FILE_PATH=./torsocks.log torsocks curl https://example.com` | File logging |
| 51 | `TORSOCKS_LOG_TIME=1 TORSOCKS_LOG_LEVEL=5 torsocks curl https://example.com` | Timestamped log |
| 52 | `torsocks --help` | Option summary |

Example config for a lab-only local exception:

```ini
TorAddress 127.0.0.1
TorPort 9050
OnionAddrRange 127.42.42.0/24
SOCKS5Username alice
SOCKS5Password s3cret
AllowInbound 0
AllowOutboundLocalhost 1
IsolatePID 1
EnableIPv6 0
```

| # | Directive | How to use |
|---|-----------|------------|
| 53 | `TorAddress 127.0.0.1` | Tor SOCKS address (numeric only) |
| 54 | `TorPort 9050` | Tor SOCKS port |
| 55 | `OnionAddrRange 127.42.42.0/24` | Fake range for `.onion` names |
| 56 | `SOCKS5Username alice` | Isolation username |
| 57 | `SOCKS5Password s3cret` | Isolation password |
| 58 | `AllowInbound 0` | Refuse non-localhost inbound |
| 59 | `AllowOutboundLocalhost 1` | TCP to loopback bypasses Tor |
| 60 | `AllowOutboundLocalhost 2` | TCP + UDP to loopback bypasses Tor |
| 61 | `IsolatePID 1` | Per-process circuit isolation |
| 62 | `EnableIPv6 1` | IPv6 sockets (incomplete support) |

**Research questions**
- Why is `AllowOutboundLocalhost` dangerous during an engagement?
- Does `-6` make the target connection IPv6, or only the local socket?
- Which log level shows the SOCKS request without exposing credentials?

**Sources**: `torsocks.conf(5)`, Tor IPv6 docs, torsocks source, issues on
IPv6 failure modes.

**Output**: a config file with every non-default choice justified and a
reminder that loopback exceptions bypass Tor.

---

## Scenario 7 — troubleshoot failures

| # | Command | How to use |
|---|---------|------------|
| 63 | `torsocks -d curl -v https://example.com` | Verbose SOCKS + curl |
| 64 | `TORSOCKS_LOG_LEVEL=5 torsocks curl https://example.com` | Maximum library logging |
| 65 | `ss -ltnp \| grep 9050` | Is Tor listening? |
| 66 | `systemctl status tor@default` | Is system Tor healthy? |
| 67 | `torsocks curl -s https://check.torproject.org/api/ip` | End-to-end proof |
| 68 | `ldd "$(command -v curl)"` | Is the binary dynamically linked? |
| 69 | `file "$(command -v nmap)"` | Static binary check |
| 70 | `torsocks -q curl https://example.com` | Suppress torsocks logs |
| 71 | `curl --proxy socks5h://127.0.0.1:9050 https://example.com` | Native SOCKS fallback |
| 72 | `torsocks -P 9150 curl -s https://check.torproject.org/api/ip` | Try Tor Browser's port |

**Research questions**
- `Connection refused` vs `SOCKS5 error` vs silent timeout: what does each
  point to?
- Which tools need `socks5h` (remote DNS) instead of `socks5`?
- Why do some Go/Rust binaries ignore `LD_PRELOAD`?

**Sources**: torsocks issue tracker, Tor logs, Go/Rust networking docs, tool
proxy documentation.

**Output**: a decision tree from symptom to fix, with the final proof command.

---

## Complete option and directive coverage

Every documented torsocks option, environment variable, and config directive
appears at least once.

| Surface | Names | Scenario |
|---------|-------|----------|
| Options | `-h/--help`, `--shell`, `--version`, `-d/--debug`, `-u/--user`, `-p/--pass`, `-i/--isolate`, `-6/--ipv6`, `-a/--address`, `-P/--port`, `-q/--quiet`, `on`, `off`, `show`/`sh` | 1–7 |
| Environment | `TORSOCKS_CONF_FILE`, `TORSOCKS_USERNAME`, `TORSOCKS_PASSWORD`, `TORSOCKS_ISOLATE_PID`, `TORSOCKS_TOR_ADDRESS`, `TORSOCKS_TOR_PORT`, `TORSOCKS_ENABLE_IPV6`, `TORSOCKS_LOG_LEVEL`, `TORSOCKS_LOG_FILE_PATH`, `TORSOCKS_LOG_TIME`, `TORSOCKS_ALLOW_INBOUND` | 3, 4, 6, 7 |
| Config | `TorAddress`, `TorPort`, `OnionAddrRange`, `SOCKS5Username`, `SOCKS5Password`, `AllowInbound`, `AllowOutboundLocalhost`, `IsolatePID`, `EnableIPv6` | 6 |

Notes:
- `-q/--quiet` is in `torsocks(1)` but not in the short `--help` text.
- torsocks is an `LD_PRELOAD` shim: statically linked binaries and raw
  sockets are not covered. Use a native SOCKS option or a Tor transparent
  proxy when that matters.
- Default config path is `/etc/tor/torsocks.conf` (present on this host);
  `TORSOCKS_CONF_FILE` overrides it.

Full help text: [`help.txt`](help.txt); authoritative reference:
`man torsocks`, `man torsocks.conf`.
