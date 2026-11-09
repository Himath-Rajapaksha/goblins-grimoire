# wpa_supplicant — WPA/WPA2/WPA3 supplicant and control client

- **Version:** 2.10-24 (Debian package `wpasupplicant`)
- **Binaries:** `/usr/sbin/wpa_supplicant`, `/usr/sbin/wpa_cli`, `/usr/bin/wpa_passphrase`
- **Help snapshots:** [`help.txt`](help.txt), [`help-wpa_cli.txt`](help-wpa_cli.txt), [`help-wpa_passphrase.txt`](help-wpa_passphrase.txt)
- **Upstream:** https://w1.fi/wpa_supplicant/
- **Licence:** BSD-3-Clause

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

`wpa_supplicant` is the daemon that authenticates a Linux station to a Wi-Fi
network: WPA/WPA2-Personal (PSK), WPA/WPA2/WPA3-Enterprise (802.1X/EAP), SAE,
OWE, and wired 802.1X. It owns the 802.11 authentication and association state
machine, runs the EAPOL four-way handshake, installs keys in the driver, and
exposes a control socket that `wpa_cli` and NetworkManager drive.

For assessments it is both a target surface and a lab tool:

- Stand up a client quickly against a test AP without NetworkManager.
- Drive scans, associations, and EAP identity changes from scripts.
- Capture the control-socket conversation to see what a supplicant leaks.
- Test WPS, PMF, SAE, and enterprise EAP configurations deterministically.
- Generate a 64-hex PSK from a passphrase for config files and hashcat checks.

## Architecture

```text
        config file (-c)                 control socket (-C/-g)
              │                                  │
              ▼                                  ▼
     ┌─────────────────┐   nl80211/wext   ┌──────────────┐
     │ wpa_supplicant  │◀───────────────▶│   driver     │
     │  state machine  │                 │  + hardware  │
     └────────┬────────┘                 └──────────────┘
              │ EAPOL / EAP
              ▼
        authenticator (AP or RADIUS)

     wpa_cli ──unix socket──▶ wpa_supplicant
     wpa_passphrase ──stdout──▶ config file
```

## Binaries

| Binary | Role |
|--------|------|
| `wpa_supplicant` | The daemon: config parsing, driver control, EAPOL, EAP |
| `wpa_cli` | Control-socket client: status, scan, network management, WPS, P2P |
| `wpa_passphrase` | Convert an SSID and passphrase into a `network={}` block with a PBKDF2 PSK |

## `wpa_supplicant` options

| Option | Meaning |
|--------|---------|
| `-b BRIDGE` | Optional bridge interface to add the interface to |
| `-B` | Run as a daemon in the background |
| `-c FILE` | Configuration file (required unless `-C` is used) |
| `-C CTRL` | `ctrl_interface` value, used only when `-c` is absent |
| `-d` | Increase debug verbosity; `-dd` is more |
| `-D DRIVER` | Driver: `nl80211`, `wext`, `wired`, `macsec_linux`, `none`; comma-separate to try several |
| `-e FILE` | Entropy file |
| `-f FILE` | Log to a debug file instead of stdout |
| `-g CTRL` | Global control interface (for `INTERFACE_ADD` and P2P) |
| `-G GROUP` | Group ownership for the global control socket |
| `-h` | Help |
| `-i IFACE` | Interface name |
| `-I FILE` | Additional configuration file |
| `-K` | Include keys and passwords in debug output |
| `-L` | Show the BSD licence |
| `-m FILE` | P2P Device configuration file |
| `-N` | Start a new interface description (repeatable) |
| `-o PARAM` | Override driver parameter for new interfaces |
| `-O CTRL` | Override `ctrl_interface` for new interfaces |
| `-p PARAM` | Driver parameters |
| `-P FILE` | PID file |
| `-q` | Decrease verbosity; `-qq` is quieter |
| `-s` | Log to syslog instead of stdout |
| `-t` | Include timestamps in debug messages |
| `-T` | Record to Linux tracing as well as logging |
| `-u` | Enable the D-Bus control interface |
| `-v` | Show version |
| `-W` | Wait for a control-interface monitor before starting |

