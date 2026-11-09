# tor — command scenarios for deep research

Installed build: Tor 0.4.9.15 (Debian `tor` `0.4.9.15-0+deb13u1`). Every torrc
option below appears in [`help-torrc-options.txt`](help-torrc-options.txt) and
every standalone flag in `man tor`. The shipped `torrc` listens on SOCKS
`9050`, DNS `5353`, and TransPort `9040`; the examples use a **separate test
instance** on ports 9150/9151 so they never disturb the system service.

> **Scope:** Tor changes where traffic appears to originate but does not make
> unauthorised access acceptable. Use it only where the target's rules and
> your engagement scope allow it, and remember that Tor does not protect
> traffic that bypasses the proxy.

## How to invoke

```text
@deep-research For an authorised privacy and network assessment on a Debian
13 host running Tor 0.4.9.15, explain how SocksPort, DNSPort, TransPort, and
the control port differ, how stream isolation flags change circuit reuse,
and how a v3 onion service is created and authenticated. Cite the tor(1) and
torrc(5) man pages, the Tor control-spec and rend-spec-v3 documents, and the
Tor Project support portal. Separate confirmed behaviour from inference and
state confidence, including what the local network can still observe.
```

Capture the baseline before any test instance:

```bash
mkdir -p ~/tor-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/tor-demo/evidence/run-start.txt
ip route > ~/tor-demo/evidence/route-before.txt
cat /etc/resolv.conf > ~/tor-demo/evidence/resolv-before.txt
ss -ltnup > ~/tor-demo/evidence/sockets-before.txt
```

## Scenario 1 — build, configuration, and service health

Tor's `--help` is deliberately terse, so the option inventory and the config
parser are the real documentation. Confirm both before changing anything.

| # | Command | How to use |
|---|---------|------------|
| 1 | `tor --version` | Version and linked libraries |
| 2 | `tor --list-torrc-options` | Every option name this build accepts |
| 3 | `tor --list-modules` | Optional modules compiled in |
| 4 | `tor --list-deprecated-options` | Options scheduled for removal |
| 5 | `tor --verify-config -f /etc/tor/torrc` | Parse and validate the system config |
| 6 | `tor --dump-config short` | Only values that differ from defaults |
| 7 | `tor --dump-config full` | Every effective value |
| 8 | `systemctl status tor` | Unit state and recent log lines |
| 9 | `systemctl is-enabled tor` | Start-at-boot policy |
| 10 | `systemctl is-active tor` | Current state |
| 11 | `ss -ltnp \| grep -E ':9050\|:9051\|:9040\|:5353'` | TCP listeners |
| 12 | `ss -lunp \| grep 5353` | UDP DNS listener |
| 13 | `journalctl -u tor -n 50 --no-pager` | Daemon log |
| 14 | `tor --quiet --verify-config -f /etc/tor/torrc` | Validate with no console noise |

**Research questions**
- Which options in `/etc/tor/torrc` differ from the built-in defaults?
- Does `--verify-config` check file permissions, or only syntax and values?
- Which module being absent changes relay, directory, or PoW behaviour?
- Why can a healthy daemon still fail every circuit build?
- Which ports are listening on all interfaces rather than loopback?

**Sources:** `tor --list-torrc-options`; `man tor`; `man torrc`; the Debian
`torrc`; `tor.service`.
**Output:** a build-and-config record with the effective listener set.

## Scenario 2 — SOCKS, DNS, and transparent-proxy client use

Tor exposes four different client surfaces. Using the wrong one is how DNS
and UDP leaks happen.

