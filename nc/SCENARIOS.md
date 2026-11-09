# nc — command scenarios for deep research

Installed build: netcat-traditional 1.10-50 (Debian package
`netcat-traditional`). `/usr/bin/nc` is an alternatives symlink to
`/usr/bin/nc.traditional`. Every flag below appears in [`help.txt`](help.txt)
or `man nc`.

> **Dangerous flags present:** `-e` and `-c` execute a program or shell after
> connect. A listener using them is unauthenticated remote code execution. Use
> only in an isolated lab or on an authorized target.

## How to invoke

```text
@deep-research Compare netcat-traditional 1.10-50 with netcat-openbsd, ncat,
and socat for ad-hoc TCP/UDP testing, file transfer, and listener behavior.
Which build is installed here, how do `-l`, `-p`, `-k`, `-q`, and `-w` differ
between them, and when does a "connection succeeded" result from `-z` or `-u`
mislead? Cite the netcat-traditional manual, the OpenBSD nc manual, the ncat
manual, and independent comparisons; return a compatibility table and a test
plan.
```

Create a demo workspace, replacing addresses with in-scope targets:

```bash
mkdir -p ~/nc-demo/evidence
printf 'netcat payload\n' > ~/nc-demo/payload.bin
sha256sum ~/nc-demo/payload.bin > ~/nc-demo/payload.sha256
```

## Scenario 1 — identify the build and its syntax

The name `nc` is ambiguous. Confirm which implementation is installed before
copying a command from a tutorial.

| # | Command | How to use |
|---|---------|------------|
| 1 | `nc -h` | Print this build's usage and version |
| 2 | `readlink -f /usr/bin/nc` | Resolve the alternatives symlink |
| 3 | `dpkg-query -W -f='${Version}\n' netcat-traditional` | Installed package version |
| 4 | `update-alternatives --query nc` | Which alternative is selected |
| 5 | `man nc \| col -b \| sed -n '1,40p'` | Synopsis and option list |
| 6 | `nc -zv -w 1 127.0.0.1 22` | Smoke test against a local port |

**Research questions**
- Which flags in an OpenBSD tutorial are absent here?
- Why does the alternatives system matter for scripts?
- Does this build accept `nc -l 4444`, and what does it do instead?
- Which of `-e`, `-c`, `-k`, `-N`, `-X`, and `--ssl` exist here?

**Sources:** `nc -h`; `man nc`; `update-alternatives --query nc`; Debian
package metadata; OpenBSD `nc(1)`.
**Output:** a compatibility table with present/absent flags and the selected
alternative.

## Scenario 2 — connectivity checks and banners

`-z` answers "can I connect", not "what is this". Use a real read for the
banner, and remember that `-w` covers connect plus the final read.

| # | Command | How to use |
|---|---------|------------|
| 7 | `nc -zv target.example 80` | Single-port check |
| 8 | `nc -zv target.example 80 443 8080` | Several ports |
| 9 | `nc -zv -w 1 target.example 1-1024` | Port range |
| 10 | `nc -zvu -w 1 target.example 53` | UDP check (weak evidence) |
| 11 | `nc -v -w 3 target.example 22` | Read the SSH banner |
| 12 | `nc -v -n 10.0.0.5 25` | Numeric address, SMTP banner |
| 13 | `printf 'HEAD / HTTP/1.1\r\nHost: target.example\r\nConnection: close\r\n\r\n' \| nc -w 5 target.example 80` | Minimal HTTP request |
| 14 | `nc -v -o ~/nc-demo/evidence/smtp.hex target.example 25` | Hex-dump the exchange |

**Research questions**
- What does `-z` prove, and what does it not prove about the service?
- Why is a UDP `-z` result unreliable in both directions?
- How does `-n` change name resolution and timing?
- Does `-w` bound the whole session or only connect and the final read?
- What is in the `-o` hex dump that a packet capture would also show?

**Sources:** `man nc`; TCP three-way handshake; RFC 5321 (SMTP); RFC 9112
(HTTP/1.1).
**Output:** a reachability table with banner text and a confidence note.

## Scenario 3 — ad-hoc client/server and file transfer

A traditional listener handles one connection and exits. Plan for that, and
verify transfers with a hash rather than trusting the absence of errors.

| # | Command | How to use |
|---|---------|------------|
| 15 | `nc -l -p 4444 -v` | One-shot TCP listener |
| 16 | `nc -w 5 127.0.0.1 4444` | Connect to it |
| 17 | `nc -l -p 9000 > ~/nc-demo/evidence/received.bin` | Receive to a file |
| 18 | `nc -w 5 receiver.example 9000 < ~/nc-demo/payload.bin` | Send a file |
| 19 | `sha256sum ~/nc-demo/payload.bin ~/nc-demo/evidence/received.bin` | Verify integrity |
| 20 | `while true; do nc -l -p 4445; done` | Re-arm the listener |
| 21 | `nc -l -p 9001 -e /bin/cat` | Lab-only echo service |
| 22 | `ss -ltnp \| grep 4444` | Confirm the listener is bound |

**Research questions**
- Why does the listener exit after the first client, and what does `-k`
  actually do here?
- Is the file transfer reliable over a lossy link, and why not?
- What does `-e /bin/cat` expose to any client that connects?
- How do you keep a listener alive without a shell loop?
- What happens if the receiver starts after the sender?

**Sources:** `man nc`; `ss(8)`; TCP flow control; socat `fork` documentation.
**Output:** a transfer log with hashes and a listener-lifecycle note.

## Scenario 4 — speaking text protocols by hand

`nc` is at its best when you need to see exactly what a service sends. Compose
the request with `printf`, close with `-w` or `-q`, and capture the transcript.

