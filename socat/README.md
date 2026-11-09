# socat — multipurpose relay between two data channels

- **Version:** socat 1.8.0.3 (`socat -V`)
- **Binary:** `/usr/bin/socat` (Debian package `socat`)
- **Full help snapshot:** [`help.txt`](help.txt) — `socat -h`
- **Option snapshot:** [`help-options.txt`](help-options.txt) — `socat -hh`
- **Upstream:** http://www.dest-unreach.org/socat/
- **Docs:** `man socat` (very detailed), `man socat1`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

socat establishes two **bidirectional** byte streams — called addresses — and
relays data between them. One address might be a TCP listener, the other a
program's stdin/stdout; one might be a TLS socket, the other a UNIX socket; one
might be a file, the other a PTY. socat does not understand the protocol, which
is exactly why it can bridge almost anything to almost anything.

```text
socat [options] <bi-address> <bi-address>
```

Each address is a *head* plus optional *options*:

```text
<address-head>[,<option>[,<option>...]]
TCP-LISTEN:8080,reuseaddr,fork
```

A **dual address** joins two single addresses with `!!` and is used when one
side needs to be opened for reading and another for writing.

## Use cases

| Situation | Address pair |
|-----------|--------------|
| Quick TCP listener that runs a shell | `TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash,pty,stderr` |
| Port forward to an internal host | `TCP-LISTEN:8080,fork TCP:10.0.0.5:80` |
| UDP relay | `UDP-RECVFROM:5353,fork UDP-SENDTO:10.0.0.5:53` |
| TLS termination in front of a plain service | `OPENSSL-LISTEN:8443,cert=...,key=...,verify=0 TCP:127.0.0.1:8080` |
| UNIX socket to TCP | `UNIX-LISTEN:/tmp/app.sock,fork TCP:127.0.0.1:9000` |
| Serial or PTY bridge | `OPEN:/dev/ttyUSB0,raw,echo=0 PTY,link=/tmp/ttyV0` |
| One-shot file transfer | `TCP-LISTEN:9000,reuseaddr OPEN:out.bin,creat,append` |
| Protocol debugging | any pair plus `-v -x` |
| Reach a host through a SOCKS proxy | `SOCKS5-CONNECT:proxy:1080:target:443 -` |

## General options

| Option | Meaning |
|--------|---------|
| `-V` | Print version and compiled features, then exit |
| `-h`, `-?` | Help for options and address types |
| `-hh` | `-h` plus common address option names |
| `-hhh` | `-hh` plus every available address option name |
| `-d[ddd]` | Increase verbosity (up to 4; 2 recommended) |
| `-d0`…`-d4` | Set verbosity explicitly (0 errors … 4 debug) |
| `-D` | Analyze file descriptors before the transfer loop |
| `--experimental` | Enable experimental features |
| `--statistics` | Print transfer statistics on exit |
| `-ly[facility]` | Log to syslog (default facility `daemon`) |
| `-lf<file>` | Log to a file |
| `-ls` | Log to stderr (default when no other log is set) |
| `-lm[facility]` | Mixed log: stderr during init, then syslog |
| `-lp<name>` | Set the program name used in logging |
| `-lu` | Microsecond timestamps in logs |
| `-lh` | Add hostname to log messages |
| `-v` | Text dump of transferred data |
| `-x` | Hexadecimal dump of transferred data |
| `-r <file>` | Raw dump of left-to-right traffic |
| `-R <file>` | Raw dump of right-to-left traffic |
| `-b<size>` | Transfer buffer size (default 8192 bytes) |
| `-s` | Sloppy mode: continue on non-fatal errors |
| `-S<mask>` | Log these signals, overriding the default mask |
| `-t<timeout>` | Seconds to wait before closing the second channel (default 0.5) |
| `-T<timeout>` | Total inactivity timeout in seconds |
| `-u` | Unidirectional: first address read-only, second write-only |
| `-U` | Unidirectional, reversed |
| `-g` | Do not check address option groups |
| `-L <file>` | Try to obtain a lock, or fail |
| `-W <file>` | Try to obtain a lock, or wait |
| `-0` | Do not prefer an IP version |
| `-4` | Prefer IPv4 when unspecified |
| `-6` | Prefer IPv6 when unspecified |

## Address types by family

