# whatweb — web technology fingerprinting

- **Version:** WhatWeb 0.5.5 (`whatweb --version`)
- **Binary:** `/usr/bin/whatweb` (Debian package `whatweb`)
- **Plugins:** `/usr/share/whatweb/plugins/` (~1,800 plugin files)
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/urbanadventurer/WhatWeb
- **Homepage:** https://www.morningstarsecurity.com/research/whatweb
- **Docs:** `man whatweb`

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

WhatWeb identifies what is running on a web endpoint. It sends HTTP requests,
matches responses against a large plugin library, and reports the content
management system, server, framework, JavaScript libraries, analytics,
embedded devices, and version numbers it can infer. It also records hashes,
titles, headers, cookies, and meta tags.

It is a **fingerprinter**, not a vulnerability scanner. A hit tells you the
technology and sometimes a version; it does not prove the version is
vulnerable. Use [`nuclei`](../nuclei/README.md) or manual testing for that.

## Use cases

| Situation | Why whatweb |
|-----------|-------------|
| First look at a web host | One request per target at default aggression |
| Bulk estate inventory | CIDR/range input and multiple threads |
| Confirm a CMS and version | `-a 3` plus the relevant plugin |
| Identify an embedded device | Default plugin set covers routers, cameras, NAS |
| Feed a technology list to other tools | `--log-json` or `--log-brief` |
| Targeted plugin research | `-l`, `-I`, `--search-plugins`, `--dorks` |
| Authenticated fingerprinting | `-u`, `-c`, `--cookie-jar` |

## Aggression levels

The aggression level is the main speed/stealth/reliability trade-off.

| Level | Name | Behavior |
|-------|------|----------|
| `1` | Stealthy (default) | One HTTP request per target, follows redirects |
| `3` | Aggressive | If a level-1 plugin matches, additional requests are made |
| `4` | Heavy | Many requests per target; all plugin URLs are attempted |

Level 2 is not described in this build's help. Treat `-a 3` and `-a 4` as
intrusive: they can generate dozens of requests per target.

## Flag reference

### Targets

| Flag | Meaning |
|------|---------|
| `<TARGETs>` | URLs, hostnames, IPs, CIDR, `x.x.x-x`, or `x.x.x.x-x.x.x.x` |
| `-i FILE`, `--input-file=FILE` | Read targets from a file; `-i /dev/stdin` for pipes |
| `--url-prefix` | Add a prefix to each target URL |
| `--url-suffix` | Add a suffix to each target URL |
| `--url-pattern` | Insert each target into a pattern (`%insert%`) |

### HTTP and authentication

| Flag | Meaning |
|------|---------|
| `-U AGENT`, `--user-agent=AGENT` | Custom user agent |
| `-H HEADER`, `--header=HEADER` | Add or replace a header |
| `--follow-redirect=WHEN` | `never`, `http-only`, `meta-only`, `same-site`, `always` (default) |
| `--max-redirects=NUM` | Redirect limit (default 10) |
| `-u USER:PASS`, `--user=USER:PASS` | HTTP basic authentication |
| `-c COOKIES`, `--cookie=COOKIES` | Cookie header |
| `--cookie-jar=FILE` | Read cookies from a file |
| `--proxy HOST[:PORT]` | Proxy (default port 8080) |
| `--proxy-user USER:PASS` | Proxy authentication |

### Plugin selection

| Flag | Meaning |
|------|---------|
| `-l`, `--list-plugins` | List all plugins |
| `-I[SEARCH]`, `--info-plugins[=SEARCH]` | Plugin details, optionally filtered |
| `--search-plugins=STRING` | Search plugin names and descriptions |
| `-p LIST`, `--plugins=LIST` | Select plugins (comma list, `+`/`-` modifiers) |
| `-g STRING\|REGEXP`, `--grep=STRING\|REGEXP` | Show only results matching a string or regex |
| `--custom-plugin=DEFINITION` | Define an inline plugin |
| `--dorks=PLUGIN` | List Google dorks for a plugin |

### Output and logging

