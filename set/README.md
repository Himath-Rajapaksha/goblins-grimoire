# SET — Social-Engineer Toolkit

- **Version:** 8.1.3
- **Launcher:** `/usr/local/bin/setoolkit`
- **Install prefix:** `/opt/set` (git checkout plus a Python 3.13.5 virtualenv)
- **Configuration:** `/etc/setoolkit/set.config` and `/etc/setoolkit/set_config.py`
- **User state:** `~/.set`
- **Help snapshot:** [`help.txt`](help.txt) — captures the first-run
  terms-of-service prompt, not a flag list
- **Upstream:** https://github.com/trustedsec/social-engineer-toolkit
- **Wiki:** https://github.com/trustedsec/social-engineer-toolkit/wiki

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

SET is a menu-driven framework that bundles common social-engineering attack
vectors: spear-phishing email and payload delivery, website attack pages
(templates, site cloner, tabnabbing, web jacking, HTA), credential harvesting,
infectious media, payload-and-listener generation, mass mail, Arduino and
PowerShell payloads, QR-code lures, SMS spoofing, and a wireless access-point
vector. It also exposes third-party modules and a Fast-Track penetration-testing
menu.

It is an orchestration layer, not a single exploit. Each module prompts for its
own parameters, may start a local web server or listener, and writes generated
files and logs into `/opt/set/src`. Many modules assume root, a low port, a
wireless adapter, or an external tool such as Metasploit; when a dependency is
missing, that module fails rather than degrading gracefully.

## Use cases

| Situation | Why SET |
|-----------|---------|
| Authorized phishing simulation | Ready-made templates and cloner |
| Credential-harvesting lab | Local harvester with logging |
| Awareness demonstration | Show clone, tabnabbing, and HTA vectors |
| Payload/listener exercise | Generate and catch a lab payload |
| QR-code lure exercise | Produce a scannable lure image |
| Wireless tabletop | AP vector when hardware is available |
| Framework comparison | Compare with Gophish and evilginx |

## Requirements & install layout

- **Root is mandatory.** The launcher checks the effective UID and exits with
  `Not running as root. Exiting the Social-Engineer Toolkit (SET).` when run as
  an ordinary user. On this host `user` cannot run SET until a sudo grant or
  an equivalent privilege path is approved.
- Python 3.13.5 virtualenv at `/opt/set/.venv`; the launcher runs
  `/opt/set/.venv/bin/python setoolkit`.
- Configuration lives under `/etc/setoolkit`. The install created
  `/opt/set/src/agreement4` so the terms prompt is skipped; the tree is owned
  `root:sectools` with group write, but that does not remove the root check.
- Ports 80 and 443 are used by several web vectors; free them before starting.
- The wireless AP vector needs a monitor-mode-capable adapter and hostapd.
- The Metasploit browser-exploit vector needs Metasploit installed and running.
- Some legacy vectors (Java applet, Flash-era pages) will not work against
  modern browsers regardless of configuration.
- Network access is needed for updates and for modules that fetch templates.

## Basic syntax

```bash
# start the interactive menu (requires root)
setoolkit

# non-interactive smoke test: print the banner and exit
printf '99\n' | setoolkit

# inspect the installed version metadata
grep -m1 '^version' /opt/set/pyproject.toml

# inspect configuration paths
ls -l /etc/setoolkit/set.config /etc/setoolkit/set_config.py
```

SET has no documented command-line flag surface in the installed snapshot. All
operation is through numbered menus.

## Menu reference

The labels below are the upstream SET 8.x menus. Menu numbers can shift between
releases; confirm the label at the prompt before selecting.

### Main menu

| Choice | Meaning |
|--------|---------|
| `1` | Social-Engineering Attacks |
| `2` | Penetration Testing (Fast-Track) |
| `3` | Third Party Modules |
| `4` | Update the Social-Engineer Toolkit |
| `5` | Update SET configuration |
| `6` | Help, Credits, and About |
| `99` | Exit |

### Social-Engineering Attacks

| Choice | Meaning |
|--------|---------|
| `1` | Spear-Phishing Attack Vectors |
| `2` | Website Attack Vectors |
| `3` | Infectious Media Generator |
| `4` | Create a Payload and Listener |
| `5` | Mass Mailer Attack |
| `6` | Arduino-Based Attack Vector |
| `7` | Wireless Access Point Attack Vector |
| `8` | QRCode Generator Attack Vector |
| `9` | Powershell Attack Vectors |
| `10` | SMS Spoofing Attack Vector |
| `11` | Third Party Modules |

