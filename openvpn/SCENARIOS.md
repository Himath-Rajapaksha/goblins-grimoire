# openvpn — command scenarios for deep research

Installed build: OpenVPN 2.6.14 (Debian `openvpn` `2.6.14-1+deb13u3`). Every
option below appears in [`help.txt`](help.txt) or the shipped man page; the
help snapshot is authoritative for the exact build. Commands that create a
tunnel, route, or device need root.

> **Scope:** A VPN connection changes routing and can expose a network the
> operator does not own. Use only profiles, servers, and credentials you are
> authorised to use, and record the pre-connection routes and DNS state so
> the tunnel can be removed cleanly.

## How to invoke

```text
@deep-research For an authorised engagement where I have a legitimate
OpenVPN 2.6 client profile, explain how to validate the profile before
connecting, how `--pull`, `--route-nopull`, `--redirect-gateway`, and
`--block-ipv6` interact, and which TLS pinning option (`--ca`,
`--verify-x509-name`, `--peer-fingerprint`) gives the strongest guarantee
without breaking the server. Cite the OpenVPN 2.6 manual, the OpenVPN
protocol documentation, and the systemd unit docs; separate confirmed
behaviour from inference and state confidence.
```

Capture the baseline before the first connection:

```bash
mkdir -p ~/ovpn-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/ovpn-demo/evidence/run-start.txt
ip addr > ~/ovpn-demo/evidence/ip-before.txt
ip route > ~/ovpn-demo/evidence/route-before.txt
ip -6 route > ~/ovpn-demo/evidence/route6-before.txt
cat /etc/resolv.conf > ~/ovpn-demo/evidence/resolv-before.txt
openvpn --show-gateway > ~/ovpn-demo/evidence/gateway-before.txt 2>&1
```

## Scenario 1 — inspect the build and validate a profile

Before connecting, learn what this binary supports and what the profile
actually asks for. A `.ovpn` file is executable configuration.

| # | Command | How to use |
|---|---------|------------|
| 1 | `openvpn --version` | Version, libraries, compile-time features, DCO status |
| 2 | `openvpn --help` | Option summary (see [`help.txt`](help.txt)) |
| 3 | `openvpn --show-gateway` | Current default gateway before the tunnel |
| 4 | `openvpn --show-ciphers` | Data-channel ciphers this build supports |
| 5 | `openvpn --show-digests` | HMAC digests |
| 6 | `openvpn --show-tls` | TLS ciphers and TLS 1.3 suites |
| 7 | `openvpn --show-engines` | Hardware crypto engines |
| 8 | `openvpn --show-pkcs11-ids` | Tokens and identities visible to PKCS#11 |
| 9 | `openvpn --config client.ovpn --test-crypto --verb 4` | Crypto self-test with the profile |
| 10 | `openvpn --config client.ovpn --verb 4 --dev null` | Parse the profile without opening a device |

**Research questions**
- Which compile-time features change what the profile can do (`DCO`,
  `PKCS11`, `LZO`, `LZ4`)?
- Which directives in the profile are dangerous (`--up`, `--plugin`,
  `--script-security`, `--route`)?
- Does `--test-crypto` exercise the same code path as a real handshake?
- What does `DCO version: N/A` imply for `--disable-dco`?
- Which ciphers in the profile are actually available in this build?

**Sources:** `openvpn --help`; `man openvpn`; OpenVPN 2.6 reference manual;
compile-time defines in `--version`.
**Output:** a profile review with a supported/unsupported feature table.

## Scenario 2 — connect, log, and verify the tunnel

The basic client path: connect, confirm the interface and routes, then
observe the timers that decide whether the session survives.

