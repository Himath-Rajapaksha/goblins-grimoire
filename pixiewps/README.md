# pixiewps — offline WPS Pixie-Dust computation

- **Version:** Pixiewps 1.4 (Debian package `pixiewps` `1.4.2-5`)
- **Binary:** `/usr/bin/pixiewps`
- **Help snapshot:** [`help.txt`](help.txt) — `pixiewps -h` (72 lines)
- **Upstream:** https://github.com/wiire-a/pixiewps
- **Docs:** `pixiewps -h`, upstream README and the Pixie-Dust paper
- **Related:** `reaver -K` calls this logic internally; `pixiewps` is the
  standalone tool for replaying or scripting the computation

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Pixie-Dust is an offline attack on WPS. It exploits weak pseudo-random number
generation in some AP chipsets: if the enrollee's Diffie-Hellman private key
(or nonce) can be derived from a small seed space, the attacker can recompute
the session keys and then the WPS PIN from a captured handshake.

`pixiewps` takes the values from the WPS exchange and tries the known weak
generators. It does **not** talk to the network — you capture with `reaver
-K`, `wash`, or a pcap, then feed the values in.

| Input | WPS message | Meaning |
|-------|-------------|---------|
| `-e, --pke` | M1 | Enrollee's DH public key |
| `-r, --pkr` | M2 | Registrar's DH public key |
| `-s, --e-hash1` | M3 | Hash of the first PIN half |
| `-z, --e-hash2` | M3 | Hash of the second PIN half |
| `-a, --authkey` | M3 | Authentication session key |
| `-n, --e-nonce` | M1 | Enrollee's nonce |
| `-m, --r-nonce` | M2 | Registrar's nonce |
| `-b, --e-bssid` | — | Enrollee's BSSID |

## Argument reference

| Option | Meaning |
|--------|---------|
| `-e, --pke` | Enrollee's DH public key (M1) |
| `-r, --pkr` | Registrar's DH public key (M2) |
| `-s, --e-hash1` | Enrollee hash-1 (M3) |
| `-z, --e-hash2` | Enrollee hash-2 (M3) |
| `-a, --authkey` | Authentication session key |
| `-n, --e-nonce` | Enrollee's nonce (M1) |
| `-m, --r-nonce` | Registrar's nonce (M2) |
| `-b, --e-bssid` | Enrollee's BSSID |
| `-S, --dh-small` | Small DH keys (deprecated) |
| `--mode N[,N]` | Select computation modes |
| `--start [mm/]yyyy` | Start date for mode 3 |
| `--end [mm/]yyyy` | End date for mode 3 |
| `-7, --m7-enc` | Encrypted settings from M7 |
| `-5, --m5-enc` | Encrypted settings from M5 |

### Modes

| Mode | Name | Status |
|------|------|--------|
| 1 | RT/MT/CL | Default |
| 2 | eCos simple | Default |
| 3 | RTL819x | Default (date-bounded) |
| 4 | eCos simplest | Experimental |
| 5 | eCos Knuth | Experimental |

Experimental modes are not used unless you name them explicitly. Mode 3
depends on a time seed, so `--start` and `--end` bound the search; `--force`
makes it start at the current time and work back to the epoch.

### `-a` vs `-S`

`-a` needs the authentication session key, which stock reaver and bully do not
expose. The practical alternative is `-S` (small DH keys), which lets pixiewps
recompute the keys from `-n`, `-m`, and `-b`. The same `-S` must be used in
reaver, and the help warns that some APs misbehave with it — prefer `-a` when
you have it.

## Recipes

**1. Show the argument reference.**

```bash
pixiewps -h
```

**2. Standard short-form computation.**

```bash
pixiewps -e <pke> -s <e-hash1> -z <e-hash2> -S -n <e-nonce> -m <r-nonce> -b <e-bssid>
```

**3. Full form with the authentication key.**

```bash
pixiewps -e <pke> -r <pkr> -s <e-hash1> -z <e-hash2> -a <authkey> \
  -n <e-nonce> -m <r-nonce> -b <e-bssid>
```

**4. Restrict to one mode.**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3
```

**5. Mode 3 with a date window.**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> \
  --mode 3 --start 01/2020 --end 12/2023
```

**6. Force a full backwards search.**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> \
  --mode 3 --force
```

**7. Try experimental modes.**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 4,5
```

**8. Recover the PSK from M5/M7 (mode 3 only).**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> \
  -7 <enc7> -5 <enc5> --mode 3
```

**9. Recover the PSK and the PIN from M5/M7.**

```bash
pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> \
  -7 <enc7> -5 <enc5> -s <e-hash1> -z <e-hash2> --mode 3
```

**10. Capture the values with reaver first.**

```bash
sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -vv
```

## Gotchas

- **Pixiewps is offline.** It never touches the radio; a failure means the AP
  is not vulnerable to the modes you tried, not that the network is down.
- **It only works on vulnerable chipsets.** Patched and well-seeded APs are
  immune. Do not interpret a failure as a bad capture until you have checked
  the values.
- **`-S` must match reaver.** If reaver used `-S`, pixiewps must too, or the
  recomputed keys will not match.
- **`-S` is deprecated and can break some APs.** The help says to avoid it
  with reaver when possible.
- **`-a` needs a modified reaver or bully.** Stock tools do not print the
  authentication session key; that is why `-S` exists.
- **Mode 3 is date-bounded.** Without `--start`/`--end`, one bound defaults to
  the current time. `--force` searches all the way back to 1970.
- **Experimental modes are opt-in.** Modes 4 and 5 are not tried by default
  because they produce false positives on some APs.
- **`-5` requires `-7`.** The M5 settings only make sense together with the
  M7 settings.
- **`-7`/`-5` only work on some mode-3 APs.** They are not a general WPS
  decryptor.
- **Values are hex.** Copy them exactly; a truncated nonce or BSSID produces a
  silent mismatch.
- **A recovered PIN is not the passphrase.** The PIN enrols a client; the PSK
  is separate unless `-7`/`-5` recovered it.
- **Results are probabilistic.** The same inputs can take different paths
  depending on mode order and date bounds.
- **No network activity means no lockout.** That is the main advantage over
  the online PIN attack.

## Related docs

- [`reaver`](../reaver/README.md) — captures the handshake and drives
  Pixie-Dust
- [`wash`](../wash/README.md) — WPS survey
- [`wifite`](../wifite/README.md) — automated WPS path
- [`aircrack-ng`](../aircrack-ng/README.md) — monitor mode and capture
- [`bettercap`](../bettercap/README.md) — alternative Wi-Fi framework
- [`hcxdumptool`](../hcxdumptool/README.md) — PMKID and handshake capture
- [`iw`](../iw/README.md) — radio inspection
