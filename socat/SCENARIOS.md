# socat — command scenarios for deep research

Installed build: socat 1.8.0.3 (Debian package `socat`). General options are in
[`help.txt`](help.txt); address heads and option names are in
[`help-options.txt`](help-options.txt). `man socat` is the authoritative
reference for semantics.

> **Dangerous by design:** `EXEC`, `SYSTEM`, and `SHELL` addresses run programs
> on the machine running socat. A listener that reaches one of them is remote
> code execution unless something else restricts access. Use only in an
> isolated lab or on an authorized target.

## How to invoke

```text
@deep-research For an authorized network pivot lab, compare socat 1.8.0.3 with
ncat, ssh -L, and rinetd for TCP and UDP relaying, TLS termination, and UNIX
socket bridging. Which tool preserves half-close and error semantics, what are
the security consequences of EXEC/SYSTEM addresses, and which options control
resource limits and peer filtering? Cite the socat manual, the ncat manual, and
independent write-ups; return a relay design with an explicit trust boundary.
```

Create a demo workspace, replacing addresses with in-scope targets:

```bash
mkdir -p ~/socat-demo/evidence
printf 'relay test\n' > ~/socat-demo/payload.txt
```

## Scenario 1 — inventory, features, and address grammar

Before writing a relay, confirm what this build can do. Feature flags decide
whether TLS, TUN, SOCKS, or PTY addresses exist at all.

| # | Command | How to use |
|---|---------|------------|
| 1 | `socat -V` | Version and compiled features |
| 2 | `socat -h \| sed -n '1,25p'` | General options |
| 3 | `socat -hh \| grep -n 'opts:'` | Start of the common option list |
| 4 | `socat -hhh \| grep -c 'groups='` | Count address and option entries |
| 5 | `socat -d -d -u /dev/null /dev/null` | Minimal two-address smoke test |
| 6 | `socat -d -d -u OPEN:/etc/hostname STDOUT` | Read a file into stdout |

**Research questions**
- Which features are compiled in, and which addresses are therefore absent?
- What is the difference between `-h`, `-hh`, and `-hhh`?
- Why does the smoke test use `-u`, and what changes without it?
- What does the `groups=` tag on an address or option mean?

**Sources:** `socat -V`; `socat -hhh`; `man socat` (`ADDRESS TYPES`,
`ADDRESS OPTIONS`).
**Output:** a feature matrix and an address-family inventory.

## Scenario 2 — TCP listeners, clients, and forwarding

This is socat's core use: bind a listener, accept connections, and relay each
one to another address. `fork` is what makes a listener serve more than one
client.

| # | Command | How to use |
|---|---------|------------|
| 7 | `socat TCP-LISTEN:8080,reuseaddr,fork TCP:127.0.0.1:80` | Local port forward |
| 8 | `socat TCP-LISTEN:8081,reuseaddr,fork,bind=127.0.0.1 TCP:127.0.0.1:8080` | Bind to loopback only |
| 9 | `socat TCP-LISTEN:8082,reuseaddr,fork,max-children=10 TCP:127.0.0.1:8080` | Cap concurrent children |
| 10 | `socat TCP-LISTEN:8083,reuseaddr,fork,range=10.0.0.0/24 TCP:127.0.0.1:8080` | Peer allow-list |
| 11 | `socat - TCP:target.example:80` | stdin/stdout to TCP |
| 12 | `socat TCP-LISTEN:9000,reuseaddr OPEN:~/socat-demo/evidence/received.bin,creat,append` | Receive to a file |
| 13 | `socat -u OPEN:~/socat-demo/payload.txt TCP:receiver.example:9000` | Send a file |
| 14 | `socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash,pty,stderr,setsid` | Lab-only shell handler |

**Research questions**
- What happens to a listener without `fork` after the first client disconnects?
- How do `max-children` and `backlog` interact under load?
- Does `range=` filter before or after the TCP handshake completes?
- What does `pty` change about the child's I/O and signal handling?
- Why is command 14 a backdoor rather than a relay?

**Sources:** `man socat` (`TCP-LISTEN`, `TCP`, `EXEC`); TCP listen backlog
semantics; socat source `xio-listen.c`.
**Output:** a relay design with bind scope, concurrency cap, and peer filter.

## Scenario 3 — UDP and datagram relays

UDP has no connection and no EOF. socat offers several address shapes because
"receive from one peer and reply" differs from "receive from anyone" and from
"send to a fixed peer".