| # | Command | How to use |
|---|---------|------------|
| 11 | `sudo openvpn --config client.ovpn` | Foreground connect |
| 12 | `sudo openvpn --config client.ovpn --verb 3` | Recommended summary verbosity |
| 13 | `sudo openvpn --config client.ovpn --daemon ovpn-client` | Detach after init |
| 14 | `sudo openvpn --config client.ovpn --log /tmp/ovpn.log` | Truncate a log file |
| 15 | `sudo openvpn --config client.ovpn --log-append /tmp/ovpn.log` | Append to a log file |
| 16 | `sudo openvpn --config client.ovpn --writepid /run/ovpn.pid` | Write the PID |
| 17 | `ip addr show tun0` | Confirm the interface and address |
| 18 | `ip route get 10.10.0.5` | Confirm which path a target takes |
| 19 | `sudo openvpn --config client.ovpn --remote vpn.example.com 1194` | Override the remote |
| 20 | `sudo openvpn --config client.ovpn --proto tcp-client` | Force TCP |
| 21 | `sudo openvpn --config client.ovpn --connect-retry 2 30 --connect-retry-max 5` | Bounded retries |
| 22 | `sudo openvpn --config client.ovpn --resolv-retry infinite` | Retry DNS forever |
| 23 | `sudo openvpn --config client.ovpn --ping 10 --ping-restart 60` | Liveness detection |
| 24 | `sudo openvpn --config client.ovpn --inactive 3600` | Exit after an idle hour |
| 25 | `sudo openvpn --config client.ovpn --session-timeout 28800` | Hard 8-hour cap |

**Research questions**
- When does `--daemon` close stdio, and where do early errors go?
- How do `--ping`, `--ping-restart`, and `--keepalive` differ in what they
  restart?
- What does `--inactive` count, and why can it fire during a bulk transfer?
- Does `--connect-retry-max` count DNS failures as attempts?
- Which routes exist only because the server pushed them?

**Sources:** `man openvpn`; systemd unit files; OpenVPN logs from the run.
**Output:** a connection record with interface, routes, and timer settings.

## Scenario 3 — credentials, challenges, and the management interface

Profiles often need a username/password, a token, or a passphrase. These
options decide where the secret comes from and whether it is cached.

| # | Command | How to use |
|---|---------|------------|
| 26 | `printf 'alice\nS3cret\n' > /tmp/ovpn-creds && chmod 600 /tmp/ovpn-creds` | Credentials file (2 lines) |
| 27 | `sudo openvpn --config client.ovpn --auth-user-pass /tmp/ovpn-creds` | Read credentials from the file |
| 28 | `sudo openvpn --config client.ovpn --auth-user-pass` | Prompt on the console |
| 29 | `sudo openvpn --config client.ovpn --auth-nocache` | Do not cache the password |
| 30 | `sudo openvpn --config client.ovpn --auth-retry nointeract` | Fail instead of prompting |
| 31 | `sudo openvpn --config client.ovpn --static-challenge 'Token: ' 1` | Challenge/response with echo |
| 32 | `sudo openvpn --config client.ovpn --askpass /tmp/pass.txt` | Private-key passphrase from a file |
| 33 | `sudo openvpn --config client.ovpn --management 127.0.0.1 7505` | Local management socket |
| 34 | `sudo openvpn --config client.ovpn --management-query-passwords` | Ask management for secrets |
| 35 | `sudo openvpn --config client.ovpn --management-hold` | Start hibernated until management says go |
| 36 | `sudo openvpn --config client.ovpn --management-forget-disconnect` | Drop cached secrets on disconnect |
| 37 | `sudo openvpn --config client.ovpn --management-log-cache 100` | Cache log lines for management |

**Research questions**
- Which of these paths leaves the password in process memory or on disk?
- What does `--auth-retry interact` do to a scripted reconnect loop?
- How does `--static-challenge` differ from `--auth-user-pass` in the
  protocol?
- What can a local management socket do, and who can connect to it?
- Does `--management-hold` change when routes are installed?

**Sources:** `man openvpn`; OpenVPN management-interface documentation;
systemd hardening guidance.
**Output:** a credential-handling decision with a leak assessment.

## Scenario 4 — routing, DNS, and leak control

The difference between "connected" and "actually using the tunnel" is the
route table. These options are where a profile usually surprises you.

