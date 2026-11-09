# SSLyze — command scenarios for deep research

Installed build: SSLyze 6.3.1. Flags come from `help.txt`; JSON schema from the
SSLyze documentation.

## How to invoke

```text
@deep-research For SSLyze 6.3.1, how do its findings compare with testssl.sh
and raw OpenSSL for protocol support, certificate validation, and ROBOT, and
which JSON paths are stable for CI gating across versions? Cite SSLyze docs,
the ScanResult schema, and cross-tool comparisons; return a JSON query set and
a disagreement-triage procedure.
```

---

## Scenario 1 — default and JSON

| # | Command | How to use |
|---|---------|------------|
| 1 | `sslyze example.com` | Default scan |
| 2 | `sslyze --json_out tls.json example.com` | JSON evidence |
| 3 | `sslyze --json_out - example.com` | JSON to stdout |
| 4 | `sslyze --quiet --json_out tls.json example.com` | No console noise |
| 5 | `sslyze --help` | Confirm command set |

**Research questions**
- Which commands does the default set include in 6.3.1?
- Which JSON keys are stable across 6.x versions?

**Sources:** docs, JSON output, changelog.
**Output:** a baseline JSON record.

---

## Scenario 2 — protocols and ciphers

| # | Command | How to use |
|---|---------|------------|
| 6 | `sslyze --sslv2 --sslv3 example.com` | Legacy protocols |
| 7 | `sslyze --tlsv1 --tlsv1_1 example.com` | Deprecated TLS |
| 8 | `sslyze --tlsv1_2 --tlsv1_3 example.com` | Modern TLS |
| 9 | `sslyze --elliptic_curves example.com` | Curves |
| 10 | `sslyze --mozilla_config modern example.com` | Mozilla modern profile |

**Research questions**
- Does SSLyze report cipher lists per protocol or only support booleans?
- How does Mozilla profile checking differ from explicit protocol flags?

**Sources:** docs, JSON, Mozilla TLS config.
**Output:** a protocol/cipher compliance matrix.

---

## Scenario 3 — certificates and mTLS

| # | Command | How to use |
|---|---------|------------|
| 11 | `sslyze --certinfo example.com` | Certificate analysis |
| 12 | `sslyze --certinfo --certinfo_ca_file /etc/ssl/certs/ca-certificates.crt example.com` | Custom trust store |
| 13 | `sslyze --cert client.pem --key client.key --certinfo internal.example.com:8443` | mTLS |
| 14 | `sslyze --sni internal.example.com 203.0.113.10` | SNI override |
| 15 | `sslyze --update_trust_stores` | Refresh CAs |

**Research questions**
- How are chain errors distinguished from expiry?
- What does mTLS validation require beyond presenting a cert?

**Sources:** docs, output, PKI guidance.
**Output:** a certificate validity assessment.

---

## Scenario 4 — vulnerabilities and headers

| # | Command | How to use |
|---|---------|------------|
| 16 | `sslyze --heartbleed example.com` | Heartbleed |
| 17 | `sslyze --robot example.com` | ROBOT |
| 18 | `sslyze --openssl_ccs --reneg example.com` | CCS and renegotiation |
| 19 | `sslyze --compression --fallback example.com` | CRIME and downgrade |
| 20 | `sslyze --http_headers example.com` | Security headers |

**Research questions**
- Which checks are definitive versus heuristic?
- How should a single-tool ROBOT finding be confirmed?

**Sources:** CVE docs, cross-tool runs, research papers.
**Output:** a confirmed vulnerability list.

---

## Scenario 5 — fleet scans and CI gating

| # | Command | How to use |
|---|---------|------------|
| 21 | `sslyze --targets_in targets.txt --json_out fleet.json --quiet` | Fleet scan |
| 22 | `sslyze --slow_connection --json_out slow.json example.com` | Fragile target |
| 23 | `jq 'keys' fleet.json` | Inspect schema |
| 24 | `jq '.. \| objects \| select(.status? == "COMPLETED")' fleet.json` | Completed scans |
| 25 | `sha256sum fleet.json` | Hash evidence |

**Research questions**
- How should CI gate on a specific finding without parsing console text?
- Which targets need `--slow_connection` to avoid false errors?

**Sources:** JSON schema, CI logs, target behaviour.
**Output:** an automated, hashed fleet report.

## Flag coverage checklist

| Help group | Flags | Scenario |
|------------|-------|----------|
| I/O | targets, `--targets_in --json_out --quiet` | 1, 5 |
| Trust | `--cert --key --keyform --pass --certinfo_ca_file --update_trust_stores` | 3 |
| Connectivity | `--slow_connection --https_tunnel --starttls --xmpp_to --sni --mozilla_config` | 2–5 |
| Scans | protocol flags, `--certinfo --heartbleed --robot --openssl_ccs --reneg --compression --http_headers` | 2–4 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — default/JSON | 5 |
| 2 — protocols | 5 |
| 3 — certificates | 5 |
| 4 — vulnerabilities | 5 |
| 5 — fleet/CI | 5 |