Drivers are tried in the order given. `nl80211` is the modern path;
`wext` is the legacy wireless-extensions path; `wired` runs 802.1X on an
Ethernet interface; `none` is for RADIUS/WPS external-registrar use.

## `wpa_cli` options

| Option | Meaning |
|--------|---------|
| `-p PATH` | Control-socket directory (default `/var/run/wpa_supplicant`) |
| `-i IFACE` | Interface to control |
| `-g CTRL` | Global control interface |
| `-G INTERVAL` | Ping interval |
| `-a FILE` | Action file run on events (daemon mode) |
| `-r` | Reconnect when the socket drops (with `-a`) |
| `-B` | Run the action daemon in the background |
| `-P FILE` | PID file |
| `-s FILE` | Client socket path |
| `-h` | Help |
| `-v` | Version |

With no command, `wpa_cli` opens an interactive prompt. With a command, it
runs one request and exits.

## `wpa_passphrase`

```text
wpa_passphrase <ssid> [passphrase]
```

If the passphrase is omitted it is read from stdin. The output is a complete
`network={}` block:

```text
network={
	ssid="TestSSID"
	#psk="correct horse battery staple"
	psk=153b214cc5a5364b59f93e23c6976ff7b67e4266857d0e76fe62cbab64e549dc
}
```

The `#psk=` line is a comment containing the plaintext passphrase. Delete it
before committing the config. A passphrase must be 8–63 characters; a 64-hex
value is treated as a pre-computed PSK.

## Configuration file

A minimal WPA2-Personal configuration:

```ini
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
country=US

network={
    ssid="lab-ap"
    psk="correct horse battery staple"
    key_mgmt=WPA-PSK
    proto=RSN
    pairwise=CCMP
    group=CCMP
}
```

An 802.1X/EAP configuration:

```ini
network={
    ssid="corp"
    key_mgmt=WPA-EAP
    eap=PEAP
    identity="alice"
    password="secret"
    phase2="auth=MSCHAPV2"
    ca_cert="/etc/ssl/certs/corp-ca.pem"
    domain_suffix_match="radius.corp.example"
}
```

WPA3-SAE:

```ini
network={
    ssid="lab-sae"
    key_mgmt=SAE
    sae_password="correct horse battery staple"
    ieee80211w=2
}
```

Common globals:

| Directive | Meaning |
|-----------|---------|
| `ctrl_interface=DIR=... GROUP=...` | Where the control socket lives and who may use it |
| `update_config=1` | Allow `wpa_cli save_config` to rewrite the file |
| `country=XX` | Regulatory country |
| `ap_scan=0/1/2` | Driver/hardware scan versus supplicant scan |
| `fast_reauth=1` | Cache PMK for fast roaming |
| `pmf=0/1/2` | Protected Management Frames policy |
| `okc=1` | Opportunistic key caching |
| `bgscan="simple:30:-70:300"` | Background scanning policy |
| `p2p_disabled=1` | Disable P2P |
| `wps_cred_processing=0/1/2` | How WPS credentials are exposed |

Network-block directives worth knowing:

| Directive | Meaning |
|-----------|---------|
| `ssid`, `bssid` | Target SSID and optional BSSID pin |
| `psk`, `sae_password`, `password` | Personal, SAE, or EAP secret |
| `key_mgmt` | `WPA-PSK`, `WPA-EAP`, `SAE`, `OWE`, `NONE`, `IEEE8021X` |
| `proto` | `WPA`, `RSN` |
| `pairwise`, `group` | `CCMP`, `TKIP`, `GCMP`, `CCMP-256`, `GCMP-256` |
| `auth_alg` | `OPEN`, `SHARED`, `LEAP`, `SAE` |
| `scan_ssid=1` | Probe for a hidden SSID |
| `priority` | Higher wins when several networks are in range |
| `eap`, `identity`, `anonymous_identity`, `phase2`, `ca_cert`, `client_cert`, `private_key` | EAP method and credentials |
| `ieee80211w` | PMF: 0 disabled, 1 optional, 2 required |
| `owe_only=1` | Require OWE transition mode |

