# sqlmap — automatic SQL injection detection and exploitation

- **Version:** 1.9.6#stable (`sqlmap --version`; the installed build prints an
  "outdated" warning, so verify before relying on a recent DBMS feature)
- **Binary:** `/usr/bin/sqlmap` (Debian package `sqlmap`)
- **Help snapshots:** [`help.txt`](help.txt) (basic), [`help-advanced.txt`](help-advanced.txt) (`-hh`)
- **Upstream:** https://github.com/sqlmapproject/sqlmap
- **Docs:** https://github.com/sqlmapproject/sqlmap/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

sqlmap automates the detection and exploitation of SQL injection. It parses a
request (URL, POST body, cookie, header, or a saved Burp request), injects
payloads for the selected DBMS, fingerprints the back end, and then enumerates
or extracts data. With sufficient privilege it can read/write files, run OS
commands, or open an interactive SQL shell.

It is a high-impact tool: a `--dump` can copy an entire database, and
`--os-shell` can execute commands on the database host. Treat every run as an
intrusive test and scope it explicitly.

## Use cases

| Situation | Why sqlmap |
|-----------|------------|
| Confirmed or suspected SQLi | Automate detection across parameters and techniques |
| Authenticated testing | Replay a logged-in request with cookies/headers |
| Burp workflow | Save a request to a file and use `-r` |
| Data-exposure proof | Enumerate schemas and dump a bounded table |
| Privilege assessment | Check `--current-user`, `--is-dba`, file read/write |
| WAF testing | `--tamper` scripts and request pacing |
| Regression check | Re-run a saved session against a fixed build |

## Requirements & install layout

- Debian package `sqlmap`; binary `/usr/bin/sqlmap`. Python 3.
- No external DB driver is needed for HTTP injection; `-d` (direct DB
  connection) needs the relevant client library.
- Session data is stored under `~/.local/share/sqlmap/output/<host>/` by default;
  `--output-dir` changes it. `--flush-session` clears the target's cache.
- `--tor` expects a Tor SOCKS proxy on `127.0.0.1:9050` by default; override
  with `--tor-port`/`--tor-type`.

## Basic syntax

```bash
sqlmap -u 'http://target/item.php?id=1' --batch
sqlmap -u 'http://target/login' --data 'user=u&pass=p' --batch
sqlmap -r request.txt --batch --level 3 --risk 2
sqlmap -u 'http://target/item.php?id=1' --dbs
```

`--batch` accepts defaults and never prompts; use it for reproducible runs and
CI. Without it, sqlmap asks before each phase.

## Flag reference

### Target

| Flag | Meaning |
|------|---------|
| `-u <url>` | Target URL |
| `-d <dsn>` | Direct database connection string |
| `-l <log>` | Parse targets from a Burp/WebScarab log |
| `-m <file>` | Scan multiple targets from a file |
| `-r <file>` | Load a raw HTTP request from a file |
| `-g <dork>` | Use Google dork results as targets |
| `-c <ini>` | Load options from a configuration file |

### Request

| Flag | Meaning |
|------|---------|
| `--data <data>` | POST body |
| `--method <verb>` | Force an HTTP method (PUT, PATCH, ...) |
| `--cookie <cookie>` | Cookie header |
| `--cookie-del <char>` | Cookie separator |
| `--load-cookies <file>` | Netscape/wget cookie file |
| `--live-cookies <file>` | Reload cookies before each request |
| `-A <ua>` / `--random-agent` | User-Agent / random UA |
| `-H <header>` | Extra header (repeatable) |
| `--headers <headers>` | Multiple headers, newline-separated |
| `--host <host>` | Host header |
| `--referer <url>` | Referer header |
| `--auth-type <type>` | Basic, Digest, Bearer, ... |
| `--auth-cred <user:pass>` | Auth credentials |
| `--proxy <url>` | HTTP/SOCKS proxy |
| `--proxy-cred <user:pass>` | Proxy credentials |
| `--proxy-file <file>` | Proxy list |
| `--tor` | Route through Tor |
| `--check-tor` | Verify Tor is working |
| `--delay <s>` | Delay between requests |
| `--timeout <s>` | Request timeout (default 30) |
| `--retries <n>` | Retries on timeout (default 3) |
| `--force-ssl` | Force HTTPS |
| `--chunked` | Chunked transfer encoding |
| `--hpp` | HTTP parameter pollution |
| `--csrf-token <param>` | Anti-CSRF token parameter |
| `--csrf-url <url>` | Page that issues the token |
| `--eval <python>` | Evaluate Python before the request |

