# SET — command scenarios for deep research

Installed build: Social-Engineer Toolkit 8.1.3 at `/opt/set`, launcher
`/usr/local/bin/setoolkit`. SET is menu-driven; the local `help.txt` captures
only the terms-of-service prompt. Menu paths below are the upstream 8.x menus
and must be confirmed at the prompt.

## How to invoke

```text
@deep-research For SET 8.1.3 in 2026, which modules still work against modern
browsers and mail gateways, how should the credential harvester and mass mailer
be isolated in a lab, and what evidence and teardown artefacts does an
authorized phishing simulation need? Cite the SET wiki, MITRE ATT&CK T1566, and
red-team field reports; return a module reliability table and a lab runbook.
```

---

## Scenario 1 — preflight and configuration

| # | Command | How to use |
|---|---------|------------|
| 1 | `id -u` | Confirm effective UID 0 |
| 2 | `printf '99\n' \| setoolkit` | Banner and clean exit |
| 3 | `ls -l /etc/setoolkit/set.config /opt/set/src/agreement4` | Config and agreement markers |
| 4 | `ss -ltnp \| grep ':80\b'` | Check the harvester port |
| 5 | `/opt/set/.venv/bin/python -c 'import sys; print(sys.version)'` | Runtime version |

**Research questions**
- Which modules fail solely because of the root requirement, and which need
  additional capabilities?
- Which configuration values are environment-specific and must be reset
  between engagements?

**Sources:** launcher source, config files, `ss` output.
**Output:** a preflight checklist and a clean starting state.

---

## Scenario 2 — credential harvester in a lab

| # | Command | How to use |
|---|---------|------------|
| 6 | `setoolkit` then `1 → 2 → 3 → 2` | Website vectors, harvester, site cloner |
| 7 | `ss -ltnp \| grep ':80\b'` | Confirm the harvester is listening |
| 8 | `curl -s http://127.0.0.1/ \| head` | Fetch the cloned page locally |
| 9 | `tail -f /opt/set/src/logs/set_logfile.log` | Watch module logging |
| 10 | `grep -iE 'username\|password' /opt/set/src/logs/*` | Lab-only credential check |

**Research questions**
- Which fields does the cloned page actually submit, and where are they logged?
- What breaks when the target uses MFA or a JavaScript-heavy login flow?

**Sources:** SET wiki, cloned page source, logs.
**Output:** a lab capture record and a list of module limitations.

---

## Scenario 3 — spear phishing and mass mailer

| # | Command | How to use |
|---|---------|------------|
| 11 | `nc -l 127.0.0.1 2525` | Local SMTP sink for safe testing |
| 12 | `setoolkit` then `1 → 5` | Mass Mailer Attack |
| 13 | `setoolkit` then `1 → 1` | Spear-Phishing vectors |
| 14 | `ss -ltnp \| grep ':2525\b'` | Confirm the sink is up |
| 15 | `tail -n 30 /opt/set/src/logs/set_logfile.log` | Review send attempts |

**Research questions**
- Which headers and encoding does SET produce, and how do gateways score them?
- What is the minimum safe lab setup that never reaches a real mailbox?

**Sources:** SET wiki, SMTP sink output, mail-gateway documentation.
**Output:** a contained email test with no external delivery.

---

## Scenario 4 — payload, listener, and QR lure

| # | Command | How to use |
|---|---------|------------|
| 16 | `setoolkit` then `1 → 4` | Payload and listener |
| 17 | `ss -ltnp \| grep -E ':(443\|8080\|4444)\b'` | Confirm listener ports |
| 18 | `setoolkit` then `1 → 8` | QRCode generator |
| 19 | `find /opt/set/src -maxdepth 2 \( -iname '*.png' -o -iname '*.bmp' \)` | Locate QR output |
| 20 | `file <qr-image>` | Validate the image type |

**Research questions**
- Which payload formats are still viable on current endpoint controls?
- How is a QR lure tracked, and what does it reveal about scanning behavior?

**Sources:** SET wiki, listener output, image metadata.
**Output:** a lab payload/listener pair and a QR artefact.

---

## Scenario 5 — evidence and teardown

| # | Command | How to use |
|---|---------|------------|
| 21 | `tar -czf set-evidence.tgz /opt/set/src/logs` | Package logs |
| 22 | `sha256sum set-evidence.tgz` | Hash evidence |
| 23 | `pkill -f setoolkit` | **Destructive:** stop running SET processes |
| 24 | `ss -ltnp \| grep -E ':(80\|443\|2525)\b'` | Confirm listeners are gone |
| 25 | `ls -la /opt/set/src/logs` | Retention review before deletion |

**Research questions**
- Which artefacts are needed for the report, and which contain credentials that
  must be destroyed?
- How is process and listener teardown proven?

**Sources:** engagement retention rules, hashes, `ss` output.
**Output:** a hashed evidence bundle and a teardown record.

## Menu coverage checklist

| Menu area | Path | Scenario |
|-----------|------|----------|
| Main menu | `99` | 1 |
| Website vectors | `1 → 2` | 2 |
| Credential harvester | `1 → 2 → 3` | 2 |
| Mass mailer | `1 → 5` | 3 |
| Spear phishing | `1 → 1` | 3 |
| Payload/listener | `1 → 4` | 4 |
| QR generator | `1 → 8` | 4 |
| Logs/teardown | shell | 5 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — preflight | 5 |
| 2 — harvester | 5 |
| 3 — email | 5 |
| 4 — payload/QR | 5 |
| 5 — evidence/teardown | 5 |
