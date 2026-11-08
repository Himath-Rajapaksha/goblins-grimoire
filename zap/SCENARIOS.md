# zap — command scenarios for deep research

Every ZAP command line form in the installed build (`help.txt`), grouped by
scenario, with the API calls each mode unlocks.

## How to invoke

```text
@deep-research For ZAP 2.17, what is the minimal automation plan that gives a
credible baseline scan of an authenticated app, and which active-scan rules are
too noisy for production? Cite ZAP docs, automation framework docs, and
release notes; return an exact plan and launch command.
```

---

## Scenario 1 — GUI session management

| # | Command | How to use |
|---|---------|------------|
| 1 | `zap.sh` | Start the GUI with the default session |
| 2 | `zap.sh -session /path/engagement.session` | Open an existing session |
| 3 | `zap.sh -newsession /path/new.session` | Create a fresh session |
| 4 | `zap.sh -dir /path/zap-home` | Use a dedicated home directory |
| 5 | `zap.sh -installdir /opt/zap` | Override install detection |
| 6 | `zap.sh -lowmem` | Favor disk over memory (experimental) |
| 7 | `zap.sh -experimentaldb` | Use the experimental database backend |
| 8 | `zap.sh -nostdout` | Silence stdout logging |
| 9 | `zap.sh -loglevel DEBUG` | Override log level |
| 10 | `zap.sh -silent` | No unsolicited requests, including update checks |

**Research questions**
- Which session format changes between versions, and how are sessions migrated?
- Do `-lowmem`/`-experimentaldb` risk session corruption on large scans?

**Sources**: ZAP docs session/CLI, release notes, issues tagged lowmem.

**Output**: a documented session layout and safe flags for the installed build.

---

## Scenario 2 — daemon mode and the API

| # | Command | How to use |
|---|---------|------------|
| 11 | `zap.sh -daemon -host 127.0.0.1 -port 8080` | Headless proxy + API |
| 12 | `zap.sh -daemon -config api.key=SECRET` | Set the API key via config |
| 13 | `zap.sh -daemon -configfile zap.properties` | Load config from a file |
| 14 | `curl 'http://127.0.0.1:8080/JSON/core/view/version/?apikey=SECRET'` | Version check |
| 15 | `curl 'http://127.0.0.1:8080/JSON/spider/action/scan/?url=https://example.com&apikey=SECRET'` | Start spider |
| 16 | `curl 'http://127.0.0.1:8080/JSON/spider/view/status/?scanId=0&apikey=SECRET'` | Poll spider |
| 17 | `curl 'http://127.0.0.1:8080/JSON/ascan/action/scan/?url=https://example.com&apikey=SECRET'` | Start active scan |
| 18 | `curl 'http://127.0.0.1:8080/JSON/ascan/view/status/?scanId=0&apikey=SECRET'` | Poll active scan |
| 19 | `curl 'http://127.0.0.1:8080/JSON/core/view/alerts/?baseurl=https://example.com&apikey=SECRET'` | Fetch alerts as JSON |
| 20 | `curl -o report.html 'http://127.0.0.1:8080/OTHER/core/other/htmlreport/?apikey=SECRET'` | HTML report |
| 21 | `zap.sh -cmd -quickurl https://example.com -quickout report.html -quickprogress` | One-shot quick scan, no daemon |

**Research questions**
- Which API endpoints are stable across 2.x, and which are add-on dependent?
- How should the API key be supplied without putting it in shell history?

**Sources**: ZAP API docs, OpenAPI spec shipped with ZAP, automation docs.

**Output**: a scripted spider→scan→report loop with the key read from the
environment, not the command line.

---

## Scenario 3 — targeted reconnaissance

| # | Command | How to use |
|---|---------|------------|
| 22 | `zap.sh -cmd -zapit https://example.com` | Quick recon scan (`-cmd` required) |
| 23 | `zap.sh -cmd -quickurl https://example.com -quickout report.json` | Quick scan with JSON output |
| 24 | `zap.sh -cmd -quickurl https://example.com -quickout report.md` | Markdown output |
| 25 | `zap.sh -cmd -quickurl https://example.com -quickout report.xml` | XML output |

**Research questions**
- What does `-zapit` actually run versus `-quickurl`?
- Which quick-scan policies are safe against production apps?

**Sources**: ZAP quick start docs, CLI help, issues on quick scan scope.

**Output**: the cheapest credible first-pass report for a new target.

---

## Scenario 4 — automation framework

| # | Command | How to use |
|---|---------|------------|
| 26 | `zap.sh -cmd -autorun plan.yaml` | Run an automation plan |
| 27 | `zap.sh -cmd -autorun https://example.com/plan.yaml` | Run a remote plan |
| 28 | `zap.sh -cmd -autocheck plan.yaml` | Validate a plan without running |
| 29 | `zap.sh -cmd -autogenmin plan.yaml` | Generate a minimal template |
| 30 | `zap.sh -cmd -autogenmax plan.yaml` | Generate a template with all parameters |
| 31 | `zap.sh -cmd -autogenconf plan.yaml` | Generate from current configuration |

