# OWASP ZAP — web app scanner & intercepting proxy

- **Version:** 2.17.0 (installed build)
- **Launcher:** `/usr/local/bin/zap.sh`
- **Type:** GUI + daemon + CLI + REST API
- **Docs:** https://www.zaproxy.org/docs/
- **Getting started:** https://www.zaproxy.org/getting-started/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

ZAP is a full-featured open-source web security platform: intercepting proxy,
spider, AJAX spider, passive scanner, active scanner, fuzzer, authentication
handling, automation framework, and a REST API. Unlike Burp Community, ZAP
includes automated scanning for free.

## Use cases

| Situation | Mode |
|-----------|------|
| Manual proxy testing | GUI |
| Automated baseline scan | `-cmd -quickurl ... -quickout ...` |
| CI/CD scanning | Automation Framework YAML (`-autorun`) |
| Headless/daemon API | `-daemon` + REST API |
| Authenticated scans | Context + auth config in GUI or automation |
| API/OpenAPI testing | `-openapifile` / `-openapiurl` |
| Postman collections | `-postmanfile` |
| Fuzzing | GUI Fuzzer / Active Scan |

## Launch options

```bash
zap.sh                 # GUI
zap.sh -daemon         # headless daemon
zap.sh -cmd ...        # run CLI options then exit
zap.sh -version
zap.sh -h              # all options incl. add-ons
```

### Core options

| Flag | Meaning |
|------|---------|
| `-version` | Print version |
| `-cmd` | Inline run; exits when options complete |
| `-daemon` | Headless, no UI |
| `-config <k=v>` | Override a config key |
| `-configfile <path>` | Load config from properties file |
| `-dir <dir>` | Home/data directory |
| `-installdir <dir>` | Install directory override |
| `-newsession <path>` | New session at path |
| `-session <path>` | Open existing session |
| `-lowmem` | Reduce memory use (DB-backed, experimental) |
| `-experimentaldb` | Experimental DB code |
| `-nostdout` | Disable stdout logging |
| `-loglevel <level>` | Log level override |
| `-silent` | No unsolicited requests/update checks |
| `-sbomzip <path>` | Export SBOMs |

### Quick scan / imports

| Flag | Meaning |
|------|---------|
| `-quickurl <url>` | Target for quick scan |
| `-quickout <file>` | Report file (format by extension: HTML/JSON/MD/XML) |
| `-quickprogress` | Progress bars |
| `-zapit <url>` | Quick recon scan (needs `-cmd`) |
| `-openapifile <path>` / `-openapiurl <url>` | Import OpenAPI |
| `-openapitargeturl <url>` | Override server URL |
| `-postmanfile <path>` / `-postmanurl <url>` | Import Postman collection |
| `-postmanendpointurl <url>` | Override base URL |
| `-graphqlfile <path>` / `-graphqlurl <url>` | Import GraphQL schema |
| `-graphqlendurl <url>` | GraphQL endpoint |

### Automation

| Flag | Meaning |
|------|---------|
| `-autorun <file-or-url>` | Run automation plan |
| `-autocheck <file-or-url>` | Validate automation plan |
| `-autogenmin <file>` | Generate minimal automation YAML |
| `-autogenmax <file>` | Generate full automation YAML |
| `-autogenconf <file>` | Generate from current config |
| `-script <script>` | Run a script |

### Proxy / browser / certs

| Flag | Meaning |
|------|---------|
| `-host <host>` / `-port <port>` | Main proxy bind |
| `-certload <path>` | Load root CA |
| `-certpubdump <path>` | Dump public CA cert |
| `-certfulldump <path>` | Dump full CA (with key) |
| `-hud` | Launch browser with HUD |
| `-hudurl <url>` / `-hudbrowser <browser>` | HUD target/browser |

### Add-ons

`-addoninstall <id>`, `-addoninstallall`, `-addonuninstall <id>`,
`-addonupdate`, `-addonlist`, `-notel` (disable telemetry).

## Recipes

### GUI quick start

```bash
zap.sh
# 1. Tools → Options → Local Proxy → confirm 127.0.0.1:8080
# 2. Set browser proxy to 127.0.0.1:8080
# 3. Import ZAP CA cert (Options → Dynamic SSL Certs → Save)
# 4. Enter URL in the Quick Start tab → Attack
```

### Headless quick scan (CI-friendly)

```bash
zap.sh -cmd -quickurl https://example.com -quickout /tmp/zap-report.html -quickprogress
```

Report format is chosen by extension: `.html`, `.json`, `.md`, `.xml`.

### Daemon + API

```bash
zap.sh -daemon -host 127.0.0.1 -port 8080 \
  -config api.key=change-me -config api.addrs.addr.name=.* \
  -config api.addrs.addr.regex=true
```

Then:

```bash
# Spider
curl 'http://127.0.0.1:8080/JSON/spider/action/scan/?url=https://example.com&apikey=change-me'

# Active scan
curl 'http://127.0.0.1:8080/JSON/ascan/action/scan/?url=https://example.com&apikey=change-me'

# Alerts
curl 'http://127.0.0.1:8080/JSON/alert/view/alerts/?baseurl=https://example.com&apikey=change-me'
```

### Automation Framework

Generate a starting plan:

```bash
zap.sh -cmd -autogenmin plan.yaml
```

Minimal plan shape:

```yaml
env:
  contexts:
    - name: target
      urls: ["https://example.com"]
  parameters:
    failOnError: true

jobs:
  - type: spider
    parameters:
      context: target
      maxDuration: 5
  - type: passiveScan-wait
  - type: activeScan
    parameters:
      context: target
  - type: report
    parameters:
      template: traditional-html
      reportDir: /tmp/zap-reports
      reportFile: report
```

Run:

```bash
zap.sh -cmd -autorun plan.yaml
```

### OpenAPI / Postman

```bash
zap.sh -cmd -openapifile api.yaml -openapitargeturl https://api.example.com
zap.sh -cmd -postmanfile collection.json -postmanendpointurl https://api.example.com
```

## Manual testing workflow

1. **Sites tree** → browse target through proxy.
2. **Spider** (traditional) or **Ajax Spider** (JS-heavy).
3. Review **Alerts** (passive findings appear as you browse).
4. Right-click request → **Attack → Active Scan** on selected node.
5. Use **Repeater** / **Manual Request Editor** for custom requests.
6. **Fuzzer** for payload positions.
7. Export report: **Report → Generate Report**.

## Important scan modes

| Mode | Behavior |
|------|----------|
| Safe | Passive only; no active attacks |
| Protected | Active attacks but no dangerous payloads |
| Standard | Normal active scanning |
| ATTACK | Full aggressive scanning (may be destructive) |

Set via the mode selector in the toolbar. Use **Safe** or **Protected** on
production.

## Gotchas

- ZAP's active scan can be destructive; confirm authorization and mode.
- The AJAX spider needs a real browser (Firefox/Chrome) and is slower.
- Authentication is the hard part: configure a context with login URL,
  credentials, and verification regex before scanning authenticated areas.
- `-quickurl` performs a baseline (passive + spider) scan, not full active.
- The API key must be set when binding beyond localhost; never expose it.
- ZAP stores sessions; use `-newsession` for clean CI runs.
- HUD changes the browsing experience; disable it if it confuses testing.

## References

- Docs: https://www.zaproxy.org/docs/
- Automation Framework: https://www.zaproxy.org/docs/automate/automation-framework/
- API: https://www.zaproxy.org/docs/api/
- Getting started: https://www.zaproxy.org/getting-started/