| # | Command | How to use |
|---|---------|------------|
| 38 | `sudo openvpn --config client.ovpn --redirect-gateway def1` | Default route via two /1 routes |
| 39 | `sudo openvpn --config client.ovpn --redirect-gateway local` | Keep the local subnet direct |
| 40 | `sudo openvpn --config client.ovpn --redirect-gateway bypass-dhcp` | Keep DHCP reachable |
| 41 | `sudo openvpn --config client.ovpn --route-nopull` | Ignore all pushed routes/DNS/DHCP |
| 42 | `sudo openvpn --config client.ovpn --route-nopull --route 10.10.0.0 255.255.0.0` | Pull nothing, add one route |
| 43 | `sudo openvpn --config client.ovpn --route-ipv6 fd00:10::/64` | Add an IPv6 route |
| 44 | `sudo openvpn --config client.ovpn --route-delay 5` | Wait before installing routes |
| 45 | `sudo openvpn --config client.ovpn --route-noexec` | Pass routes to scripts instead |
| 46 | `sudo openvpn --config client.ovpn --block-ipv6` | Send ICMPv6 unreachable instead of leaking |
| 47 | `sudo openvpn --config client.ovpn --pull-filter ignore 'route'` | Drop only pushed routes |
| 48 | `sudo openvpn --config client.ovpn --pull-filter reject 'redirect-gateway'` | Reject a pushed redirect |
| 49 | `sudo openvpn --config client.ovpn --allow-pull-fqdn` | Accept DNS names in pushed routes |
| 50 | `ip route` | Inspect the resulting table |
| 51 | `ip -6 route` | Inspect IPv6 |
| 52 | `cat /etc/resolv.conf` | Confirm which resolver is in use |

**Research questions**
- Why does `def1` use `0.0.0.0/1` and `128.0.0.0/1` instead of `0.0.0.0/0`?
- What is the difference between `--route-nopull` and `--pull-filter ignore`?
- Does `--block-ipv6` stop DNS AAAA queries from leaking?
- When does `--route-delay` matter, and when does it race the server push?
- How do you prove no traffic uses the physical default route?

**Sources:** `man openvpn`; Linux routing documentation; OpenVPN 2.6
reference manual.
**Output:** a route/DNS record and a leak verdict.

## Scenario 5 — proxies, transport tuning, and MTU

When the server is only reachable through a proxy or the path MTU is wrong,
these options are the difference between a handshake and a hang.

| # | Command | How to use |
|---|---------|------------|
| 53 | `sudo openvpn --config client.ovpn --http-proxy 127.0.0.1 8080` | HTTP CONNECT proxy |
| 54 | `sudo openvpn --config client.ovpn --http-proxy 127.0.0.1 8080 /tmp/proxy-creds` | Proxy credentials file |
| 55 | `sudo openvpn --config client.ovpn --http-proxy 127.0.0.1 8080 /tmp/proxy-creds ntlm` | NTLM proxy auth |
| 56 | `sudo openvpn --config client.ovpn --http-proxy 127.0.0.1 8080 auto` | Auto-detect proxy auth |
| 57 | `sudo openvpn --config client.ovpn --http-proxy-option VERSION 1.1` | Proxy HTTP version |
| 58 | `sudo openvpn --config client.ovpn --http-proxy-option AGENT 'Mozilla/5.0'` | Proxy user agent |
| 59 | `sudo openvpn --config client.ovpn --socks-proxy 127.0.0.1 1080` | SOCKS5 proxy |
| 60 | `sudo openvpn --config client.ovpn --socks-proxy 127.0.0.1 1080 /tmp/proxy-creds` | SOCKS5 with credentials |
| 61 | `sudo openvpn --config client.ovpn --socks-proxy-retry` | Retry forever on SOCKS errors |
| 62 | `sudo openvpn --config client.ovpn --tun-mtu 1400` | Lower the tunnel MTU |
| 63 | `sudo openvpn --config client.ovpn --mssfix 1360` | Clamp TCP MSS |
| 64 | `sudo openvpn --config client.ovpn --fragment 1300` | Fragment UDP datagrams |
| 65 | `sudo openvpn --config client.ovpn --mtu-test` | Measure path MTU |
| 66 | `sudo openvpn --config client.ovpn --sndbuf 0 --rcvbuf 0` | Let the kernel size buffers |
| 67 | `sudo openvpn --config client.ovpn --mark 0x1` | fwmark for policy routing |

**Research questions**
- Which proxy auth methods does this build actually offer?
- Why can a working handshake still stall on large packets?
- Does `--fragment` interact badly with `--mssfix`?
- What does `--mtu-test` change in the route table?
- When is `--mark` needed instead of a route?