## `wpa_cli` command groups

The full list is in [`help-wpa_cli.txt`](help-wpa_cli.txt). Useful groups:

| Group | Commands |
|-------|----------|
| Status | `status [verbose]`, `ifname`, `ping`, `mib`, `signal_poll`, `pktcnt_poll`, `driver_flags`, `get <name>` |
| Networks | `list_networks`, `add_network`, `set_network`, `get_network`, `remove_network`, `select_network`, `enable_network`, `disable_network`, `save_config` |
| Scan | `scan`, `scan_results`, `abort_scan`, `bss <idx\|bssid>`, `all_bss`, `bss_flush`, `scan_interval` |
| Association | `reassociate`, `reattach`, `reconnect`, `disconnect`, `roam <addr>`, `preauthenticate <bssid>` |
| Credentials | `list_creds`, `add_cred`, `set_cred`, `get_cred`, `remove_cred` |
| PMKSA | `pmksa`, `pmksa_flush`, `pmksa_get`, `pmksa_add` |
| WPS | `wps_pbc`, `wps_pin`, `wps_check_pin`, `wps_cancel`, `wps_reg`, `wps_er_*`, `wps_ap_pin` |
| P2P | `p2p_find`, `p2p_connect`, `p2p_listen`, `p2p_group_add`, `p2p_peers`, `p2p_invite`, ... |
| AP mode | `sta`, `all_sta`, `list_sta`, `deauthenticate`, `disassociate`, `chan_switch` |
| Enterprise | `identity`, `password`, `new_password`, `otp`, `sim`, `get_capability`, `interworking_*`, `anqp_get`, `hs20_*` |
| DPP | `dpp_qr_code`, `dpp_bootstrap_*`, `dpp_auth_init`, `dpp_configurator_*` |
| Misc | `log_level`, `reconfigure`, `terminate`, `raw <cmd>`, `flush`, `vendor`, `twt_*` |

## Recipes

**1. Start against a lab AP in the foreground.**

```bash
sudo wpa_supplicant -Dnl80211 -iwlan0 -c/etc/wpa_supplicant/lab.conf
```

**2. Start as a daemon with a PID file and log.**

```bash
sudo wpa_supplicant -B -P /run/wpa_supplicant.wlan0.pid \
  -f /var/log/wpa_supplicant.wlan0.log \
  -Dnl80211 -iwlan0 -c/etc/wpa_supplicant/lab.conf
```

**3. Check link state.**

```bash
wpa_cli -i wlan0 status
wpa_cli -i wlan0 signal_poll
wpa_cli -i wlan0 pktcnt_poll
```

**4. Scan and list results.**

```bash
wpa_cli -i wlan0 scan
sleep 3
wpa_cli -i wlan0 scan_results
wpa_cli -i wlan0 all_bss
```

**5. Add a network from the control socket.**

```bash
wpa_cli -i wlan0 add_network
wpa_cli -i wlan0 set_network 0 ssid '"lab-ap"'
wpa_cli -i wlan0 set_network 0 psk '"correct horse battery staple"'
wpa_cli -i wlan0 enable_network 0
wpa_cli -i wlan0 select_network 0
wpa_cli -i wlan0 save_config
```

**6. Generate a PSK block.**

```bash
wpa_passphrase lab-ap 'correct horse battery staple' \
  | sed '/#psk=/d' >> /etc/wpa_supplicant/wpa_supplicant.conf
```

**7. Read a passphrase from stdin.**

```bash
printf '%s\n' "$WIFI_PASSPHRASE" | wpa_passphrase lab-ap
```

**8. Run a wired 802.1X supplicant.**

```bash
sudo wpa_supplicant -Dwired -ieth0 -c/etc/wpa_supplicant/wired.conf
```

**9. Drive multiple interfaces from one process.**