| # | Command | How to use |
|---|---------|------------|
| 23 | `printf 'EHLO example\r\nQUIT\r\n' \| nc -w 5 target.example 25` | SMTP greeting |
| 24 | `printf 'GET / HTTP/1.1\r\nHost: target.example\r\nConnection: close\r\n\r\n' \| nc -w 5 target.example 80` | HTTP GET |
| 25 | `printf 'PING\r\n' \| nc -w 2 target.example 6379` | Redis PING |
| 26 | `printf 'STAT\r\n' \| nc -w 2 target.example 11211` | Memcached stats |
| 27 | `printf 'HELP\r\n' \| nc -w 2 target.example 21` | FTP help |
| 28 | `printf 'QUIT\r\n' \| nc -w 2 target.example 110` | POP3 quit |
| 29 | `printf 'GET / HTTP/1.0\r\n\r\n' \| nc -w 5 target.example 8080 \| head -40` | Limit output |

**Research questions**
- Which protocols require CRLF, and what happens with bare LF?
- When does the server close first, and when must `-q` force the close?
- Which of these commands would change state on the server?
- How do you distinguish a protocol error from a network timeout?
- Why is `head -40` risky on a binary or streaming response?

**Sources:** RFC 5321 (SMTP); RFC 9112 (HTTP/1.1); Redis and memcached protocol
documentation; RFC 959 (FTP); RFC 1939 (POP3).
**Output:** a protocol transcript per service with request, response, and
close behavior.

## Scenario 5 — UDP

UDP has no handshake and no EOF. A successful `nc -u` send proves only that
the local write succeeded.

| # | Command | How to use |
|---|---------|------------|
| 30 | `nc -u -l -p 5353 -v` | UDP listener |
| 31 | `printf 'hello\n' \| nc -u -w 2 receiver.example 5353` | UDP sender |
| 32 | `nc -u -l -p 5353 -o ~/nc-demo/evidence/udp.hex` | Hex-dump received datagrams |
| 33 | `nc -u -z -v target.example 53` | UDP port probe |
| 34 | `nc -u -b -l -p 5353` | Broadcast-enabled listener |
| 35 | `printf 'broadcast\n' \| nc -u -b -w 1 255.255.255.255 5353` | Broadcast sender |
| 36 | `nc -u -l -p 5353 -s 127.0.0.1` | Bind a specific source address |

**Research questions**
- What does `-b` change, and why is broadcast disabled by default?
- Why can a UDP listener require `-p` here when TCP does too?
- How do you confirm a datagram arrived rather than merely left?
- What is the difference between `-s` and `-p` in UDP mode?
- When should you use `socat UDP-RECVFROM` instead?

**Sources:** `man nc`; RFC 768; `socat` UDP address documentation; `tcpdump`
capture.
**Output:** a datagram send/receive log with a capture proving arrival.

## Scenario 6 — evidence, automation, and limits

Record what happened, bound the run time, and know when to switch tools. `nc`
has no TLS, no retry logic, and no structured output.

| # | Command | How to use |
|---|---------|------------|
| 37 | `nc -v -o ~/nc-demo/evidence/session.hex target.example 80` | Record an exchange |
| 38 | `xxd ~/nc-demo/evidence/session.hex \| head` | Inspect the dump |
| 39 | `printf 'HEAD / HTTP/1.0\r\n\r\n' \| timeout 5 nc -w 3 target.example 80` | Hard wall-clock bound |
| 40 | `nc -zv -w 1 target.example 80; echo "exit=$?"` | Capture the exit code |
| 41 | `tcpdump -i any -n port 4444 -c 5` | Independent packet evidence |
| 42 | `openssl s_client -connect target.example:443 -servername target.example` | TLS, which nc cannot do |
| 43 | `shred -u ~/nc-demo/payload.bin ~/nc-demo/payload.sha256` | Remove demo material |
| 44 | `rm -f ~/nc-demo/evidence/*.hex` | Remove captured payloads |

**Research questions**
- What exit codes does `nc` return for success, refusal, and timeout?
- Does `timeout` kill the socket cleanly, and what does the peer see?
- Which evidence is stronger: the nc transcript or a packet capture?
- When is `openssl s_client` or `socat OPENSSL` required instead of nc?
- What must be redacted from a captured protocol transcript?

**Sources:** `man nc`; `timeout(1)`; `tcpdump(8)`; `openssl-s_client(1)`.
**Output:** an evidence bundle with exit codes, a packet capture, and a
tool-selection note.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-l`, `-p` | 1, 3, 5 |
| `-u`, `-b`, `-s` | 2, 5 |
| `-z`, `-v`, `-n`, `-w`, `-i` | 2, 5, 6 |
| `-q`, `-k`, `-r` | 3, 6 |
| `-C`, `-t`, `-T` | 4 |
| `-o` | 2, 5, 6 |
| `-g`, `-G` | 6 (discussion) |
| `-e`, `-c` | 3 (lab only) |
| `-h` | 1 |

Port syntax: single, list, range (`lo-hi`), and escaped service names. Full
text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Identify the build | 6 |
| 2 | Connectivity and banners | 8 |
| 3 | Client/server and file transfer | 8 |
| 4 | Text protocols | 7 |
| 5 | UDP | 7 |
| 6 | Evidence, automation, limits | 8 |
| **Total** | | **44** |

## Legal and safety

`nc` reaches whatever the network allows and `-e`/`-c` turn a listener into a
shell. Use it only on systems you own or are authorized to test, bind
listeners to loopback or a lab interface, and never leave an exec listener
running. Captured protocol transcripts can contain credentials and tokens;
store them as secrets and delete them at engagement close.