**Sources:** `man openvpn`; HTTP CONNECT and SOCKS5 RFCs; Linux policy
routing documentation.
**Output:** a transport record with proxy and MTU settings.

## Scenario 6 — key material and certificate pinning

TLS is the part of OpenVPN that decides whether you are talking to the right
server. Generate keys deliberately and pin the peer deliberately.

| # | Command | How to use |
|---|---------|------------|
| 68 | `openvpn --genkey secret static.key` | Static shared secret |
| 69 | `openvpn --genkey tls-auth ta.key` | tls-auth HMAC key |
| 70 | `openvpn --genkey tls-crypt tc.key` | tls-crypt key |
| 71 | `openvpn --genkey tls-crypt-v2-server v2-server.key` | tls-crypt-v2 server key |
| 72 | `openvpn --tls-crypt-v2 v2-server.key --genkey tls-crypt-v2-client v2-client.key` | Wrapped client key |
| 73 | `openvpn --genkey auth-token token.key` | auth-gen-token key |
| 74 | `sudo openvpn --config client.ovpn --ca ca.crt --cert client.crt --key client.key` | Explicit identity |
| 75 | `sudo openvpn --config client.ovpn --pkcs12 bundle.p12` | PKCS#12 bundle |
| 76 | `sudo openvpn --config client.ovpn --tls-auth ta.key 0` | tls-auth with direction |
| 77 | `sudo openvpn --config client.ovpn --tls-crypt tc.key` | Encrypted control channel |
| 78 | `sudo openvpn --config client.ovpn --tls-version-min 1.2` | Minimum TLS |
| 79 | `sudo openvpn --config client.ovpn --tls-cipher TLS-ECDHE-RSA-WITH-AES-256-GCM-SHA384` | Restrict TLS 1.2 ciphers |
| 80 | `sudo openvpn --config client.ovpn --tls-ciphersuites TLS_AES_256_GCM_SHA384` | Restrict TLS 1.3 suites |
| 81 | `sudo openvpn --config client.ovpn --verify-x509-name vpn.example.com name` | Pin the subject CN |
| 82 | `sudo openvpn --config client.ovpn --verify-hash AB:CD:... SHA256` | Pin a level-1 fingerprint |
| 83 | `sudo openvpn --config client.ovpn --peer-fingerprint AB:CD:...` | Pin the peer cert |
| 84 | `sudo openvpn --config client.ovpn --remote-cert-tls server` | Require server EKU |
| 85 | `sudo openvpn --config client.ovpn --crl-verify crl.pem` | Check revocation |
| 86 | `sudo openvpn --config client.ovpn --data-ciphers AES-256-GCM:AES-128-GCM` | Restrict data ciphers |
| 87 | `sudo openvpn --config client.ovpn --data-ciphers-fallback AES-256-CBC` | Legacy fallback |
| 88 | `sudo openvpn --config client.ovpn --auth SHA256` | HMAC digest |
| 89 | `sudo openvpn --config client.ovpn --reneg-sec 3600` | Renegotiate hourly |

**Research questions**
- Which pinning option survives a CA compromise, and which does not?
- What exactly does `--verify-hash` pin compared with `--peer-fingerprint`?
- Does `--remote-cert-tls server` check the CN, the EKU, or both?
- Which of these options break a profile that uses a self-signed server?
- What happens to a session when `--reneg-sec` fires?

**Sources:** `man openvpn`; RFC 5280; OpenVPN 2.6 reference manual;
`--show-tls` output.
**Output:** a TLS policy record with the pinning decision and its residual
risk.

## Scenario 7 — server mode and client management

Standing up a lab server is the fastest way to understand what a client
profile is doing. These options cover the pool, pushes, and per-client
policy.