```bash
sudo wpa_supplicant -g/run/wpa_supplicant-global \
  -N -iwlan0 -c/etc/wpa_supplicant/wlan0.conf \
  -N -iwlan1 -c/etc/wpa_supplicant/wlan1.conf
```

**10. Attach a script to events.**

```bash
cat > /usr/local/bin/wpa-event.sh <<'EOF'
#!/bin/sh
case "$2" in
  CONNECTED) logger "wpa: connected to $3" ;;
  DISCONNECTED) logger "wpa: disconnected" ;;
esac
EOF
chmod +x /usr/local/bin/wpa-event.sh
wpa_cli -i wlan0 -a /usr/local/bin/wpa-event.sh
```

**11. Enable WPS push-button on a test AP.**

```bash
wpa_cli -i wlan0 wps_pbc
```

**12. Tear down cleanly.**

```bash
wpa_cli -i wlan0 terminate
```

## Exit status

| Binary | Code | Meaning |
|--------|------|---------|
| `wpa_supplicant` | 0 | Clean exit after a signal |
| `wpa_supplicant` | 255 | Startup failure: bad driver, missing interface, bad config |
| `wpa_cli` | 0 | Command completed (including `FAIL` responses) |
| `wpa_cli` | 255 | Cannot connect to the control socket |
| `wpa_passphrase` | 0 | Block written |
| `wpa_passphrase` | 1 | Missing SSID, missing stdin passphrase, or passphrase length error |

`wpa_supplicant` prints `Successfully initialized wpa_supplicant` before many
startup failures, so do not treat that line as success.

## Quirks and limits

- **The Debian package is `wpasupplicant`.** `dpkg -s wpa_supplicant` reports
  "not installed" even though the binaries exist.
- **The daemon returns 255 on startup errors.** Missing interface, bad driver,
  and unreadable config all exit 255 after the "Successfully initialized" line.
- **`-D wired -i lo` runs and blocks.** It is a valid configuration, so
  `timeout` is the only way to end it non-interactively.
- **`-c` is required unless you use `-C`.** Passing `/dev/null` as the config
  is accepted but leaves no networks.
- **`wpa_cli` needs a live socket.** With a bad `-p`, it exits 255 with
  `Failed to connect to non-global ctrl_ifname`.
- **`wpa_cli` returns 0 even when the command answers `FAIL`.** Parse the
  output, not the exit code.
- **`wpa_passphrase` echoes the passphrase in a comment.** Strip `#psk=` before
  storing or sharing the config.
- **A short passphrase is an error, not a truncation.** It prints
  `Passphrase must be 8..63 characters` and exits 1.
- **A 64-hex passphrase is a PSK, not a passphrase.** It is copied through
  without PBKDF2.
- **`-K` logs secrets.** Only use it on a lab machine with protected logs.
- **Control-socket permissions matter.** `GROUP=netdev` lets unprivileged
  users drive the daemon; that is a local privilege boundary.
- **PMF and SAE depend on the driver.** `ieee80211w=2` and `key_mgmt=SAE` fail
  on adapters without the required nl80211 features.
- **WPS is legacy and often disabled.** `wps_pbc` may time out even on APs that
  advertise WPS.
- **The control protocol is text over a UNIX socket.** `socat` can speak it
  directly, which is useful for testing and for understanding what a local
  attacker could do.

## Related tools

- [`iw`](../iw/) — nl80211 interface configuration and scan inspection.
- [`wireless-tools`](../wireless-tools/) — legacy `iwconfig`/`iwlist` interface.
- [`aircrack-ng`](../aircrack-ng/) — capture and crack WPA handshakes.
- [`hcxdumptool`](../hcxdumptool/) — PMKID/EAPOL capture.
- [`hcxtools`](../hcxtools/) — convert captures for hashcat.
- [`eaphammer`](../eaphammer/) — rogue 802.1X/EAP authentication server.
- `hostapd` — the AP-side counterpart (not installed here).
- [`wireshark`](../wireshark/) — dissect EAPOL and EAP exchanges.
