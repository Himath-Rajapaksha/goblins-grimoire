# dirb — web content and directory scanner

- **Version:** DIRB v2.22 (`dirb` with no arguments)
- **Binary:** `/usr/bin/dirb` (Debian package `dirb`)
- **Wordlists:** `/usr/share/dirb/wordlists/` (`common.txt` is the built-in
  default)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://dirb.sourceforge.net/
- **Docs:** `man dirb`; `/usr/share/doc/dirb/README.txt`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

dirb is a recursive web content scanner. It takes a base URL and one or more
wordlists, requests every candidate path, and reports responses that are not
the site's "not found" response. It classifies results by status code, follows
directories it finds, and can append file extensions.

It is older and slower than [`ffuf`](../ffuf/README.md) or
[`gobuster`](../gobuster/README.md), but it has a few properties those tools do
not: built-in recursive scanning, interactive hotkeys, session resume, and a
shipped wordlist set. It is useful when you want a quick, no-configuration
sweep or when a target's soft-404 behavior needs dirb's `-f` tuning.

## Use cases

| Situation | Why dirb |
|-----------|----------|
| Quick content sweep on a new host | Default `common.txt`, no flags |
| Recursive directory discovery | Recursion is on by default |
| Extension sweep (`.php`, `.bak`, `.old`) | `-X .php,.bak,.old` |
| Soft-404 or wildcard responses | `-f`, `-N`, `-v` |
| Authenticated area | `-u`, `-c`, `-H` |
| Slow or monitored target | `-z` delay, `-p` proxy |
| Long scan that may be interrupted | `q` hotkey saves state, `-resume` restarts |

## Flag reference

### Request and target

| Flag | Meaning |
|------|---------|
| `-a <agent>` | Custom `User-Agent` |
| `-b` | Use the path as-is (do not normalize) |
| `-c <cookie>` | Cookie header |
| `-E <cert>` | Client certificate |
| `-H <header>` | Extra HTTP header |
| `-u <user:pass>` | HTTP basic authentication |
| `-p <proxy[:port]>` | Proxy (default port 1080) |
| `-P <user:pass>` | Proxy authentication |

### Discovery behavior

| Flag | Meaning |
|------|---------|
| `-X <exts>` | Append each word with these extensions |
| `-x <file>` | Read extensions from a file |
| `-i` | Case-insensitive search |
| `-l` | Print the `Location` header when found |
| `-N <code>` | Ignore responses with this HTTP code |
| `-r` | Do not search recursively |
| `-R` | Interactive recursion (ask per directory) |
| `-t` | Do not force a trailing `/` on URLs |

### Noise, output, and pacing

| Flag | Meaning |
|------|---------|
| `-f` | Fine-tune "not found" (404) detection |
| `-v` | Also show NOT_FOUND pages |
| `-S` | Silent mode; do not show tested words |
| `-w` | Do not stop on WARNING messages |
| `-o <file>` | Save output to disk |
| `-z <ms>` | Delay in milliseconds between requests |

## Hotkeys

While a scan is running:

| Key | Action |
|-----|--------|
| `n` | Go to the next directory |
| `q` | Stop and save state for resume |
| `r` | Show remaining scan statistics |

## Recipes

**1. Default scan.**

```bash
dirb https://target.example/
```

**2. Explicit wordlist and extensions.**

```bash
dirb https://target.example/ /usr/share/dirb/wordlists/common.txt -X .php,.bak,.old
```

**3. Multiple wordlists.**

```bash
dirb https://target.example/ \
  /usr/share/dirb/wordlists/common.txt,/usr/share/dirb/wordlists/vulns/apache.txt
```

**4. Authenticated scan with a cookie and header.**

```bash
dirb https://target.example/ -u 'alice:Summer2026!' \
  -c 'session=abc123' -H 'X-Forwarded-For: 127.0.0.1'
```

**5. Slow scan through a proxy.**

```bash
dirb https://target.example/ -p 127.0.0.1:8080 -z 250 -S -o ~/dirb-demo/scan.txt
```

**6. Ignore noisy response codes.**

```bash
dirb https://target.example/ -N 403 -N 429
```

**7. No recursion, no trailing slash.**

```bash
dirb https://target.example/ -r -t
```

**8. Resume an interrupted scan.**

```bash
dirb -resume
```

## Gotchas

- **No `--help`.** Running `dirb` with no arguments prints the usage block;
  there is no help flag.
- **The default wordlist is `common.txt`.** It is small; a real engagement
  should choose a wordlist deliberately.
- **Recursion is on by default.** On a large site this can explode the request
  count. Use `-r` unless recursion is intended.
- **`-X` and `-x` are different.** `-X` takes a comma list, `-x` takes a file.
- **`-f` is not "force".** It fine-tunes 404 detection; use `-v` to see what is
  being classified as not found.
- **`-N` takes one code per flag.** Repeat it for multiple codes.
- **`-resume` is a pseudo-flag**, not listed in the options table. It restarts
  from the state saved by the `q` hotkey.
- **dirb is not a fuzzer.** It substitutes whole paths, not arbitrary positions
  in a request. Use `ffuf` for parameters, headers, and JSON bodies.
- **Results are only as good as the wordlist.** A 200 response may be a custom
  error page; verify interesting hits manually.

## Related docs

- [`ffuf`](../ffuf/README.md) — faster, position-based fuzzing
- [`gobuster`](../gobuster/README.md) — dir/DNS/vhost/fuzz modes
- [`dirsearch`](../dirsearch/README.md) — another recursive content scanner
- [`wfuzz`](../wfuzz/README.md) — general-purpose web fuzzer
- [`whatweb`](../whatweb/README.md) — identify the stack before choosing paths