| # | Command | How to use |
|---|---------|------------|
| 90 | `sudo openvpn --mode server --tls-server --dev tun --port 1194 --proto udp --server 10.8.0.0 255.255.255.0 --ca ca.crt --cert server.crt --key server.key --dh dh.pem` | Minimal TLS server |
| 91 | `sudo openvpn --config server.conf --server-ipv6 fd00:8::/64` | Add an IPv6 pool |
| 92 | `sudo openvpn --config server.conf --server-bridge 10.8.0.4 255.255.255.0 10.8.0.50 10.8.0.100` | tap bridge pool |
| 93 | `sudo openvpn --config server.conf --push 'route 10.10.0.0 255.255.0.0'` | Push a route |
| 94 | `sudo openvpn --config server.conf --push 'dhcp-option DNS 10.8.0.1'` | Push DNS |
| 95 | `sudo openvpn --config server.conf --ifconfig-pool 10.8.0.10 10.8.0.200` | Explicit pool |
| 96 | `sudo openvpn --config server.conf --ifconfig-pool-persist /etc/openvpn/ipp.txt 600` | Persist leases |
| 97 | `sudo openvpn --config server.conf --client-config-dir /etc/openvpn/ccd` | Per-client configs |
| 98 | `sudo openvpn --config server.conf --ccd-exclusive` | Require a per-client file |
| 99 | `sudo openvpn --config server.conf --client-to-client` | Route client traffic internally |
| 100 | `sudo openvpn --config server.conf --max-clients 50` | Connection cap |
| 101 | `sudo openvpn --config server.conf --connect-freq 10 60` | Rate-limit new clients |
| 102 | `sudo openvpn --config server.conf --verify-client-cert require` | Require client certs |
| 103 | `sudo openvpn --config server.conf --auth-user-pass-verify /etc/openvpn/check.sh via-file` | External credential check |
| 104 | `sudo openvpn --config server.conf --username-as-common-name` | Use the auth username as CN |
| 105 | `sudo openvpn --config server.conf --duplicate-cn` | Allow duplicate CNs |
| 106 | `sudo openvpn --config server.conf --client-connect /etc/openvpn/connect.sh` | Per-client connect hook |
| 107 | `sudo openvpn --config server.conf --learn-address /etc/openvpn/learn.sh` | Validate addresses |
| 108 | `sudo openvpn --config server.conf --iroute 192.168.50.0 255.255.255.0` | Route a client subnet |
| 109 | `sudo openvpn --config server.conf --port-share 127.0.0.1 443` | Share TCP with HTTPS |
| 110 | `sudo openvpn --config server.conf --management 127.0.0.1 7505 --management-client-auth` | Delegate client auth |
| 111 | `sudo openvpn --config server.conf --status /var/log/openvpn-status.log 30` | Status file every 30 s |
| 112 | `sudo openvpn --config server.conf --explicit-exit-notify 1` | Tell clients on exit |

**Research questions**
- What does `--server` push that `--server-bridge` does not?
- Which hook runs before the client is fully authenticated?
- What breaks when `--duplicate-cn` is combined with `--ifconfig-pool-persist`?
- How does `--port-share` distinguish VPN from HTTPS on one TCP port?
- Which of these options are unsafe defaults for an internet-facing server?

**Sources:** `man openvpn`; OpenVPN 2.6 reference manual; `easy-rsa`
documentation in `/etc/openvpn/easy-rsa/`.
**Output:** a lab server configuration with a per-client policy table.

## Scenario 8 — operations, privilege drop, and cleanup

A tunnel is only clean if it can be removed. These options control scripts,
privileges, and the final state.