### Injection and detection

| Flag | Meaning |
|------|---------|
| `-p <params>` | Test only these parameters |
| `--skip <params>` | Skip these parameters |
| `--skip-static` | Skip non-dynamic parameters |
| `--param-exclude <regex>` | Exclude parameters by regex |
| `--dbms <name>` | Force the back-end DBMS |
| `--os <name>` | Force the back-end OS |
| `--level <1-5>` | Test breadth (default 1) |
| `--risk <1-3>` | Payload risk (default 1) |
| `--technique <BEUSTQ>` | Boolean, Error, Union, Stacked, Time, inline Query |
| `--time-sec <s>` | Time-based delay (default 5) |
| `--union-cols <n>` | Union column count |
| `--union-char <char>` | Union filler character |
| `--prefix <s>` / `--suffix <s>` | Payload wrappers |
| `--tamper <script>` | Tamper script(s), comma-separated |
| `--parse-errors` | Parse DBMS error messages |
| `--forms` | Parse and test HTML forms |
| `--crawl <depth>` | Crawl the site for targets |

### Enumeration and extraction

| Flag | Meaning |
|------|---------|
| `-a` / `--all` | Retrieve everything |
| `-b` / `--banner` | DBMS banner |
| `--current-user` | Current DBMS user |
| `--current-db` | Current database |
| `--is-dba` | Check DBA privileges |
| `--users` | Enumerate DBMS users |
| `--passwords` | Enumerate password hashes |
| `--privileges` | Enumerate privileges |
| `--roles` | Enumerate roles |
| `--dbs` | Enumerate databases |
| `--tables` | Enumerate tables |
| `--columns` | Enumerate columns |
| `--schema` | Enumerate schema |
| `--dump` | Dump table entries |
| `--dump-all` | Dump all databases |
| `-D <db>` / `-T <table>` / `-C <cols>` | Scope the dump |
| `--exclude-sysdbs` | Skip system databases |
| `--where <clause>` | Add a WHERE clause |
| `--start <n>` / `--stop <n>` | Row range |
| `--sql-query <sql>` | Run a custom query |
| `--sql-shell` | Interactive SQL shell |
| `--file-read <path>` | Read a file from the DB host |
| `--file-write <local>` | Write a local file to the DB host |
| `--file-dest <path>` | Destination for `--file-write` |
| `--os-shell` | Interactive OS shell |
| `--os-pwn` | OOB shell / Meterpreter / VNC |
| `--os-cmd <cmd>` | Run one OS command |

### General and output

| Flag | Meaning |
|------|---------|
| `--batch` | Never ask; use defaults |
| `--flush-session` | Clear cached session data for the target |
| `--output-dir <dir>` | Directory for logs and dumps |
| `--har <file>` | Save all HTTP traffic as HAR |
| `-v <0-6>` | Verbosity |
| `--wizard` | Beginner wizard |
| `--answers <answers>` | Scripted answers to prompts |
| `--disable-coloring` | Plain output |
| `--offline` | Use only cached data |

## Recipes

Authorized targets only.

**1. Basic GET parameter test.**

```bash
sqlmap -u 'https://target.example/item.php?id=1' --batch --level 2 --risk 2
```

**2. Authenticated POST with a cookie and CSRF token.**