This build has `WITH_OPENSSL`, `WITH_SYSTEM`, `WITH_SHELL`, `WITH_EXEC`,
`WITH_PTY`, `WITH_TUN`, `WITH_SOCKS4/4A/5`, `WITH_PROXY`, `WITH_VSOCK`,
`WITH_POSIXMQ`, and `WITH_NAMESPACES`. `WITH_READLINE` and `WITH_FIPS` are
**not** compiled in.

| Family | Heads |
|--------|-------|
| TCP | `TCP-CONNECT`, `TCP-LISTEN`, `TCP4-*`, `TCP6-*` |
| UDP | `UDP-CONNECT`, `UDP-LISTEN`, `UDP-RECV`, `UDP-RECVFROM`, `UDP-SENDTO`, `UDP-DATAGRAM`, `UDP4-*`, `UDP6-*` |
| UDP-Lite | `UDPLITE-CONNECT`, `UDPLITE-LISTEN`, `UDPLITE-RECV*`, `UDPLITE-SENDTO` |
| SCTP / DCCP | `SCTP-*`, `DCCP-*` |
| TLS | `OPENSSL`, `OPENSSL-LISTEN`, `OPENSSL-DTLS-CLIENT`, `OPENSSL-DTLS-SERVER` |
| UNIX | `UNIX-CLIENT`, `UNIX-CONNECT`, `UNIX-LISTEN`, `UNIX-RECV*`, `UNIX-SENDTO`, `ABSTRACT-*` |
| Files / FDs | `OPEN`, `CREATE`, `GOPEN`, `FD`, `PIPE`, `SOCKETPAIR` |
| Process | `EXEC`, `SYSTEM`, `SHELL`, `PTY` |
| Proxy | `PROXY`, `SOCKS4`, `SOCKS4A`, `SOCKS5-CONNECT`, `SOCKS5-LISTEN` |
| Raw / link | `IP-*`, `IP4-*`, `IP6-*`, `INTERFACE`, `TUN`, `SOCKET-*` |
| Other | `ACCEPT-FD`, `POSIXMQ-*`, `VSOCK-CONNECT`, `VSOCK-LISTEN` |

Run `socat -hhh` for the complete list on this install.

## Address options

Options are appended after a comma and apply only to that address. The ones
that matter most in practice:

| Option | Applies to | Meaning |
|--------|-----------|---------|
| `fork` | listeners | Fork a child per connection |
| `reuseaddr` | listeners | Set `SO_REUSEADDR` |
| `reuseport` | listeners | Set `SO_REUSEPORT` |
| `bind=<addr>` | sockets | Bind to a specific local address |
| `backlog=<n>` | listeners | Listen backlog |
| `max-children=<n>` | listeners | Cap concurrent forked children |
| `range=<a-b>` | listeners | Accept only peers in this address range |
| `tcpwrap[=<file>]` | listeners | Apply TCP wrappers access control |
| `retry=<n>` | connect | Retry a failed connect `n` times |
| `forever` | connect | Retry forever |
| `interval=<sec>` | retry | Delay between retries |
| `connect-timeout=<sec>` | sockets | Connect timeout |
| `keepalive` | sockets | Enable TCP keepalive |
| `linger=<on>,<sec>` | sockets | Set `SO_LINGER` |
| `mss=<n>` | TCP | Set the maximum segment size |
| `pf=<family>` | sockets | Force `ip4` or `ip6` |
| `sourceport=<port>` | TCP/UDP | Fix the source port |
| `lowport` | TCP/UDP | Prefer a source port below 1024 |
| `nonblock` | FDs | Non-blocking I/O |
| `cloexec` | FDs | Close-on-exec |
| `raw` | terminals | Raw mode |
| `echo=<0\|1>` | terminals | Terminal echo |
| `crnl` | streams | Convert LF to CRLF |
| `ignoreeof` | streams | Keep the channel open after EOF |
| `pty` | `EXEC`/`SYSTEM` | Give the child a PTY |
| `stderr` | `EXEC`/`SYSTEM` | Merge the child's stderr into the stream |
| `setsid` | `EXEC`/`SYSTEM` | Run the child in a new session |
| `pipes` | `EXEC`/`SYSTEM` | Use pipes instead of a socketpair |
| `path=<dir>` | `EXEC` | Set the child's `PATH` |
| `login` | `EXEC` | Make the child a login shell |
| `su=<user>` | process | Switch user |
| `chroot=<dir>` | process | Change root |
| `umask=<mode>` | files/process | Set umask |
| `mode=<mode>` | files | Create mode |
| `user=`/`group=` | files | Ownership of a created node |
| `unlink-early`/`unlink-close` | UNIX sockets | Control socket-file removal |
| `verify=<0\|1>` | TLS | Verify the peer certificate |
| `cafile=`, `capath=` | TLS | CA bundle or directory |
| `cert=`, `key=` | TLS | Local certificate and key |
| `ciphers=` | TLS | Cipher list |
| `snihost=` | TLS | SNI hostname |
| `lockfile=<file>` | `APPL` | Serialize access with a lock |