| # | Command | How to use |
|---|---------|------------|
| 113 | `sudo systemctl start openvpn-client@client.service` | Start the template unit |
| 114 | `journalctl -u openvpn-client@client.service -f` | Follow its log |
| 115 | `sudo systemctl stop openvpn-client@client.service` | Stop it |
| 116 | `sudo openvpn --mktun --dev tun9` | Create a persistent tun |
| 117 | `sudo openvpn --rmtun --dev tun9` | Remove it |
| 118 | `sudo openvpn --config client.ovpn --persist-tun --persist-key` | Keep device and keys across restarts |
| 119 | `sudo openvpn --config client.ovpn --remap-usr1 SIGTERM` | Make SIGUSR1 terminate |
| 120 | `sudo openvpn --config client.ovpn --mlock` | Lock key material in RAM |
| 121 | `sudo openvpn --config client.ovpn --user nobody --group nogroup` | Drop privileges |
| 122 | `sudo openvpn --config client.ovpn --chroot /var/empty` | Chroot after init |
| 123 | `sudo openvpn --config client.ovpn --script-security 2` | Allow scripts |
| 124 | `sudo openvpn --config client.ovpn --up /etc/openvpn/up.sh` | Run on device open |
| 125 | `sudo openvpn --config client.ovpn --down /etc/openvpn/down.sh` | Run on device close |
| 126 | `sudo openvpn --config client.ovpn --up-delay --up-restart` | Delay and repeat the hook |
| 127 | `sudo openvpn --config client.ovpn --down-pre` | Run down before device close |
| 128 | `sudo openvpn --config client.ovpn --setenv ROLE lab` | Pass a variable to hooks |
| 129 | `sudo openvpn --config client.ovpn --nice -5` | Raise priority |
| 130 | `sudo openvpn --config client.ovpn --suppress-timestamps` | Clean log lines |
| 131 | `sudo openvpn --config client.ovpn --machine-readable-output` | Stable log format |
| 132 | `sudo openvpn --config client.ovpn --mute 10` | Cap repeated messages |
| 133 | `sudo openvpn --config client.ovpn --status /tmp/ovpn-status.log 5 --status-version 3` | Machine-readable status |
| 134 | `ip route \| diff - ~/ovpn-demo/evidence/route-before.txt` | Prove the routes were restored |

**Research questions**
- Which script environment variables contain untrusted data?
- Does `--user` run before or after `--up`?
- What does `--persist-tun` leave behind after a failed reconnect?
- How do you prove the DNS and route state match the pre-connection baseline?
- Which systemd sandboxing options are compatible with tun/tap?

**Sources:** `man openvpn`; systemd `openvpn-client@.service`; OpenVPN
hardening guides.
**Output:** a teardown record with a diff against the baseline.

## Flag coverage checklist

