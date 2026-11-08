# Tor Browser — hardened browser over the Tor network

- **Version:** 16.0 (Tor Project Firefox 153.6.0esr, bundled Tor 0.4.9.15)
- **Install root:** `/home/user/tor-browser`
- **Launcher:** `Browser/start-tor-browser` and the `.desktop` file
- **Desktop entry:** `~/.local/share/applications/start-tor-browser.desktop`
- **Full help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://www.torproject.org/
- **User manual:** https://support.torproject.org/tor-browser/
- **Verification:** https://check.torproject.org/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Tor Browser is Firefox ESR preconfigured to route all traffic through Tor,
with anti-fingerprinting patches, per-site circuit isolation, HTTPS-first
behavior, and three security levels. It bundles its own Tor daemon (SOCKS on
`127.0.0.1:9150`) so it does not depend on the system Tor service or the
`torsocks` shim.

Keep it in a per-user directory; do not mix it with a system Firefox profile.

## Use cases

| Situation | Approach |
|-----------|----------|
| Anonymous browsing | Normal launch, default security level |
| Checking a site as a Tor exit sees it | Launch, browse, verify at `check.torproject.org` |
| Censored network | Configure a bridge / pluggable transport |
| High-risk browsing | Security level *Safest* |
| Isolating identities | Separate `-profile` directories or *New Identity* |
| Onion services | Enter the `.onion` address directly |

## Launch

```bash
# Normal GUI launch
/home/user/tor-browser/Browser/start-tor-browser

# Detached (returns immediately), as the desktop entry does
/home/user/tor-browser/Browser/start-tor-browser --detach

# Verbose / debug output
/home/user/tor-browser/Browser/start-tor-browser --verbose
/home/user/tor-browser/Browser/start-tor-browser --debug

# Log to a file
/home/user/tor-browser/Browser/start-tor-browser --log /tmp/tb.log
```

Or launch **Tor Browser** from the application menu.

## `start-tor-browser` options

| Option | Meaning |
|--------|---------|
| `--detach` | Detach from the terminal and return |
| `-v, --verbose`, `-d, --debug` | Show launcher/browser output |
| `-h, -?, --help, -help` | Print usage |
| `-l, --log [file]` | Write debug output to a log file |
| `--register-app` | Register a desktop entry for this user |
| `--unregister-app` | Remove the desktop entry |

Note: `--detach` and verbose modes are mutually exclusive; the script
silently drops `--detach` when output is requested.

## First run

1. The first launch creates the profile under
   `Browser/TorBrowser/Data/Browser/profile.default/`.
2. Choose **Connect** (direct) or **Configure** (bridge/proxy).
3. Confirm Tor works: visit `https://check.torproject.org/`.
4. Optionally set the security level via the shield icon.

The bundled Tor listens on `127.0.0.1:9150` once the browser is running. That
port is separate from a system Tor on `9050` used by `torsocks` or
transparent Tor routing.

## Security levels

Set from the **shield icon → Security Level**, or per-profile via prefs.

| Level | Effect |
|-------|--------|
| **Standard** | All features enabled; least protection, best compatibility |
| **Safer** | Disables JavaScript on HTTP sites, some fonts, math rendering, and media autoplay on non-HTTPS origins |
| **Safest** | Disables JavaScript everywhere, limits fonts/icons, blocks audio/video, and restricts some rendering |

*Safest* breaks many sites — use it deliberately. Level changes take effect
after a reload and can be reset with **New Identity**.

## Bridges and pluggable transports

For censored or filtered networks, use **Configure → Bridges** on first run, or
**Settings → Connection → Bridges** later.

| Transport | Bundled binary | Use |
|-----------|----------------|-----|
| obfs4 | `lyrebird` | Most common; hides Tor traffic |
| meek (meek_lite) | `lyrebird` | Domain fronting |
| snowflake | `lyrebird` | Volunteer WebRTC proxies |
| webtunnel | `lyrebird` | HTTPS-looking traffic |
| conjure | `conjure-client` | Refraction networking |

Built-in bridge lines are available under *Select a built-in bridge*. To use a
custom bridge:

```
obfs4 <ip>:<port> <fingerprint> cert=<cert> iat-mode=0
```

Pluggable transports are configured in the bundled
`Browser/TorBrowser/Tor/torrc-defaults`:

```
ClientTransportPlugin meek_lite,obfs2,obfs3,obfs4,scramblesuit,webtunnel exec ./TorBrowser/Tor/PluggableTransports/lyrebird
ClientTransportPlugin snowflake exec ./TorBrowser/Tor/PluggableTransports/lyrebird
ClientTransportPlugin conjure exec ./TorBrowser/Tor/PluggableTransports/conjure-client -registerURL https://registration.refraction.network/api
```

Do not edit that file unless you need custom transports; the UI writes the
per-profile settings.

## Profile isolation

- **New Identity** (hamburger menu → *New Identity*) clears the current
  session's state and rotates circuits without restarting.
- **New Circuit for this Site** (shield icon → *New Circuit*) rotates the exit
  for the current tab only.
- True identity separation needs **separate profiles**:

```bash
# Launch with a distinct profile
/home/user/tor-browser/Browser/start-tor-browser \
  -profile /home/user/tor-browser-profiles/work
```

Firefox-style `-profile` / `-P` arguments pass through to the bundled browser.
Never run two instances on the same profile simultaneously.

## Desktop entry

A normalized entry is installed at:

```
/home/user/.local/share/applications/start-tor-browser.desktop
```

Key fields:

```ini
[Desktop Entry]
Type=Application
Name=Tor Browser
Exec=/home/user/tor-browser/Browser/start-tor-browser --detach %U
Icon=/home/user/tor-browser/Browser/browser/chrome/icons/default/default128.png
Categories=Network;WebBrowser;Security;
StartupWMClass=Tor Browser
```

The upstream `.desktop` uses a shell-wrapped `Exec=` line that fails
`desktop-file-validate`; the simplified `Exec=` above passes validation.

Validate after editing:

```bash
desktop-file-validate ~/.local/share/applications/start-tor-browser.desktop
```

> **Re-register caveat:** running
> `Browser/start-tor-browser --register-app` rewrites the desktop entry back to
> Tor's upstream shell-wrapped `Exec=` line and reintroduces the validation
> error. If you re-register, reapply the simplified `Exec=` afterwards.

## Recipes

```bash
# Launch detached
/home/user/tor-browser/Browser/start-tor-browser --detach

# Launch with logs for troubleshooting
/home/user/tor-browser/Browser/start-tor-browser --verbose --log /tmp/tb.log

# Check the bundled Tor version
/home/user/tor-browser/Browser/TorBrowser/Tor/tor --version

# Confirm the browser SOCKS port while running
ss -ltnp | grep 9150

# Verify the exit node (from inside the browser, visit:)
#   https://check.torproject.org/

# Use the browser's Tor daemon from the CLI while it runs
torsocks -P 9150 curl -s https://check.torproject.org/api/ip
```

## Updating

Tor Browser self-updates through its own updater (recommended). Manual updates:

1. Download the new Linux tarball from https://www.torproject.org/download/
2. Verify the GPG signature with the Tor Browser signing key.
3. Extract over the existing tree (or into a fresh directory and relink the
   desktop entry).

Do not mix a system Firefox profile with Tor Browser's profile.

## Gotchas

- **Never run Tor Browser as root.** Install and launch it as an unprivileged user.
- Tor Browser's Tor is separate from the system Tor. `torsocks` and
  transparent Tor routing do not control it, and it does not use `/etc/tor/torrc`.
- `check.torproject.org` is the ground truth — if it says *not using Tor*,
  something is misconfigured.
- Security level *Safest* disables JavaScript; modern SPAs will look broken.
- Adding extensions, changing `about:config`, or maximizing the window can
  reduce fingerprinting resistance.
- Do not resize the window to unusual dimensions if anonymity matters; letterboxing
  is intentional.
- The `.desktop` entry is per-user; `--register-app` must be run as that user.
- The launcher checks the install path at runtime; moving the directory breaks
  the desktop entry until it is re-registered.
- Screenshots and clipboard can leak identity; Tor Browser warns on some
  operations for a reason.

## References

- Download & signatures: https://www.torproject.org/download/
- User manual: https://support.torproject.org/tor-browser/
- Bridges: https://support.torproject.org/tor-browser/bridges/
- Security levels: https://support.torproject.org/tor-browser/features/security-levels/
- Check service: https://check.torproject.org/