**Research questions**
- Which automation jobs cover auth, spider, passive, active, and report?
- How are plan secrets (credentials) referenced without hardcoding them?

**Sources**: ZAP automation framework docs, example plans, GitHub issues.

**Output**: a validated, versioned plan that can run unattended and fail loudly.

---

## Scenario 5 — importing API definitions

| # | Command | How to use |
|---|---------|------------|
| 32 | `zap.sh -openapifile api.yaml` | Import an OpenAPI definition from disk |
| 33 | `zap.sh -openapiurl https://example.com/openapi.json` | Import from URL |
| 34 | `zap.sh -openapitargeturl https://staging.example.com` | Override the server URL |
| 35 | `zap.sh -postmanfile collection.json` | Import a Postman collection |
| 36 | `zap.sh -postmanurl https://example.com/collection.json` | Import Postman from URL |
| 37 | `zap.sh -postmanendpointurl https://staging.example.com` | Override Postman base URL |
| 38 | `zap.sh -graphqlfile schema.graphql` | Import a GraphQL schema from disk |
| 39 | `zap.sh -graphqlurl https://example.com/graphql` | Import a GraphQL schema from URL |
| 40 | `zap.sh -graphqlendurl https://staging.example.com/graphql` | Set the GraphQL endpoint |

**Research questions**
- Which import path preserves auth headers and which drops them?
- How complete is GraphQL coverage compared with REST?

**Sources**: ZAP OpenAPI/Postman/GraphQL add-on docs, issues on import fidelity.

**Output**: the API surface imported with a verified endpoint override.

---

## Scenario 6 — add-ons, certificates, scripts

| # | Command | How to use |
|---|---------|------------|
| 41 | `zap.sh -addonlist` | List installed add-ons |
| 42 | `zap.sh -addoninstall <id>` | Install one add-on |
| 43 | `zap.sh -addoninstallall` | Install everything available |
| 44 | `zap.sh -addonupdate` | Update changed add-ons |
| 45 | `zap.sh -addonuninstall <id>` | Remove an add-on |
| 46 | `zap.sh -notel` | Disable telemetry |
| 47 | `zap.sh -certload /path/rootca.crt` | Load a custom root CA |
| 48 | `zap.sh -certpubdump /path/zap-pub.crt` | Export the public CA for browsers |
| 49 | `zap.sh -certfulldump /path/zap-full.crt` | Export CA including private key |
| 50 | `zap.sh -script script.js` | Run a script from CLI or load it in the GUI |
| 51 | `zap.sh -sbomzip /path/sboms.zip` | Bundle all available SBOMs |

**Research questions**
- Which add-ons are required for automation jobs used here?
- What is the blast radius of exposing `-certfulldump` output?

**Sources**: ZAP Marketplace pages, add-on help, cert docs.

**Output**: a minimal add-on set and a safely stored CA.

---

## Scenario 7 — HUD and support

| # | Command | How to use |
|---|---------|------------|
| 52 | `zap.sh -hud` | Launch a browser proxied through ZAP with HUD |
| 53 | `zap.sh -hudurl https://example.com` | HUD at a specific URL |
| 54 | `zap.sh -hudbrowser Chrome` | Choose Chrome or Firefox |
| 55 | `zap.sh -suppinfo` | Print support info and exit |
| 56 | `zap.sh -version` | Print the ZAP version |
| 57 | `zap.sh -h` / `-help` | Full option list including add-ons |

**Research questions**
- Does `-suppinfo` contain paths or data that must be redacted before sharing?
- Is the HUD maintained for the installed browser versions?

**Sources**: ZAP HUD docs, support guidance, release notes.

**Output**: a redacted support bundle and a working HUD launch.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Core | `-version`, `-cmd`, `-daemon`, `-config`, `-configfile`, `-dir`, `-installdir`, `-h`/`-help`, `-newsession`, `-session`, `-lowmem`, `-experimentaldb`, `-nostdout`, `-loglevel`, `-sbomzip`, `-suppinfo`, `-silent` | 1, 2, 7 |
| Add-ons | `-notel`, `-addoninstall`, `-addoninstallall`, `-addonuninstall`, `-addonupdate`, `-addonlist` | 6 |
| Quick scan | `-quickurl`, `-quickout`, `-quickprogress`, `-zapit` | 3 |
| Imports | `-postmanfile`, `-postmanurl`, `-postmanendpointurl`, `-graphqlfile`, `-graphqlurl`, `-graphqlendurl`, `-openapifile`, `-openapiurl`, `-openapitargeturl` | 5 |
| Automation | `-autorun`, `-autogenmin`, `-autogenmax`, `-autogenconf`, `-autocheck` | 4 |
| Scripts/certs | `-script`, `-certload`, `-certpubdump`, `-certfulldump` | 6 |
| Proxy/HUD | `-host`, `-port`, `-hud`, `-hudurl`, `-hudbrowser` | 2, 7 |

Full per-flag text: [`help.txt`](help.txt).