```bash
sqlmap -u 'https://target.example/account' \
  --data 'id=1&csrf=TOKEN' --cookie 'SESSION=abc123' --batch
```

**3. Replay a saved Burp request.** This preserves method, headers, cookies,
and body exactly.

```bash
sqlmap -r ~/engagements/acme/request.txt --batch --level 3 --risk 2
```

**4. Enumerate and dump one table.**

```bash
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  --dbs
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  -D appdb --tables
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  -D appdb -T users -C username,password --dump
```

**5. Check privilege and file access.**

```bash
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  --current-user --is-dba --privileges
```

**6. WAF-paced run with a tamper script.**

```bash
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  --tamper=space2comment --delay 1 --random-agent --level 3 --risk 2
```

**7. Route through Burp for manual inspection.**

```bash
sqlmap -u 'https://target.example/item.php?id=1' --proxy http://127.0.0.1:8080
```

**8. Save evidence.** `--output-dir` keeps logs and dumped data in the
engagement directory; `--har` records the HTTP traffic.

```bash
sqlmap -u 'https://target.example/item.php?id=1' --batch \
  --output-dir ~/engagements/acme/sqlmap --har ~/engagements/acme/sqlmap/traffic.har
```

## Output & pipelines

- sqlmap prints a per-parameter verdict (`parameter 'id' is vulnerable`), the
  injection type, payload, and DBMS.
- Dumped rows are written under the output directory; `--dump` also prints a
  table to stdout.
- `--har` is the cleanest way to hand the exact requests to a report or another
  tool.
- `--sql-query` returns raw rows; use `--csv-del`/`--dump-format` for
  machine-readable output.
- Exit status is not a reliable vulnerability signal; parse the log for
  `is vulnerable` / `injection` lines.

## Gotchas & operational security

- **`--risk 3` can change data.** Time-based and stacked payloads may update or
  delete rows; use the lowest risk that answers the question.
- **`--level 5` is slow and loud.** It tests headers, cookies, and user agents;
  raise it only when lower levels are inconclusive.
- **`--os-shell` writes files and runs commands.** It is a post-exploitation
  action, not a detection check; confirm authorization first.
- **`--dump-all` can exfiltrate the whole server.** Bound it with `-D`, `-T`,
  `--where`, `--start`, and `--stop`.
- **Session caching hides changes.** If the target was fixed, `--flush-session`
  is required to re-test.
- **`--tamper` scripts are DBMS- and WAF-specific.** Stacking incompatible
  tampers can break payloads; test one at a time.
- **Tor is not anonymity for the target.** The target still sees the request;
  Tor only changes the source network. Do not use it to evade authorization.
- **Dumps contain personal data.** Encrypt or delete them at engagement close.

## Legal & safety

Use sqlmap only on applications you own or have **explicit written
authorization** to test. SQL injection testing can read, modify, or delete data
and can lead to full host compromise. Do not use `--os-shell`, `--file-write`,
or `--dump-all` unless the rules of engagement specifically permit it. Stop and
report if you reach data outside the agreed scope.

## Related tools

- [`burp/`](../burp/README.md) — capture and replay the request sqlmap consumes.
- [`ffuf/`](../ffuf/README.md) — find the parameters and endpoints worth testing.
- [`nuclei/`](../nuclei/README.md) — broad template checks before targeted SQLi.
- [`wfuzz/`](../wfuzz/README.md) — manual payload fuzzing for comparison.
- [`mitmproxy/`](../mitmproxy/README.md) — scriptable request/response inspection.

## References

- Installed help snapshots: [`help.txt`](help.txt), [`help-advanced.txt`](help-advanced.txt)
- Upstream wiki: https://github.com/sqlmapproject/sqlmap/wiki
- Usage docs: https://github.com/sqlmapproject/sqlmap/wiki/Usage
- Kali tool page: https://www.kali.org/tools/sqlmap/