| Group / option | Scenario |
|----------------|----------|
| `--version`, `--help`, `--show-gateway`, `--show-ciphers`, `--show-digests`, `--show-tls`, `--show-engines`, `--show-pkcs11-ids` | 1 |
| `--config`, `--test-crypto`, `--dev null`, `--verb` | 1, 2 |
| `--daemon`, `--log`, `--log-append`, `--writepid` | 2 |
| `--remote`, `--proto`, `--connect-retry`, `--connect-retry-max`, `--resolv-retry` | 2 |
| `--ping`, `--ping-restart`, `--inactive`, `--session-timeout` | 2 |
| `--auth-user-pass`, `--auth-nocache`, `--auth-retry`, `--static-challenge`, `--askpass` | 3 |
| `--management`, `--management-query-passwords`, `--management-hold`, `--management-forget-disconnect`, `--management-log-cache` | 3 |
| `--redirect-gateway`, `--route-nopull`, `--route`, `--route-ipv6`, `--route-delay`, `--route-noexec` | 4 |
| `--block-ipv6`, `--pull-filter`, `--allow-pull-fqdn` | 4 |
| `--http-proxy`, `--http-proxy-option`, `--socks-proxy`, `--socks-proxy-retry` | 5 |
| `--tun-mtu`, `--mssfix`, `--fragment`, `--mtu-test`, `--sndbuf`, `--rcvbuf`, `--mark` | 5 |
| `--genkey` (`secret`, `tls-auth`, `tls-crypt`, `tls-crypt-v2-server`, `tls-crypt-v2-client`, `auth-token`) | 6 |
| `--ca`, `--cert`, `--key`, `--pkcs12`, `--tls-auth`, `--tls-crypt`, `--tls-crypt-v2` | 6 |
| `--tls-version-min`, `--tls-cipher`, `--tls-ciphersuites`, `--tls-cert-profile` | 6 |
| `--verify-x509-name`, `--verify-hash`, `--peer-fingerprint`, `--remote-cert-tls`, `--crl-verify` | 6 |
| `--data-ciphers`, `--data-ciphers-fallback`, `--auth`, `--reneg-sec` | 6 |
| `--mode server`, `--tls-server`, `--server`, `--server-ipv6`, `--server-bridge` | 7 |
| `--push`, `--ifconfig-pool`, `--ifconfig-pool-persist`, `--client-config-dir`, `--ccd-exclusive` | 7 |
| `--client-to-client`, `--max-clients`, `--connect-freq`, `--verify-client-cert` | 7 |
| `--auth-user-pass-verify`, `--username-as-common-name`, `--duplicate-cn` | 7 |
| `--client-connect`, `--learn-address`, `--iroute`, `--port-share` | 7 |
| `--management-client-auth`, `--status`, `--status-version`, `--explicit-exit-notify` | 7, 8 |
| `--mktun`, `--rmtun`, `--persist-tun`, `--persist-key`, `--remap-usr1`, `--mlock` | 8 |
| `--user`, `--group`, `--chroot`, `--script-security`, `--up`, `--down`, `--up-delay`, `--up-restart`, `--down-pre`, `--setenv` | 8 |
| `--nice`, `--suppress-timestamps`, `--machine-readable-output`, `--mute` | 8 |
| `--local`, `--lport`, `--rport`, `--bind`, `--nobind`, `--float`, `--dev-type`, `--dev-node`, `--topology` | reference (transport variants) |
| `--ifconfig`, `--ifconfig-ipv6`, `--route-gateway`, `--route-ipv6-gateway`, `--route-metric`, `--redirect-private` | reference (addressing variants) |
| `--keepalive`, `--ping-exit`, `--ping-timer-rem`, `--ipchange`, `--multihome`, `--fast-io` | reference (timers/lifecycle) |
| `--shaper`, `--link-mtu`, `--tun-mtu-extra`, `--mtu-disc`, `--bind-dev`, `--txqueuelen`, `--memstats` | reference (link tuning) |
| `--secret`, `--replay-window`, `--replay-persist`, `--mute-replay-warnings`, `--no-replay` | reference (static-key/legacy) |
| `--capath`, `--dh`, `--extra-certs`, `--engine`, `--providers`, `--tls-timeout`, `--hand-window`, `--tran-window`, `--single-session`, `--tls-exit` | reference (TLS variants) |
| `--pkcs11-providers`, `--pkcs11-id`, `--pkcs11-protected-authentication`, `--pkcs11-private-mode`, `--pkcs11-cert-private`, `--pkcs11-pin-cache`, `--pkcs11-id-management` | reference (token variants) |
| `--x509-username-field`, `--x509-track`, `--keying-material-exporter`, `--remote-cert-ku`, `--remote-cert-eku`, `--ns-cert-type` | reference (certificate variants) |
| `--client-nat`, `--push-peer-info`, `--setenv FORWARD_COMPATIBLE`, `--ignore-unknown-option` | reference (compatibility) |
| `--ifconfig-noexec`, `--ifconfig-nowarn`, `--redirect-private`, `--block-ipv6` | reference (addressing/leak variants) |
| `--push-reset`, `--push-remove`, `--ifconfig-push`, `--ifconfig-ipv6-push`, `--iroute-ipv6`, `--disable`, `--tmp-dir` | reference (server variants) |
| `--hash-size`, `--bcast-buffers`, `--tcp-queue-limit`, `--tcp-nodelay`, `--stale-routes-check`, `--max-routes-per-client` | reference (server tuning) |
| `--gremlin`, `--compress`, `--allow-compression`, `--comp-lzo`, `--comp-noadapt` | reference (compression/debug) |
| `--vlan-tagging`, `--vlan-accept`, `--vlan-pvid`, `--plugin` | reference (server features) |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inspect the build and validate a profile | 10 |
| 2 | Connect, log, and verify the tunnel | 15 |
| 3 | Credentials, challenges, and management | 12 |
| 4 | Routing, DNS, and leak control | 15 |
| 5 | Proxies, transport tuning, and MTU | 15 |
| 6 | Key material and certificate pinning | 22 |
| 7 | Server mode and client management | 23 |
| 8 | Operations, privilege drop, and cleanup | 22 |
| **Total** | | **134** |

## Legal and safety

OpenVPN creates a real network interface and changes the system route table.
`--redirect-gateway` can move all traffic, `--block-ipv6` can break
connectivity, and `--up`/`--down` hooks run with the privileges OpenVPN has
at that point. Use only profiles and servers you are authorised to use,
record the route/DNS baseline before connecting, and restore it afterwards.
A `.ovpn` file can execute commands; treat it as code, not as data.