| # | Command | How to use |
|---|---------|------------|
| 15 | `curl --socks5-hostname 127.0.0.1:9050 https://check.torproject.org/api/ip` | Resolve the name at Tor (no DNS leak) |
| 16 | `curl --socks5 127.0.0.1:9050 https://check.torproject.org/api/ip` | Resolve locally first (leaks the name) |
| 17 | `curl -x socks5h://127.0.0.1:9050 https://example.com` | Same as `--socks5-hostname` |
| 18 | `torsocks curl https://check.torproject.org/api/ip` | Wrap an arbitrary program |
| 19 | `torsocks ssh user@203.0.113.10` | SSH through the SOCKS port |
| 20 | `proxychains4 -q curl https://check.torproject.org/api/ip` | Compare with a generic chainer |
| 21 | `dig @127.0.0.1 -p 5353 example.com` | Resolve through Tor's DNSPort |
| 22 | `dig @127.0.0.1 -p 5353 +short example.com` | Script-friendly form |
| 23 | `ss -lunp \| grep 5353` | Confirm DNSPort is UDP on loopback |
| 24 | `torsocks --shell` | Interactive torified shell |
| 25 | `torsocks -i curl https://check.torproject.org/api/ip` | Isolated SOCKS session |
| 26 | `torsocks show` | Current `LD_PRELOAD` state |
| 27 | `. torsocks on` | Set the current shell to Tor mode |
| 28 | `. torsocks off` | Unset it |

**Research questions**
- Which of these commands can leak a hostname to the local resolver?
- Does `torsocks` cover statically linked binaries, and why not?
- Which applications bypass `LD_PRELOAD` (setuid, Go static builds)?
- Does `DNSPort` answer TCP queries as well as UDP?
- What does `torsocks -i` isolate: the circuit, the credentials, or both?

**Sources:** `man torsocks`, `torsocks.conf(5)`; `man tor` (`DNSPort`,
`SocksPort`); Tor Project DNS-leak guidance.
**Output:** a client-surface table with a leak verdict per method.

## Scenario 3 — control port, authentication, and live state

The control port is a text protocol that exposes circuit and stream state and
accepts `SIGNAL` commands. Run it on a test instance with authentication.