### Website Attack Vectors

| Choice | Meaning |
|--------|---------|
| `1` | Java Applet Attack Method |
| `2` | Metasploit Browser Exploit Method |
| `3` | Credential Harvester Attack Method |
| `4` | Tabnabbing Attack Method |
| `5` | Web Jacking Attack Method |
| `6` | Multi-Attack Web Method |
| `7` | HTA Attack Method |

### Credential Harvester

| Choice | Meaning |
|--------|---------|
| `1` | Web Templates |
| `2` | Site Cloner |
| `3` | Custom Import |

## Recipes

```bash
# 1. Smoke test the launcher and exit
printf '99\n' | setoolkit

# 2. Start the credential harvester with the site cloner
setoolkit
#   choose 1 -> 2 -> 3 -> 2, then enter the clone URL and listener address

# 3. Start a spear-phishing workflow
setoolkit
#   choose 1 -> 1, then follow the mass-email or payload prompts

# 4. Generate a QR-code lure
setoolkit
#   choose 1 -> 8, enter the target URL, and note the output image path

# 5. Create a payload and listener
setoolkit
#   choose 1 -> 4 and follow the payload and listener prompts

# 6. Launch the mass mailer against a lab SMTP sink
setoolkit
#   choose 1 -> 5 and point the SMTP server at 127.0.0.1:2525

# 7. Infectious media generator for a lab USB image
setoolkit
#   choose 1 -> 3 and follow the prompts

# 8. Wireless AP vector (lab adapter required)
setoolkit
#   choose 1 -> 7 and select the wireless interface
```

## Output & pipelines

- Generated pages, payloads, QR images, and cloned content are written under
  `/opt/set/src` (for example `src/html`, `src/phishing`, `src/payloads`).
- Logs are written to `/opt/set/src/logs/set_logfile.log`.
- Harvested credentials appear in the console and in module logs; treat both as
  sensitive evidence.
- Listener and web-server state is visible with `ss -ltnp`.
- There is no JSON output; parse logs and captured files directly.
- Package evidence with `tar` and hash it before storage or transfer.

## Gotchas & operational security

- The root check is unconditional. A sudo grant is the only straightforward
  way for `user` to run SET; capabilities alone will not satisfy the UID
  check.
- Menu numbers are not stable across versions. Read the printed menu.
- Port 80 conflicts are common; stop other web servers before the harvester.
- The site cloner copies HTML but not always every asset; cloned pages may look
  broken and may contact the real site.
- Legacy vectors are unreliable on modern browsers and can waste engagement
  time.
- Metasploit-dependent modules fail if `msfconsole` is absent or the RPC
  service is not running.
- Email modules can send real mail. Point them at a lab sink and use test
  recipients only.
- The wireless vector needs hardware and regulatory compliance; transmitting
  on a channel you do not own may be illegal.
- Updates (menu 4) pull from the network and can change module behaviour
  mid-engagement; freeze versions during a test.
- Captured credentials, cookies, and payloads are sensitive. Encrypt evidence,
  restrict access, and delete it under the engagement's retention rules.
- `seautomate` and `seproxy` ship in `/opt/set` but are not described by the
  local help snapshot; read their source before relying on them.

## Legal & safety

SET is a social-engineering and credential-capture framework. Use it only in
authorized engagements with written scope, named targets, data-handling rules,
and a teardown plan. Sending phishing email, cloning a third-party site,
capturing real credentials, or transmitting on wireless spectrum without
authorization is unlawful and harmful. Prefer lab identities and test accounts,
coordinate with defenders, and stop immediately if real user data is captured
outside the agreed scope.

## Related tools

- [`gophish/`](../gophish/README.md) — campaign management and tracking
- [`evilginx/`](../evilginx/README.md) — AiTM proxy for MFA-resistant phishing
- [`wifiphisher/`](../wifiphisher/README.md) — wireless captive-portal phishing
- [`theharvester/`](../theharvester/README.md) — target and contact discovery
- `msfconsole` — payload generation and listeners

## References

- https://github.com/trustedsec/social-engineer-toolkit
- https://github.com/trustedsec/social-engineer-toolkit/wiki
- https://attack.mitre.org/techniques/T1566/ (Phishing)
- https://attack.mitre.org/techniques/T1598/ (Phishing for Information)
