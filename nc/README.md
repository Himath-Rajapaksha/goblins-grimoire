# nc — netcat-traditional TCP/IP swiss army knife

- **Version:** netcat-traditional 1.10-50 (`nc -h` prints `[v1.10-50]`)
- **Binary:** `/usr/bin/nc` → alternatives → `/usr/bin/nc.traditional`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** netcat 1.10 by Hobbit (avian.org)
- **Docs:** `man nc`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`nc` opens a raw TCP or UDP connection, or listens for one, and shovels bytes
between the socket and stdin/stdout. It does not speak a protocol; it gives you
the socket. That makes it the fastest way to test whether a service is
reachable, to speak a text protocol by hand, to move a file without SCP, or to
stand up a throwaway listener.

**This is netcat-traditional, not netcat-openbsd.** The two share a name and a
purpose but differ in flags and in listener syntax. The differences that bite:

| Behavior | This build (traditional) | netcat-openbsd |
|----------|--------------------------|----------------|
| Listen | `nc -l -p 4444` | `nc -l 4444` |
| `-p` | Local/listen port | Source port |
| `-k` | Set `SO_KEEPALIVE` | Keep listening after a client disconnects |
| `-e` / `-c` | Present (exec after connect) | Not present |
| `-N` | Not present | Shutdown socket after stdin EOF |
| TLS | Not present | `-X`/`--ssl` in some builds |

If a tutorial uses `nc -lvnp 4444`, this build accepts it only because `-l -p`
is split across the combined flags; verify with `nc -h` before assuming any
other OpenBSD flag exists.

## Use cases

| Situation | Why nc |
|-----------|--------|
| Is the port open? | `nc -zv host port` |
| What does the service say first? | `nc host port` and read the banner |
| Speak HTTP/SMTP/Redis by hand | Type the request on stdin |
| Move a file with no SSH | Listener writes to a file, sender pipes it |
| Throwaway UDP receiver | `nc -u -l -p 5353` |
| Quick listener for a reverse shell (lab) | `nc -l -p 4444` plus `-e` |
| Test a proxy or a firewall rule | Connect through and observe |
| Port-range sweep | `nc -zv host 20-30` |

## Flag reference

### Connection and mode

| Flag | Meaning |
|------|---------|
| `-l` | Listen mode, for inbound connects |
| `-p port` | Local port number; **required for `-l`** |
| `-u` | UDP mode |
| `-z` | Zero-I/O mode, for scanning |
| `-v` | Verbose; use twice for more |
| `-n` | Numeric-only addresses, no DNS |
| `-w secs` | Timeout for connects and final net reads |
| `-i secs` | Delay interval for lines sent and ports scanned |
| `-q secs` | After EOF on stdin, wait this long, then quit (negative = forever) |
| `-k` | Set the keepalive option on the socket |
| `-r` | Randomize local and remote ports |
| `-b` | Allow UDP broadcasts |
| `-C` | Send CRLF as the line ending |
| `-t` | Answer TELNET negotiation |
| `-T tos` | Set Type Of Service |
| `-s addr` | Local source address |
| `-o file` | Hex dump of traffic |
| `-g gateway` | Source-routing hop point, up to 8 |
| `-G num` | Source-routing pointer: 4, 8, 12, … |
| `-e filename` | Program to exec after connect (dangerous) |
| `-c string` | Shell commands to exec after connect (dangerous) |
| `-h` | Help |

Ports may be single (`80`), a range (`20-30`), or a service name. Service
names containing hyphens must be escaped (`ftp\-data`).

## Recipes

**1. Banner grab.**

```bash
nc -v target.example 22
```

**2. Port check without sending data.**

```bash
nc -zv target.example 80 443 8080
```

**3. Port range sweep.**

```bash
nc -zv -w 1 target.example 1-1024
```

**4. Speak HTTP by hand.**

```bash
printf 'HEAD / HTTP/1.1\r\nHost: target.example\r\nConnection: close\r\n\r\n' \
  | nc -w 5 target.example 80
```

**5. Throwaway TCP listener.**

```bash
nc -l -p 4444 -v
```

**6. Receive a file.**

```bash
nc -l -p 9000 > received.bin
```

**7. Send a file.**

```bash
nc -w 5 receiver.example 9000 < source.bin
```

**8. UDP receiver.**

```bash
nc -u -l -p 5353 -v
```

**9. UDP sender.**

```bash
printf 'hello\n' | nc -u -w 2 receiver.example 5353
```

**10. Hex-dump an exchange while using it.**

```bash
nc -v -o ~/nc-demo/session.hex target.example 25
```

**11. Lab-only shell handler.**

```bash
nc -l -p 4444 -e /bin/bash
```

## Gotchas

- **`-l` needs `-p`.** `nc -l 4444` is an OpenBSD form; here the port after
  `-l` is treated differently. Use `nc -l -p 4444`.
- **`-k` is keepalive, not "keep listening".** A traditional listener exits
  after the first client disconnects. Wrap it in a loop or use `socat` with
  `fork`.
- **`-e` and `-c` are remote code execution.** They are present in this build.
  A listener with `-e` is an unauthenticated shell; never expose one.
- **`-z` does not read a banner.** It reports whether the connection succeeds,
  not what the service says.
- **`-w` applies to connect and to the final read.** It is not a per-read
  idle timeout; a chatty service can keep the connection open longer.
- **`-q` is about stdin EOF.** Without it, nc keeps the socket open until the
  network side closes. That is why `printf ... | nc host port` can appear to
  hang.
- **UDP has no connection or EOF.** `-u` succeeds even when nothing is
  listening, and a UDP listener needs `-p`.
- **`-C` rewrites line endings.** Do not use it for binary transfers.
- **`-o` records payload bytes in hex.** It can capture credentials; treat the
  file as sensitive.
- **No TLS.** Use `openssl s_client` or `socat OPENSSL` for TLS.
- **`nc` is not a scanner.** It is fine for a handful of ports; use
  [`nmap`](../nmap/README.md) or [`masscan`](../masscan/README.md) for real
  scanning.

## Related docs

- [`socat`](../socat/README.md) — relay with `fork`, TLS, PTY, and address
  options
- [`nmap`](../nmap/README.md) — real port and service scanning
- [`tcpdump`](../tcpdump/README.md) — capture what nc sends
- [`proxychains4`](../proxychains4/README.md) — force nc through a proxy