| # | Command | How to use |
|---|---------|------------|
| 29 | `mkdir -p /tmp/tor-demo/data` | Private state directory |
| 30 | `printf 'SocksPort 9150\nControlPort 9151\nCookieAuthentication 1\nDataDirectory /tmp/tor-demo/data\nLog notice file /tmp/tor-demo/tor.log\n' > /tmp/tor-demo/torrc` | Minimal test config |
| 31 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate it |
| 32 | `tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Start the instance |
| 33 | `ss -ltnp \| grep -E ':9150\|:9151'` | Confirm both listeners |
| 34 | `xxd -p /tmp/tor-demo/data/control_auth_cookie \| tr -d '\n'` | Read the cookie as hex |
| 35 | `printf 'AUTHENTICATE <hex>\r\nGETINFO version\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Cookie-authenticated request |
| 36 | `tor --hash-password 'demo-passphrase'` | Generate a control password hash |
| 37 | `printf 'HashedControlPassword 16:...\n' >> /tmp/tor-demo/torrc` | Add password auth |
| 38 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart with both auth methods |
| 39 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO circuit-status\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Password-authenticated circuit state |
| 40 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETCONF SocksPort\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Read one live option |
| 41 | `printf 'AUTHENTICATE "demo-passphrase"\r\nSETCONF ExitNodes={nl}\r\nGETCONF ExitNodes\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Change an option at runtime |
| 42 | `printf 'AUTHENTICATE "demo-passphrase"\r\nSIGNAL NEWNYM\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Ask for fresh circuits |
| 43 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO stream-status\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Per-stream state |
| 44 | `printf 'AUTHENTICATE "demo-passphrase"\r\nSIGNAL HALT\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Clean shutdown |

**Research questions**
- What is the difference between cookie and password authentication, and
  which is safer on a multi-user host?
- Does `SETCONF` persist across a restart, or only until then?
- What does `SIGNAL NEWNYM` guarantee, and what does it not guarantee?
- Which control commands can change routing, logging, or exit policy?
- What happens if the control port is bound to a non-loopback address?

**Sources:** `man tor` (`ControlPort`, `CookieAuthentication`,
`HashedControlPassword`); `control-spec.txt` from torspec.
**Output:** a control-session transcript with an auth-method decision.

## Scenario 4 — bridges and pluggable transports

When direct relay connections are blocked, bridges replace the public entry
guard. Pluggable transports disguise the traffic, but they are **separate
binaries** that may not be installed.

| # | Command | How to use |
|---|---------|------------|
| 45 | `command -v obfs4proxy snowflake-client lyrebird meek-client` | Which transports exist (empty here) |
| 46 | `tor --list-modules` | What the build itself supports |
| 47 | `printf 'UseBridges 1\nBridge 192.0.2.10:443 0123456789ABCDEF0123456789ABCDEF01234567\n' >> /tmp/tor-demo/torrc` | Add a direct bridge line |
| 48 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate the bridge stanza |
| 49 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart |
| 50 | `grep -iE 'bridge\|bootstrap' /tmp/tor-demo/tor.log \| tail -20` | Did it use the bridge? |
| 51 | `printf 'ClientTransportPlugin obfs4 exec /usr/bin/obfs4proxy\nBridge obfs4 192.0.2.10:443 0123456789ABCDEF0123456789ABCDEF01234567 cert=EXAMPLE iat-mode=0\n' >> /tmp/tor-demo/torrc` | Declare a PT that is not installed |
| 52 | `tor --verify-config -f /tmp/tor-demo/torrc` | Does config validation catch a missing binary? |
| 53 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart and observe the failure |
| 54 | `grep -iE 'obfs\|transport\|bridge' /tmp/tor-demo/tor.log \| tail -20` | Exact error text |
| 55 | `sed -i '/^UseBridges 1$/,$d' /tmp/tor-demo/torrc` | Remove the bridge block |
| 56 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Return to direct relays |

**Research questions**
- Does `--verify-config` check that `ClientTransportPlugin` binaries exist?
- What does Tor log when a bridge line has a malformed fingerprint?
- Which transports are recommended today, and which are deprecated?
- Can a bridge line be reused by another user without leaking it?
- What is the difference between a bridge and a pluggable transport?

**Sources:** `man tor` (`UseBridges`, `Bridge`, `ClientTransportPlugin`);
Tor Project bridge and PT documentation; `tor --list-modules`.
**Output:** a transport inventory with the installed/missing gap recorded.

## Scenario 5 — publish a v3 onion service

An onion service is a local TCP target published under a self-authenticating
v3 address. Tor creates the key material when it first starts with the
`HiddenServiceDir` configured.

| # | Command | How to use |
|---|---------|------------|
| 57 | `mkdir -p /tmp/tor-demo/onion` | Service directory |
| 58 | `printf 'HiddenServiceDir /tmp/tor-demo/onion\nHiddenServicePort 80 127.0.0.1:8080\nHiddenServiceVersion 3\n' >> /tmp/tor-demo/torrc` | Map virtual port 80 to a local listener |
| 59 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate the service block |
| 60 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Start and generate keys |
| 61 | `cat /tmp/tor-demo/onion/hostname` | The generated `.onion` address |
| 62 | `ls -la /tmp/tor-demo/onion` | Key files and permissions |
| 63 | `python3 -m http.server 8080 --bind 127.0.0.1 & echo $! > /tmp/tor-demo/http.pid` | Local target |
| 64 | `curl --socks5-hostname 127.0.0.1:9150 http://<address>.onion/` | Reach the service through Tor |
| 65 | `tar -C /tmp/tor-demo -czf /tmp/tor-demo/onion-backup.tgz onion` | Back up the identity |
| 66 | `chmod 600 /tmp/tor-demo/onion-backup.tgz` | Protect the key archive |
| 67 | `grep -i hidden /tmp/tor-demo/tor.log \| tail -10` | Publication log |
| 68 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO onions/current\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Service state over control port |
| 69 | `printf 'HiddenServiceMaxStreams 10\nHiddenServiceEnableIntroDoSDefense 1\n' >> /tmp/tor-demo/torrc` | Stream cap and intro-point defence |
| 70 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate the additions |
| 71 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart |
| 72 | `cat /tmp/tor-demo/onion/hostname` | Address must be unchanged |
| 73 | `kill "$(cat /tmp/tor-demo/http.pid)"` | Stop the local target |
| 74 | `rm -rf /tmp/tor-demo/onion /tmp/tor-demo/onion-backup.tgz` | Destroy the test identity |

**Research questions**
- Which files in `HiddenServiceDir` are secret, and what does each one do?
- Why must the address stay the same after a restart, and what changes it?
- Does `HiddenServiceVersion 3` accept any other value in this build?
- What does `HiddenServiceEnableIntroDoSDefense` actually rate-limit?
- Can a service survive `Sandbox 1` if it is added at runtime?

