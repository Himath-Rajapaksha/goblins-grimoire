# CeWL — command scenarios for deep research

Installed build: CeWL 6.2.1. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For CeWL 6.2.1 in 2026, how effective are site-derived
wordlists for password guessing compared with generic lists plus mangling
rules, which depth/min-length/group settings produce the best signal, and how
should metadata and email extraction be scoped lawfully? Cite CeWL docs,
password-research literature, and field reports; return a tuned extraction
profile and a rule-based combination strategy.
```

---

## Scenario 1 — basic extraction

| # | Command | How to use |
|---|---------|------------|
| 1 | `cewl https://target/` | Print words |
| 2 | `cewl -w words.txt https://target/` | Save wordlist |
| 3 | `cewl -m 5 -x 20 -w words.txt https://target/` | Length bounds |
| 4 | `cewl --lowercase --with-numbers -w words.txt https://target/` | Normalise |
| 5 | `cewl -c --no-words https://target/ \| sort -rn \| head` | Top words |

**Research questions**
- What tokenisation rules does CeWL apply (hyphens, punctuation, Unicode)?
- Does `--lowercase` run before or after deduplication?

**Sources:** source code, output inspection.
**Output:** a clean, deduplicated target wordlist.

---

## Scenario 2 — spider control

| # | Command | How to use |
|---|---------|------------|
| 6 | `cewl -d 1 -w words.txt https://target/` | Shallow crawl |
| 7 | `cewl -d 4 -w words.txt https://target/` | Deep crawl |
| 8 | `cewl --allowed '/blog/.*' -w words.txt https://target/` | Scoped paths |
| 9 | `cewl --exclude excludes.txt -w words.txt https://target/` | Exclude paths |
| 10 | `cewl -o -w words.txt https://target/` | Follow offsite |

**Research questions**
- How does depth interact with duplicate pages and query strings?
- What is the request footprint per depth level?

**Sources:** docs, target logs (lab), source.
**Output:** a scoped crawl that avoids third-party leakage.

---

## Scenario 3 — metadata and email

| # | Command | How to use |
|---|---------|------------|
| 11 | `cewl -e --email_file emails.txt https://target/` | Collect emails |
| 12 | `cewl -a --meta_file meta.txt https://target/` | Collect metadata |
| 13 | `cewl -n -a --meta_file meta.txt https://target/` | Metadata only |
| 14 | `cewl -e -a --email_file e.txt --meta_file m.txt -w w.txt https://target/` | All outputs |
| 15 | `exiftool -r -Author -Creator meta.txt` | Review authors |

**Research questions**
- Which metadata fields does `-a` capture, and from which file types?
- Does `-e` find obfuscated addresses (`[at]`, entities)?

**Sources:** exiftool output, docs, sample pages.
**Output:** email and author intelligence.

---

## Scenario 4 — authenticated and proxied crawls

| # | Command | How to use |
|---|---------|------------|
| 16 | `cewl --auth_type basic --auth_user alice --auth_pass 'x' -w w.txt https://target/` | Basic auth |
| 17 | `cewl --auth_type digest --auth_user alice --auth_pass 'x' -w w.txt https://target/` | Digest auth |
| 18 | `cewl --proxy_host 127.0.0.1 --proxy_port 8080 -w w.txt https://target/` | Proxy crawl |
| 19 | `cewl -H "Authorization: Bearer <token>" -w w.txt https://target/` | Header auth |
| 20 | `cewl -u "Mozilla/5.0" -w w.txt https://target/` | Custom UA |

**Research questions**
- Does cookie-based auth need `-H "Cookie: ..."` rather than the auth flags?
- How does the proxy handle TLS interception errors?

**Sources:** docs, lab proxy, source.
**Output:** words from protected content.

---

## Scenario 5 — wordlist quality and handoff

| # | Command | How to use |
|---|---------|------------|
| 21 | `cewl -g 2 -w groups.txt https://target/` | Two-word groups |
| 22 | `cat words.txt rockyou.txt \| sort -u > combined.txt` | Merge lists |
| 23 | `wc -l words.txt combined.txt` | Compare sizes |
| 24 | `hashcat --stdout -r rules/best64.rule words.txt > mangled.txt` | Apply rules |
| 25 | `sha256sum words.txt combined.txt` | Hash evidence |

**Research questions**
- Do word groups improve passphrase coverage enough to justify size?
- Which mangling rules best complement site vocabulary?

**Sources:** hashcat docs, cracking benchmarks, engagement results.
**Output:** a ranked, mangled wordlist ready for authorized testing.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Extraction | `-d -m -x -w -n -g --lowercase --with-numbers -c` | 1, 5 |
| Scope | `-o --exclude --allowed -k` | 2 |
| Metadata | `-a --meta_file -e --email_file` | 3 |
| Request | `-u --auth_type --auth_user --auth_pass --proxy_* -H` | 4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — basic | 5 |
| 2 — spider control | 5 |
| 3 — metadata | 5 |
| 4 — auth/proxy | 5 |
| 5 — quality | 5 |
