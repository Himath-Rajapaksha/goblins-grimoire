# CMSeeK — command scenarios for deep research

Installed build: CMSeeK 1.1.3. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For CMSeeK 1.1.3 in 2026, which CMS detection signatures are
still accurate for WordPress, Joomla, and Drupal, which modules produce the
most false positives, and how should version findings be verified before CVE
mapping? Cite the CMSeeK repo, CMS release docs, and WAF behaviour; return a
reliability table and a verified scan workflow.
```

---

## Scenario 1 — CMS identification

| # | Command | How to use |
|---|---------|------------|
| 1 | `cmseek -u https://target/ --only-cms` | CMS only |
| 2 | `cmseek -u https://target/ --light-scan` | CMS + version |
| 3 | `cmseek -u https://target/ -v` | Full scan |
| 4 | `cmseek -u https://target/ --no-redirect` | No redirects |
| 5 | `cmseek --version` | Record version |

**Research questions**
- Which signatures identify each CMS most reliably?
- How does redirect handling change detection?

**Sources:** modules, target responses.
**Output:** CMS name and version.

---

## Scenario 2 — components and users

| # | Command | How to use |
|---|---------|------------|
| 6 | `cmseek -u https://target/ -v` | Deep scan |
| 7 | `cmseek -u https://target/ --light-scan` | Skip deep enumeration |
| 8 | `cmseek -u https://target/ -i 1` | Ignore WordPress |
| 9 | `cmseek -u https://target/ --strict-cms 1` | WordPress only |
| 10 | `ls /opt/cmseek/Result/` | Inspect results |

**Research questions**
- Which component enumerators work without authentication?
- How are detected plugins mapped to versions?

**Sources:** module source, output, CMS docs.
**Output:** a component inventory for vuln mapping.

---

## Scenario 3 — batch scanning

| # | Command | How to use |
|---|---------|------------|
| 11 | `cmseek -l targets.txt --batch` | Batch, no prompts |
| 12 | `cmseek -l targets.txt --batch -v` | Verbose batch |
| 13 | `cmseek -l targets.txt --skip-scanned` | Resume |
| 14 | `cmseek -l targets.txt --batch -r` | Random UA |
| 15 | `cmseek -l targets.txt --batch > batch.log` | Log output |

**Research questions**
- How does `--skip-scanned` decide what to skip?
- What pacing avoids WAF blocks during batches?

**Sources:** source, logs, target behaviour.
**Output:** a technology map across many hosts.

---

## Scenario 4 — evasion and reliability

| # | Command | How to use |
|---|---------|------------|
| 16 | `cmseek -u https://target/ -r` | Random UA |
| 17 | `cmseek -u https://target/ --googlebot` | Googlebot UA |
| 18 | `cmseek -u https://target/ --user-agent "Mozilla/5.0"` | Custom UA |
| 19 | `cmseek -u https://target/ --follow-redirect` | Follow redirects |
| 20 | `curl -sI https://target/ \| head -20` | Manual header check |

**Research questions**
- Which UA strings bypass common WAF rules?
- Does `--googlebot` actually change detection or only the header?

**Sources:** WAF docs, response diff, source.
**Output:** consistent detection through a WAF.

---

## Scenario 5 — evidence and CVE mapping

| # | Command | How to use |
|---|---------|------------|
| 21 | `cmseek -u https://target/ -v > scan.log` | Capture report |
| 22 | `grep -Ei 'version\|plugin\|theme' scan.log` | Extract versions |
| 23 | `searchsploit wordpress <version>` | Local exploit lookup |
| 24 | `nuclei -u https://target/ -tags wordpress` | Template validation |
| 25 | `sha256sum scan.log` | Hash evidence |

**Research questions**
- Which CVE sources are current enough for CMS plugin versions?
- How should false version matches be excluded?

**Sources:** CVE databases, nuclei templates, output.
**Output:** a verified CMS vulnerability shortlist.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Target | `-u -l` | 1, 3 |
| Scan | `-i --strict-cms --skip-scanned --light-scan -o` | 1, 2 |
| Request | `--follow-redirect --no-redirect -r --googlebot --user-agent` | 4 |
| Output | `-v --batch` | 3–5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — identification | 5 |
| 2 — components | 5 |
| 3 — batch | 5 |
| 4 — evasion | 5 |
| 5 — CVE mapping | 5 |
