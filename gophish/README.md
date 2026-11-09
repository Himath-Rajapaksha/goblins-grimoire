# gophish — phishing campaign management platform

- **Version:** 0.12.1
- **Binary:** `/usr/local/bin/gophish` → `/opt/gophish/gophish`
- **Help snapshots:** [`help.txt`](help.txt) · [`help-long.txt`](help-long.txt)
- **Upstream:** https://github.com/gophish/gophish
- **Docs:** https://docs.getgophish.com/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Gophish is a self-hosted phishing framework with a web admin UI and a separate
phishing server. You define sending profiles (SMTP), landing pages, email
templates, and recipient groups, then launch a campaign and track opens,
clicks, submitted data, and reported messages. Results are stored in SQLite and
exposed through a REST API.

## Use cases

| Situation | Why gophish |
|-----------|-------------|
| Authorized phishing simulation | Full campaign lifecycle with metrics |
| Security awareness testing | Track who clicks and submits |
| Landing-page testing | Capture submitted credentials in a lab |
| Mail-gateway evaluation | Measure what filters block or allow |
| Multi-system deployment | Split admin and phish modes |
| API-driven automation | Create campaigns from scripts |

## Requirements & install layout

- Installed at `/opt/gophish`; wrapper in `/usr/local/bin`.
- Default admin UI: `https://127.0.0.1:3333` (self-signed cert).
- Default phish server: `http://127.0.0.1:80`.
- First start generates `config.json` and an admin password (printed once) in
  the working directory — run it from a dedicated directory.
- Ports 80/443 require root or `CAP_NET_BIND_SERVICE`.
- SQLite database `gophish.db` lives beside `config.json`.

## Basic syntax

```bash
# first run: generate config, print initial admin password
gophish

# custom config and working directory
cd /opt/gophish && gophish --config=/opt/gophish/config.json

# admin UI only (no phish server)
gophish --mode=admin

# phish server only (behind a load balancer)
gophish --mode=phish

# disable the mailer for multi-system setups
gophish --disable-mailer
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `--config <path>` | Location of `config.json` (default `./config.json`) |
| `--mode <all\|admin\|phish>` | Run all components, admin only, or phish only |
| `--disable-mailer` | Disable the mailer (multi-system deployments) |
| `--version` | Print version |
| `-h, --help` | Help (`--help-long`, `--help-man` for more) |

## Configuration highlights

`config.json` sections:

| Section | Key settings |
|---------|--------------|
| `admin_server` | `listen_url`, `use_tls`, `cert_path`, `key_path` |
| `phish_server` | `listen_url`, `use_tls`, `cert_path`, `key_path` |
| `db` | `driver` (`sqlite`/`mysql`), `path` or DSN |
| `smtp` | `host`, `port`, `username`, `password`, `from_address` |
| `logging` | `filename`, `level` |
| `admin_server` | `trusted_origins` for API access |

## Recipes

```bash
# 1. Prepare a dedicated directory
sudo mkdir -p /opt/gophish-data && cd /opt/gophish-data

# 2. First run to generate config and admin password
sudo gophish

# 3. Bind the admin UI to localhost only, keep phish on 80
#    edit config.json:
#    "admin_server": { "listen_url": "127.0.0.1:3333", "use_tls": true, ... }
#    "phish_server": { "listen_url": "0.0.0.0:80", "use_tls": false, ... }

# 4. Start with an explicit config
sudo gophish --config=/opt/gophish-data/config.json

# 5. Admin-only process (UI + API, no landing pages)
sudo gophish --mode=admin --config=/opt/gophish-data/config.json

# 6. Phish-only process behind a reverse proxy
sudo gophish --mode=phish --config=/opt/gophish-data/config.json

# 7. Allow the API from a admin workstation
#    "admin_server": { "trusted_origins": ["https://admin.example.net"] }

# 8. Verify the API
curl -k https://127.0.0.1:3333/api/campaigns/?api_key=<key>
```

## Output & pipelines

- Web UI for campaign creation and results.
- REST API under `/api/` (`/api/campaigns/`, `/api/groups/`,
  `/api/templates/`, `/api/pages/`, `/api/smtp/`) authenticated with an API
  key.
- SQLite `gophish.db` for offline analysis; back it up before upgrades.
- Webhooks per campaign post events (open, click, submit, report) to a URL.
- CSV export of campaign results from the UI.

## Gotchas & operational security

- **Only use on authorized engagements.** Phishing without written permission
  is unlawful.
- Default TLS certificate is self-signed; browsers and mail clients will warn.
- The initial admin password is printed once — capture it securely.
- Binding the admin UI to `0.0.0.0` exposes a powerful console; keep it on
  localhost or behind a VPN/reverse proxy with MFA.
- Landing pages that capture credentials store them in the DB in plaintext;
  encrypt the volume and purge after the engagement.
- Mail providers rate-limit and block; warm up domains and honor opt-outs.
- `--disable-mailer` means campaigns will not send — use only for split
  deployments.
- Version 0.12.1 is older; check upstream for security fixes before exposing it.

## Legal & safety

Phishing simulation requires explicit **written authorization**, a defined
target list, and an agreed stop condition. Never target personal accounts,
never use real credential-harvesting pages outside a lab, and report findings
responsibly.

## Related tools

- [`evilginx/`](../evilginx/README.md) — adversary-in-the-middle phishing
- [`set/`](../set/README.md) — social-engineering toolkit
- [`wifiphisher/`](../wifiphisher/README.md) — wireless phishing
- [`recon-ng/`](../recon-ng/README.md), [`theharvester/`](../theharvester/README.md) — target lists

## References

- https://docs.getgophish.com/
- https://github.com/gophish/gophish
- https://docs.getgophish.com/api-documentation