**Sources:** `man tor` (`HiddenServiceDir`, `HiddenServicePort`,
`HiddenServiceVersion`, `HiddenServiceMaxStreams`,
`HiddenServiceEnableIntroDoSDefense`); `rend-spec-v3.txt`.
**Output:** an onion-service record with address, key inventory, and backup
status.

## Scenario 6 — path selection, isolation, and padding

Exit policy, country constraints, and stream isolation decide which circuits
exist and which streams may share them.

| # | Command | How to use |
|---|---------|------------|
| 75 | `printf 'ExitNodes {nl}\n' >> /tmp/tor-demo/torrc` | Prefer Dutch exits |
| 76 | `printf 'StrictNodes 1\n' >> /tmp/tor-demo/torrc` | Fail instead of falling back |
| 77 | `printf 'ExcludeNodes {cn},{ir}\n' >> /tmp/tor-demo/torrc` | Exclude relays by country |
| 78 | `printf 'ExcludeExitNodes {ru}\n' >> /tmp/tor-demo/torrc` | Exclude exits by country |
| 79 | `printf 'EntryNodes {de}\n' >> /tmp/tor-demo/torrc` | Constrain guards (may conflict) |
| 80 | `printf 'NewCircuitPeriod 30\n' >> /tmp/tor-demo/torrc` | Consider a new circuit every 30 s |
| 81 | `printf 'MaxCircuitDirtiness 600\n' >> /tmp/tor-demo/torrc` | Maximum circuit age |
| 82 | `printf 'CircuitBuildTimeout 30\n' >> /tmp/tor-demo/torrc` | Fixed build deadline |
| 83 | `printf 'LearnCircuitBuildTimeout 0\n' >> /tmp/tor-demo/torrc` | Disable adaptive timeout |
| 84 | `printf 'EnforceDistinctSubnets 1\n' >> /tmp/tor-demo/torrc` | Keep relays in distinct /16s |
| 85 | `printf 'ClientUseIPv4 0\nClientUseIPv6 1\n' >> /tmp/tor-demo/torrc` | IPv6-only client policy |
| 86 | `printf 'ConnectionPadding 1\nCircuitPadding 1\n' >> /tmp/tor-demo/torrc` | Traffic padding |
| 87 | `printf 'SocksPort 9160 IsolateSOCKSAuth\n' >> /tmp/tor-demo/torrc` | Isolate by SOCKS credentials |
| 88 | `printf 'SocksPort 9161 IsolateDestAddr\n' >> /tmp/tor-demo/torrc` | Isolate by destination address |
| 89 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate the whole block |
| 90 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart |
| 91 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO circuit-status\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Circuits after the constraints |
| 92 | `printf 'AUTHENTICATE "demo-passphrase"\r\nSIGNAL NEWNYM\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Force new circuits |
| 93 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO circuit-status\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Compare the new set |
| 94 | `printf 'AUTHENTICATE "demo-passphrase"\r\nGETINFO stream-status\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Which streams share a circuit |

**Research questions**
- Which combinations of `EntryNodes`, `ExitNodes`, and `StrictNodes` can
  make every circuit fail?
- Does `ExcludeNodes` also exclude a relay from being a guard or middle?
- What exactly does `MaxCircuitDirtiness` age out?
- How does `IsolateSOCKSAuth` decide which streams share a circuit?
- Does `ConnectionPadding` protect against a local observer or a global one?

**Sources:** `man tor` (path-selection and isolation options); Tor path-spec;
Tor Project stream-isolation documentation.
**Output:** a circuit table with the constraint set and an isolation map.

## Scenario 7 — relay keys and relay options (lab only)

Running a relay from an assessment host is usually inappropriate. These
commands exist to read the key and fingerprint surface, and to build a lab
relay on an isolated network.