## Recipes

**1. One-shot TCP listener that hands out a shell (lab only).**

```bash
socat TCP-LISTEN:4444,reuseaddr,fork EXEC:/bin/bash,pty,stderr,setsid
```

**2. Local port forward.**

```bash
socat TCP-LISTEN:8080,reuseaddr,fork TCP:10.0.0.5:80
```

**3. UDP relay.**

```bash
socat UDP-RECVFROM:5353,fork UDP-SENDTO:10.0.0.5:53
```

**4. TLS listener in front of a plain HTTP service.**

```bash
socat OPENSSL-LISTEN:8443,reuseaddr,fork,cert=server.pem,key=server.key,verify=0 \
  TCP:127.0.0.1:8080
```

**5. UNIX socket to TCP.**

```bash
socat UNIX-LISTEN:/tmp/app.sock,fork,unlink-early TCP:127.0.0.1:9000
```

**6. PTY bridge for a serial device.**

```bash
socat OPEN:/dev/ttyUSB0,raw,echo=0 PTY,link=/tmp/ttyV0,wait-slave
```

**7. Capture a stream while relaying it.**

```bash
socat -v -x TCP-LISTEN:9000,reuseaddr,fork TCP:10.0.0.5:80 \
  2>~/socat-demo/relay.log
```

**8. Through a SOCKS5 proxy.**

```bash
socat SOCKS5-CONNECT:127.0.0.1:1080:target.example:443 -
```

**9. Two-way file copy over TCP.**

```bash
# receiver
socat TCP-LISTEN:9000,reuseaddr OPEN:received.bin,creat,append
# sender
socat OPEN:source.bin TCP:receiver.example:9000
```

**10. Transfer statistics and a hard inactivity limit.**

```bash
socat --statistics -T 30 -d -d TCP-LISTEN:9000,reuseaddr,fork TCP:10.0.0.5:80
```

## Gotchas

- **socat is a byte pump, not a protocol proxy.** It will happily relay HTTP,
  but it does not parse, rewrite, or validate it.
- **`fork` is not automatic.** Without it, a listener handles one connection
  and exits. With it, each connection gets a child process.
- **`EXEC` and `SYSTEM` are remote code execution by design.** A listener that
  reaches `EXEC:/bin/bash` is an unauthenticated shell unless something else
  restricts access. Never expose one to an untrusted network.
- **`SYSTEM` runs `/bin/sh -c`; `EXEC` uses `execvp` with space-separated
  arguments.** Quoting rules differ, and `SYSTEM` is the more dangerous of the
  two.
- **`-t` is not a connect timeout.** It is the wait after one channel reaches
  EOF. Connect timeout is the `connect-timeout` address option; total
  inactivity is `-T`.
- **`-u` / `-U` are unidirectional.** Using them on a protocol that expects a
  reply produces a hang or a truncated exchange.
- **`reuseaddr` is usually required** to restart a listener quickly after a
  crash or a `Ctrl-C`.
- **TLS defaults are permissive.** `verify=0` disables peer verification; set
  `cafile`/`capath` and `verify=1` for a real check.
- **`PTY,link=` leaves a symlink behind.** Clean it up, or use
  `unlink-early`/`unlink-close` where applicable.
- **Logging can leak credentials.** `-v` and `-x` dump payloads, including
  passwords and tokens. Treat the log as sensitive.
- **`--statistics` writes to stderr on exit.** Redirect it if you need a clean
  data stream.

## Related docs

- [`nc`](../nc/README.md) — simpler ad-hoc TCP/UDP client and listener
- [`proxychains4`](../proxychains4/README.md) — force arbitrary programs
  through a proxy
- [`tcpdump`](../tcpdump/README.md) — capture what socat is relaying
- [`nmap`](../nmap/README.md) — verify that a socat listener is reachable