| # | Command | How to use |
|---|---------|------------|
| 15 | `socat UDP-RECVFROM:5353,fork UDP-SENDTO:10.0.0.5:53` | Request/response relay |
| 16 | `socat UDP-LISTEN:5353,fork UDP:10.0.0.5:53` | Connected-style relay |
| 17 | `socat -u UDP-RECV:5353 OPEN:~/socat-demo/evidence/capture.bin,creat,append` | One-way capture |
| 18 | `socat -u OPEN:~/socat-demo/payload.txt UDP-SENDTO:10.0.0.5:5353` | One-way send |
| 19 | `socat UDP-DATAGRAM:10.0.0.5:5353,sourceport=5353 -` | Fixed source port |
| 20 | `socat -u UDP4-RECV:5353,reuseaddr - \| xxd \| head` | Hex-dump received datagrams |
| 21 | `socat -T 5 UDP-RECVFROM:5353,fork UDP-SENDTO:10.0.0.5:53` | Inactivity limit |

**Research questions**
- What is the difference between `UDP-RECV`, `UDP-RECVFROM`, and `UDP-LISTEN`?
- Why can `UDP:` not report EOF, and how does that affect relay shutdown?
- When is `sourceport=` required by the upstream service?
- What does `fork` mean for a datagram address?
- How do you avoid a relay loop when both sides are UDP?

**Sources:** `man socat` (`UDP-*`); RFC 768; socat source `xio-udp.c`.
**Output:** a datagram flow diagram and a shutdown-condition note.

## Scenario 4 — UNIX sockets, files, PTYs, and child processes

These addresses bridge socat's byte pump to local operating-system objects.
They are the most useful for lab tooling and the most dangerous when exposed.

| # | Command | How to use |
|---|---------|------------|
| 22 | `socat UNIX-LISTEN:/tmp/app.sock,fork,unlink-early TCP:127.0.0.1:9000` | UNIX socket to TCP |
| 23 | `socat - UNIX-CONNECT:/tmp/app.sock` | Client for that socket |
| 24 | `socat OPEN:/dev/ttyUSB0,raw,echo=0 PTY,link=/tmp/ttyV0,wait-slave` | Serial to PTY |
| 25 | `socat EXEC:'/bin/date',pty,stderr -` | Run one program |
| 26 | `socat SYSTEM:'echo hello; id' -` | Run a shell command |
| 27 | `socat -u OPEN:/etc/hostname CREATE:~/socat-demo/evidence/copy.txt` | Write-only file |
| 28 | `socat -u OPEN:/etc/hostname OPEN:~/socat-demo/evidence/copy2.txt,creat,trunc` | Read/write file |
| 29 | `socat UNIX-LISTEN:/tmp/locked.sock,fork,unlink-early,mode=0600 TCP:127.0.0.1:9000` | Restrict socket mode |

**Research questions**
- Why does `CREATE` need `-u` or a one-way peer?
- What is the difference between `EXEC` and `SYSTEM` argument handling?
- When does `unlink-early` race with a connecting client?
- What does `wait-slave` do for a PTY consumer?
- Which of these addresses can escalate privileges if socat runs as root?

**Sources:** `man socat` (`OPEN`, `CREATE`, `EXEC`, `SYSTEM`, `PTY`);
`unix(7)`; `pty(7)`.
**Output:** a local-object bridge inventory with ownership and mode notes.

## Scenario 5 — TLS wrapping

socat can terminate or originate TLS without a separate proxy. Certificate
verification is off by default on the listener side unless configured, so the
options matter.

| # | Command | How to use |
|---|---------|------------|
| 30 | `openssl req -x509 -newkey rsa:2048 -nodes -keyout ~/socat-demo/key.pem -out ~/socat-demo/cert.pem -days 1 -subj '/CN=localhost'` | Self-signed test cert |
| 31 | `socat OPENSSL-LISTEN:8443,reuseaddr,fork,cert=~/socat-demo/cert.pem,key=~/socat-demo/key.pem,verify=0 TCP:127.0.0.1:8080` | TLS termination |
| 32 | `socat OPENSSL:target.example:443,verify=1,cafile=/etc/ssl/certs/ca-certificates.crt -` | Verified TLS client |
| 33 | `socat OPENSSL-LISTEN:8444,reuseaddr,fork,cert=server.pem,key=server.key,cafile=ca.pem,verify=1 TCP:127.0.0.1:8080` | Mutual TLS |
| 34 | `openssl s_client -connect 127.0.0.1:8443 -servername localhost` | Verify the listener |
| 35 | `socat OPENSSL-LISTEN:8445,reuseaddr,fork,cert=cert.pem,key=key.pem,ciphers='ECDHE-RSA-AES256-GCM-SHA384' TCP:127.0.0.1:8080` | Pin a cipher suite |
| 36 | `socat OPENSSL-LISTEN:8446,reuseaddr,fork,cert=cert.pem,key=key.pem,snihost=localhost TCP:127.0.0.1:8080` | Set SNI |

