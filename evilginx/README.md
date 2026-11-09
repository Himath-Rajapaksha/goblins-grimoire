# evilginx — adversary-in-the-middle phishing proxy

- **Version:** 3.3.0
- **Binary:** `/usr/local/bin/evilginx` (Go)
- **Help snapshot:** [`help.txt`](help.txt) (command-line flags only; the
  interactive console is documented upstream)
- **Upstream:** https://github.com/kgretzky/evilginx2
- **Project site:** https://evilginx.com/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

evilginx is a reverse-proxy framework for authorized phishing simulations. A
**phishlet** describes how to proxy a specific legitimate site: which domains
and paths to relay, which credentials to capture, and which session cookies to
record. A **lure** is a generated URL that ties a target to a phishlet through a
redirector page. Captured usernames, passwords, and cookies are stored in
**sessions** that an operator can inspect.

Because it relays the real site rather than serving a clone, the target sees the
genuine login flow and any MFA prompt. That is what makes it an
adversary-in-the-middle (AiTM) tool: if the target completes authentication, the
resulting session cookie can be replayed, so MFA alone does not stop the
simulation. This capability is why the tool is restricted to red-team
engagements with explicit written authorization.

## Use cases

| Situation | Why evilginx |
|-----------|--------------|
| Authorized phishing simulation | Measure whether MFA resists AiTM |
| Red-team initial access | Demonstrate session-token risk |
| Detection engineering | Generate realistic proxy/auth telemetry |
| Awareness training | Show the difference between clone and proxy phishing |
| Control validation | Test conditional access, token binding, and cookie policy |
| Lab research | Study phishlet and reverse-proxy behaviour safely |

## Requirements & install layout

- Static Go binary at `/usr/local/bin/evilginx`; no runtime dependencies.
- Configuration, phishlets, and redirector pages are directory arguments; this
  install does not create them automatically.
- Normal operation binds TCP 80 and 443, which requires root or
  `CAP_NET_BIND_SERVICE`. The `user` account currently has neither, so the
  proxy cannot be started without a privilege grant.
- Normal TLS mode obtains certificates for configured hostnames; `-developer`
  generates self-signed certificates for every hostname and is intended for
  isolated labs only.
- You need control of the DNS names used by the lure and phishlet hostnames.
- Phishlets are site-specific YAML files; a phishlet that no longer matches the
  target's flow will fail or capture nothing.
- Sessions and logs contain credentials and cookies: treat them as sensitive
  evidence.

## Basic syntax

```bash
# version
evilginx -v

# run with explicit configuration, phishlet, and redirector directories
evilginx -c ./lab/conf -p ./lab/phishlets -t ./lab/redirectors

# verbose run for troubleshooting a lab phishlet
evilginx -debug -c ./lab/conf -p ./lab/phishlets -t ./lab/redirectors

# self-signed certificates for an isolated lab
evilginx -developer -c ./lab/conf -p ./lab/phishlets -t ./lab/redirectors
```

Starting the proxy is an intrusive action. Do not launch it against a real
brand, domain, or audience without written authorization and a defined scope.

## Flag reference

The installed build exposes five command-line flags.

| Flag | Meaning |
|------|---------|
| `-c <path>` | Configuration directory path |
| `-p <path>` | Phishlets directory path |
| `-t <path>` | HTML redirector pages directory path |
| `-debug` | Enable debug output |
| `-developer` | Developer mode; generate self-signed certificates for all hostnames |
| `-v` | Show version |

There is no `-h` entry in the snapshot; an unknown flag prints the Go `flag`
usage block.

## Interactive console

evilginx drops into its own console after startup. The commands below are
upstream-documented for the 3.x line and are **not** captured in `help.txt`;
confirm them with `help` at the prompt.