| # | Command | How to use |
|---|---------|------------|
| 95 | `mkdir -p /tmp/tor-relay/keys` | Relay state directory |
| 96 | `tor --keygen --DataDirectory /tmp/tor-relay/keys` | Generate ed25519 identity keys |
| 97 | `tor --key-expiration sign --format iso8601 --DataDirectory /tmp/tor-relay/keys` | Signing-cert expiry |
| 98 | `printf 'Nickname goblinlab\n' >> /tmp/tor-demo/torrc` | Relay name |
| 99 | `printf 'ContactInfo lab@example.invalid\n' >> /tmp/tor-demo/torrc` | Operator contact |
| 100 | `printf 'ORPort 9001\n' >> /tmp/tor-demo/torrc` | Relay listener |
| 101 | `printf 'DirPort 9030\n' >> /tmp/tor-demo/torrc` | Directory mirror listener |
| 102 | `printf 'ExitRelay 0\n' >> /tmp/tor-demo/torrc` | Non-exit relay |
| 103 | `printf 'BandwidthRate 1 MB\nBandwidthBurst 2 MB\n' >> /tmp/tor-demo/torrc` | Rate limits |
| 104 | `printf 'AccountingMax 50 GB\nAccountingStart month 1 00:00\n' >> /tmp/tor-demo/torrc` | Transfer quota |
| 105 | `printf 'MyFamily <fingerprint>\n' >> /tmp/tor-demo/torrc` | Declare relay family |
| 106 | `printf 'Address 203.0.113.10\n' >> /tmp/tor-demo/torrc` | Public address override |
| 107 | `tor --verify-config -f /tmp/tor-demo/torrc` | Validate the relay block |
| 108 | `tor --list-fingerprint --DataDirectory /tmp/tor-relay/keys` | Nickname and fingerprint |

**Research questions**
- Which key files are created by `--keygen`, and which are secret?
- What does `--key-expiration sign` read, and what if no cert exists?
- Does `ExitRelay 0` alone make the relay a middle-only relay?
- What is `MyFamily` protecting against, and how is it verified?
- Why is `Address` needed behind NAT, and what breaks without it?

**Sources:** `man tor` (relay options, `--keygen`, `--key-expiration`);
Tor relay lifecycle and family-ID documentation.
**Output:** a key inventory and a lab-relay config with the non-exit policy
stated explicitly.

## Scenario 8 — logging, hardening, and teardown

The last pass tightens the test instance, proves it stopped, and compares
the host state with the baseline.

| # | Command | How to use |
|---|---------|------------|
| 109 | `printf 'Log info file /tmp/tor-demo/tor-info.log\n' >> /tmp/tor-demo/torrc` | Second log target at info level |
| 110 | `printf 'SafeLogging 1\n' >> /tmp/tor-demo/torrc` | Redact addresses in logs |
| 111 | `printf 'Sandbox 1\n' >> /tmp/tor-demo/torrc` | Seccomp sandbox |
| 112 | `printf 'User debian-tor\n' >> /tmp/tor-demo/torrc` | Drop privileges when started as root |
| 113 | `chown -R debian-tor:debian-tor /tmp/tor-demo` | Make the state directory writable after the drop |
| 114 | `printf 'PidFile /tmp/tor-demo/tor.pid\n' >> /tmp/tor-demo/torrc` | Explicit PID file |
| 115 | `printf 'RunAsDaemon 0\n' >> /tmp/tor-demo/torrc` | Stay in the foreground |
| 116 | `tor --quiet --verify-config -f /tmp/tor-demo/torrc` | Validate without console output |
| 117 | `tor --verify-config -f /tmp/tor-demo/torrc 2>&1 \| tail -3` | See the final verdict line |
| 118 | `kill "$(cat /tmp/tor-demo/tor.pid)"; tor -f /tmp/tor-demo/torrc & echo $! > /tmp/tor-demo/tor.pid` | Restart with the hardened config |
| 119 | `ss -ltnp \| grep -E ':9150\|:9151\|:9160\|:9161'` | Confirm the listener set |
| 120 | `printf 'AUTHENTICATE "demo-passphrase"\r\nSIGNAL HALT\r\nQUIT\r\n' \| nc 127.0.0.1 9151` | Clean shutdown |
| 121 | `ss -ltnp \| grep -E ':9150\|:9151' \|\| echo closed` | Prove the ports are gone |
| 122 | `ip route \| diff - ~/tor-demo/evidence/route-before.txt` | Routes unchanged |
| 123 | `rm -rf /tmp/tor-demo /tmp/tor-relay` | Remove all test state |

**Research questions**
- Does `Sandbox 1` block the control port, the onion service, or both?
- What does `SafeLogging` redact, and what still appears in the log?
- Does `User debian-tor` apply before or after the data directory is opened?
- Which log targets survive a `SIGNAL RELOAD`?
- How do you prove no test listener or route survived the teardown?