**Research questions**
- What exactly does `verify=0` disable, and what still protects the client?
- Does `cafile` on a listener require `verify=1` to have any effect?
- Which TLS versions does this OpenSSL build offer, and how do you restrict
  them through socat?
- How does `snihost=` change name-based virtual hosting?
- What evidence proves the relay is actually encrypting traffic?

**Sources:** `man socat` (`OPENSSL*`); OpenSSL `s_client` documentation; RFC
8446.
**Output:** a TLS relay config with a verification statement and a captured
handshake summary.

## Scenario 6 — proxies, logging, and verification

socat can originate through HTTP CONNECT and SOCKS, and it can record both
directions of a relay. That makes it a useful adapter and a useful evidence
tool.

| # | Command | How to use |
|---|---------|------------|
| 37 | `socat SOCKS5-CONNECT:127.0.0.1:1080:target.example:443 -` | SOCKS5 client |
| 38 | `socat SOCKS4A:127.0.0.1:1080:target.example:80 -` | SOCKS4a client |
| 39 | `socat PROXY:proxy.example:8080:target.example:80 -` | HTTP CONNECT |
| 40 | `socat -v -x TCP-LISTEN:9000,reuseaddr,fork TCP:10.0.0.5:80 2>~/socat-demo/evidence/relay.log` | Text and hex dump |
| 41 | `socat -r ~/socat-demo/evidence/left.bin -R ~/socat-demo/evidence/right.bin TCP-LISTEN:9001,reuseaddr,fork TCP:10.0.0.5:80` | Raw directional dumps |
| 42 | `socat --statistics -T 30 -d -d TCP-LISTEN:9002,reuseaddr,fork TCP:10.0.0.5:80` | Stats and inactivity limit |
| 43 | `socat -L /tmp/socat.lock TCP-LISTEN:9003,reuseaddr,fork TCP:10.0.0.5:80` | Single-instance lock |
| 44 | `ss -ltnp \| grep 9002` | Confirm the listener |
| 45 | `curl -sS http://127.0.0.1:9000/` | Confirm the relay path |

**Research questions**
- Which proxy protocol does the target actually accept, and how do you tell?
- Do `-v` and `-x` record both directions or only one?
- What do `-r` and `-R` capture that `-v` does not?
- Where does `--statistics` write, and what fields does it report?
- What does `-L` protect against, and what does it not?

**Sources:** `man socat` (`PROXY`, `SOCKS*`, logging options); RFC 1928 (SOCKS5);
RFC 7231 §4.3.6 (CONNECT); `ss(8)`.
**Output:** a relay evidence bundle with directional captures and a
verification transcript.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-V`, `-h`, `-hh`, `-hhh` | 1 |
| `-d`, `-d0`…`-d4`, `-D`, `--statistics` | 1, 6 |
| `-ly`, `-lf`, `-ls`, `-lm`, `-lp`, `-lu`, `-lh` | 6 |
| `-v`, `-x`, `-r`, `-R` | 6 |
| `-b`, `-s`, `-S` | 2, 6 |
| `-t`, `-T` | 3, 6 |
| `-u`, `-U` | 1, 3, 4 |
| `-g`, `-L`, `-W` | 6 |
| `-0`, `-4`, `-6` | 2, 3 |
| `fork`, `reuseaddr`, `bind`, `range`, `max-children`, `backlog` | 2 |
| `pty`, `raw`, `echo`, `stderr`, `setsid`, `wait-slave`, `mode` | 4 |
| `verify`, `cafile`, `capath`, `cert`, `key`, `ciphers`, `snihost` | 5 |
| `sourceport`, `lowport`, `connect-timeout`, `retry`, `forever`, `interval` | 2, 3 |

Defaults: buffer 8192 bytes, `-t` 0.5 s. Full text: [`help.txt`](help.txt) and
[`help-options.txt`](help-options.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Inventory and address grammar | 6 |
| 2 | TCP listeners and forwarding | 8 |
| 3 | UDP and datagram relays | 7 |
| 4 | UNIX sockets, files, PTYs, processes | 8 |
| 5 | TLS wrapping | 7 |
| 6 | Proxies, logging, verification | 9 |
| **Total** | | **45** |

## Legal and safety

socat can expose a shell, bypass a network boundary, or relay traffic around a
control. Use it only on systems you own or are authorized to test, bind
listeners to the narrowest interface that works, and never leave an `EXEC` or
`SYSTEM` listener reachable from an untrusted network. Traffic dumps contain
credentials; store them as secrets and delete them at engagement close.