| Console command | Meaning |
|-----------------|---------|
| `config domain <domain>` | Set the base domain used by lures and phishlets |
| `config ipv4 external <ip>` | Set the public address used in generated URLs |
| `phishlets hostname <phishlet> <hostname>` | Assign a hostname to a phishlet |
| `phishlets enable <phishlet>` | Enable proxying for a configured phishlet |
| `phishlets disable <phishlet>` | Disable a phishlet |
| `lures create <phishlet>` | Create a lure and print its ID |
| `lures get-url <id>` | Print the lure URL |
| `lures delete <id>` | Delete a lure |
| `sessions` | List captured sessions |
| `sessions <id>` | Show one session's captured data |
| `help` | List available console commands |

## Recipes

```bash
# 1. Confirm the build and flag surface
evilginx -v
evilginx 2>&1 | head -20

# 2. Prepare isolated lab directories
mkdir -p ./lab/conf ./lab/phishlets ./lab/redirectors

# 3. Check that the lure hostname resolves to the lab proxy
dig +short login.lab.example.test

# 4. Start a developer-mode proxy on an isolated lab host
evilginx -developer -c ./lab/conf -p ./lab/phishlets -t ./lab/redirectors

# 5. In the console: configure domain and external address
#    config domain lab.example.test
#    config ipv4 external 192.0.2.10

# 6. In the console: bind a phishlet and create a lure
#    phishlets hostname o365 login.lab.example.test
#    phishlets enable o365
#    lures create o365
#    lures get-url 0

# 7. Verify the listeners from another shell
ss -ltnp | grep -E ':(80|443)\b'

# 8. Review captured sessions, then disable the phishlet
#    sessions
#    phishlets disable o365
```

## Output & pipelines

- Console output is interactive; `-debug` adds verbose proxy and TLS detail.
- Captured sessions are held in the configuration directory and displayed in
  the console. Export or screenshot only within the engagement's evidence
  rules.
- Redirector pages are served from `-t`; keep them generic and non-branded
  unless the engagement authorizes otherwise.
- Proxy logs and session data can feed detection engineering: user-agent,
  cookie, and conditional-access telemetry.
- Destroy session data at engagement close according to the data-retention
  agreement.

## Gotchas & operational security

- Root or `CAP_NET_BIND_SERVICE` is required for ports 80/443. Without it the
  process fails at bind time.
- TLS certificates are issued for the configured hostnames; DNS must be
  correct before startup, and rate limits apply to public ACME services.
- `-developer` self-signed certificates trigger browser warnings and teach the
  target the wrong lesson; use it only in an isolated lab.
- Phishlets are brittle. A target UI change can break credential or cookie
  capture without an obvious error.
- Session cookies are equivalent to credentials. Do not replay them outside the
  authorized test account, and never retain them longer than the engagement
  allows.
- `-debug` logs URLs, headers, and token material; protect and delete logs.
- A lure URL is bearer-like: anyone who finds it can reach the proxy. Limit
  distribution and expire lures.
- The proxy is exposed to the internet by design; add network controls, monitor
  access, and shut it down immediately after the test window.
- Do not use real brand names, logos, or domains without written authorization
  from the brand owner and the testing organization.

## Legal & safety

evilginx is a phishing and credential-capture framework. Use it only in
authorized red-team or security-awareness engagements with a written scope,
defined target list, data-handling rules, and a teardown plan. Running it
against third parties, capturing real users' credentials without consent, or
replaying session cookies outside the agreed test is unlawful and can cause
serious harm. Prefer test accounts and lab identities; if real users must be
targeted, obtain explicit approval and coordinate with the defenders.

## Related tools

- [`gophish/`](../gophish/README.md) — campaign management and landing pages
- [`set/`](../set/README.md) — menu-driven social-engineering framework
- [`wifiphisher/`](../wifiphisher/README.md) — wireless captive-portal phishing
- `mitmproxy` — inspect and script proxy traffic
- `dnstwist` — look-alike domain discovery for defensive testing

## References

- https://github.com/kgretzky/evilginx2
- https://evilginx.com/
- https://attack.mitre.org/techniques/T1557/ (Adversary-in-the-Middle)
- https://attack.mitre.org/techniques/T1539/ (Steal Web Session Cookie)