**Sources:** `man tor` (`Log`, `SafeLogging`, `Sandbox`, `User`, `PidFile`,
`RunAsDaemon`); Debian `tor.service` hardening; Tor Project support portal.
**Output:** a hardening record and a teardown diff against the baseline.

## Flag coverage checklist

| Group / option | Scenario |
|----------------|----------|
| `--version`, `--list-torrc-options`, `--list-modules`, `--list-deprecated-options` | 1 |
| `--verify-config`, `--dump-config short\|full`, `--quiet` | 1, 8 |
| `SocksPort`, `DNSPort`, `TransPort`, `NATDPort`, `HTTPTunnelPort` | 2 (client surfaces) |
| `SocksPolicy`, `SocksPort IsolateSOCKSAuth`, `IsolateDestAddr` | 2, 6 |
| `ControlPort`, `CookieAuthentication`, `CookieAuthFile`, `HashedControlPassword`, `ControlPortWriteToFile` | 3 |
| `GETINFO version`, `circuit-status`, `stream-status`, `onions/current`, `GETCONF`, `SETCONF`, `SIGNAL NEWNYM/RELOAD/HALT` | 3, 5, 6, 8 |
| `UseBridges`, `Bridge`, `ClientTransportPlugin`, `UpdateBridgesFromAuthority`, `BridgeDistribution` | 4 |
| `HiddenServiceDir`, `HiddenServicePort`, `HiddenServiceVersion`, `HiddenServiceMaxStreams`, `HiddenServiceEnableIntroDoSDefense` | 5 |
| `ExitNodes`, `EntryNodes`, `ExcludeNodes`, `ExcludeExitNodes`, `StrictNodes` | 6 |
| `NewCircuitPeriod`, `MaxCircuitDirtiness`, `CircuitBuildTimeout`, `LearnCircuitBuildTimeout` | 6 |
| `EnforceDistinctSubnets`, `ClientUseIPv4`, `ClientUseIPv6` | 6 |
| `ConnectionPadding`, `CircuitPadding`, `__DisablePredictedCircuits` | 6 |
| `--keygen`, `--key-expiration`, `--list-fingerprint` | 7 |
| `ORPort`, `DirPort`, `Nickname`, `ContactInfo`, `ExitRelay`, `ExitPolicy` | 7 |
| `BandwidthRate`, `BandwidthBurst`, `AccountingMax`, `AccountingStart`, `AccountingRule` | 7 |
| `MyFamily`, `Address`, `RelayBandwidthRate`, `RelayBandwidthBurst` | 7 |
| `Log`, `SafeLogging`, `Sandbox`, `User`, `PidFile`, `RunAsDaemon`, `DataDirectory`, `HardwareAccel` | 8 |
| `GeoIPFile`, `GeoIPv6File`, `HiddenServiceSingleHopMode`, `HiddenServiceNonAnonymousMode` | reference (country DB and unsafe modes) |
| `NATDPort`, `SocksPolicy`, `ControlSocket`, `CookieAuthFile`, `ControlPortWriteToFile` | reference (alternate transports and control auth) |
| `--defaults-torrc`, `--ignore-missing-torrc`, `--allow-missing-torrc`, `--hush`, `--keygen-family` | reference (invocation variants) |

Full text: [`help.txt`](help.txt) and
[`help-torrc-options.txt`](help-torrc-options.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Build, configuration, and service health | 14 |
| 2 | SOCKS, DNS, and transparent-proxy client use | 14 |
| 3 | Control port, authentication, live state | 16 |
| 4 | Bridges and pluggable transports | 12 |
| 5 | Publish a v3 onion service | 18 |
| 6 | Path selection, isolation, padding | 20 |
| 7 | Relay keys and relay options (lab) | 14 |
| 8 | Logging, hardening, teardown | 15 |
| **Total** | | **123** |

## Legal and safety

Tor is legal in most jurisdictions but restricted in some, and using it may
violate an employer's acceptable-use policy or a target's rules of
engagement. It hides the source of a connection from the destination, not
from a local observer, an ISP, or a compromised endpoint. Publishing an
onion service exposes a real local port; bind the target to loopback and
destroy test identities when finished. The control port can change routing
and stop the daemon, so never expose it beyond loopback.