| Flag | Meaning |
|------|---------|
| `-v`, `--verbose` | Verbose output with plugin descriptions; repeat for debug |
| `--colour=WHEN`, `--color=WHEN` | `never`, `always`, `auto` |
| `-q`, `--quiet` | Suppress brief logging to stdout |
| `--no-errors` | Suppress error messages |
| `--log-brief=FILE` | One-line output |
| `--log-verbose=FILE` | Verbose output |
| `--log-errors=FILE` | Errors only |
| `--log-xml=FILE` | XML |
| `--log-json=FILE` | JSON |
| `--log-json-verbose=FILE` | Verbose JSON |
| `--log-sql=FILE` | SQL `INSERT` statements |
| `--log-sql-create=FILE` | Create SQL tables |
| `--log-magictree=FILE` | MagicTree XML |
| `--log-object=FILE` | Ruby object inspection |
| `--log-mongo-*` | MongoDB output |
| `--log-elastic-*` | Elasticsearch output |

### Performance and help

| Flag | Meaning |
|------|---------|
| `-t NUM`, `--max-threads=NUM` | Simultaneous threads (default 25) |
| `--open-timeout=SEC` | Connection timeout (default 15) |
| `--read-timeout=SEC` | Read timeout (default 30) |
| `--wait=SEC` | Wait between connections |
| `--short-help` | Short usage |
| `--help`, `-h` | Full usage |
| `--debug` | Raise plugin errors |
| `--version` | Version |

## Recipes

**1. Single target.**

```bash
whatweb https://target.example/
```

**2. Verbose, with plugin descriptions.**

```bash
whatweb -v https://target.example/
```

**3. Aggressive CMS/version detection.**

```bash
whatweb -a 3 https://target.example/
```

**4. Bulk CIDR, suppress errors.**

```bash
whatweb --no-errors 192.168.10.0/24
```

**5. Only HTTPS targets in a range.**

```bash
whatweb --no-errors --url-prefix https:// 192.168.10.0/24
```

**6. Read targets from stdin.**

```bash
cat hosts.txt | whatweb -i /dev/stdin --log-json=~/whatweb-demo/results.json
```

**7. Select and search plugins.**

```bash
whatweb -l | grep -i wordpress
whatweb --search-plugins=wordpress
whatweb -I=wordpress
whatweb -p wordpress,php https://target.example/
```

**8. Grep results.**

```bash
whatweb -g 'WordPress|PHP' https://target.example/
```

**9. Authenticated and proxied.**

```bash
whatweb -u 'alice:Summer2026!' -c 'session=abc123' \
  --proxy 127.0.0.1:8080 https://target.example/
```

**10. Structured output for later processing.**

```bash
whatweb --log-json=~/whatweb-demo/results.json \
  --log-brief=~/whatweb-demo/results.txt https://target.example/
```

## Gotchas

- **`-p` selects plugins, not a port.** Use the URL for a non-default port.
- **`-g` filters results, not the request.** It is a reporting convenience.
- **`-a 3` and `-a 4` are intrusive.** They can multiply requests per target.
- **Default aggression is 1.** It makes one request per target and follows
  redirects; it will miss version details that need extra probes.
- **Version numbers are inferences.** Confirm a version with a second method
  before reporting it as fact.
- **`--log-json` is the stable machine format.** `--log-object` is Ruby-specific
  and not portable.
- **The plugin library is large (~1,800 files).** `-p` with a short list is much
  faster than the default set.
- **Redirects can leave scope.** `--follow-redirect=same-site` or `never`
  prevents a target from bouncing the scan to an unrelated host.
- **A WAF may return a generic block page.** That is a result about the WAF, not
  about the application stack.

## Related docs

- [`nuclei`](../nuclei/README.md) — template-based vulnerability checks
- [`httpx`](../httpx/README.md) — fast HTTP probing and tech detection
- [`nikto`](../nikto/README.md) — server misconfiguration scanning
- [`ffuf`](../ffuf/README.md) — content discovery after fingerprinting
- [`dirb`](../dirb/README.md) — directory scanning
