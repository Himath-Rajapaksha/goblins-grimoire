# gophish — command scenarios for deep research

Installed build: gophish 0.12.1. Launch flags come from `help.txt`; the admin
and phish server behaviour is configured through `config.json`.

## How to invoke

```text
@deep-research For gophish 0.12.1, which deployment topology and config.json
settings keep the admin UI private while the phishing server stays reachable,
and how do TLS, trusted_origins, webhooks, and the API behave behind a reverse
proxy? Cite gophish docs, API documentation, and release notes; return a
hardened config and an API-driven campaign test.
```

---

## Scenario 1 — first run and credentials

| # | Command | How to use |
|---|---------|------------|
| 1 | `cd /opt/gophish-data && sudo gophish` | Generate config, print admin password |
| 2 | `ls -la /opt/gophish-data` | Inspect generated files |
| 3 | `sudo gophish --version` | Record the version |
| 4 | `sqlite3 gophish.db '.tables'` | Inspect the database schema |
| 5 | `curl -k https://127.0.0.1:3333/` | Confirm the admin UI responds |

**Research questions**
- Where is the initial password printed, and is it logged anywhere?
- Which files must be backed up before an upgrade?

**Sources:** gophish docs, generated config, release notes.
**Output:** a working install with a captured admin credential.

---

## Scenario 2 — split admin/phish deployment

| # | Command | How to use |
|---|---------|------------|
| 6 | `gophish --mode=admin --config=config.json` | Admin process only |
| 7 | `gophish --mode=phish --config=config.json` | Phish process only |
| 8 | `gophish --disable-mailer --config=config.json` | No local mailer |
| 9 | `gophish --config=/etc/gophish/config.json --mode=all` | Explicit config path |
| 10 | `ss -lntp \| grep -E '3333\|80'` | Verify listeners |

**Research questions**
- Which database does each mode open, and can they share one safely?
- Does `--disable-mailer` break campaign creation or only sending?

**Sources:** gophish docs, process logs, DB locks.
**Output:** a two-process deployment with separated roles.

---

## Scenario 3 — TLS and reverse proxy

| # | Command | How to use |
|---|---------|------------|
| 11 | `openssl req -x509 -newkey rsa:4096 -nodes -keyout key.pem -out cert.pem -days 365 -subj /CN=phish.example.net` | Self-signed cert |
| 12 | `grep -A8 admin_server config.json` | Review TLS settings |
| 13 | `grep -A8 phish_server config.json` | Review phish TLS settings |
| 14 | `nginx -t` | Validate a reverse-proxy config |
| 15 | `curl -I https://phish.example.net/` | Confirm TLS delivery |

**Research questions**
- Which proxy headers does gophish trust for client IPs?
- Does `use_tls` conflict with TLS termination at the proxy?

**Sources:** gophish docs, nginx docs, certificate transparency.
**Output:** valid TLS on both endpoints.

---

## Scenario 4 — API-driven campaign

| # | Command | How to use |
|---|---------|------------|
| 16 | `curl -k https://127.0.0.1:3333/api/campaigns/?api_key=<key>` | List campaigns |
| 17 | `curl -k -X POST https://127.0.0.1:3333/api/groups/ -H 'Content-Type: application/json' -d @group.json` | Create a group |
| 18 | `curl -k -X POST https://127.0.0.1:3333/api/templates/ -d @template.json` | Create a template |
| 19 | `curl -k -X POST https://127.0.0.1:3333/api/pages/ -d @page.json` | Create a landing page |
| 20 | `curl -k -X POST https://127.0.0.1:3333/api/campaigns/ -d @campaign.json` | Launch a campaign |
| 21 | `curl -k https://127.0.0.1:3333/api/campaigns/1/results?api_key=<key>` | Poll results |

**Research questions**
- Which API fields are mandatory for a campaign, and which default silently?
- How are webhook payloads signed or authenticated?

**Sources:** gophish API docs, payload samples, webhook receiver logs.
**Output:** a scripted campaign and a results feed.

---

## Scenario 5 — evidence, cleanup, and safety

| # | Command | How to use |
|---|---------|------------|
| 22 | `sqlite3 gophish.db '.backup /evidence/gophish.db'` | Back up results |
| 23 | `curl -k https://127.0.0.1:3333/api/campaigns/1/results?api_key=<key> > results.json` | Export results |
| 24 | `grep -c 'submitted_data' results.json` | Count submissions |
| 25 | `systemctl stop gophish` | Stop after the engagement |
| 26 | `shred -u /evidence/gophish.db` | Destroy captured data per policy |
| 27 | `iptables -L -n \| grep :80` | Confirm listeners are closed |

**Research questions**
- What retention period applies to captured credentials?
- Does deleting a campaign remove submitted data from SQLite?

**Sources:** engagement policy, gophish source, privacy law.
**Output:** a hashed evidence bundle and a verified teardown.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| Launch | `--config --disable-mailer --mode --version -h` | 1–3 |
| Config | admin/phish servers, TLS, DB, SMTP, logging, trusted_origins | 2–3 |
| API | campaigns, groups, templates, pages, results | 4 |
| Evidence | DB backup, results export, teardown | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — first run | 5 |
| 2 — split modes | 5 |
| 3 — TLS/proxy | 5 |
| 4 — API campaign | 6 |
| 5 — evidence/cleanup | 6 |
