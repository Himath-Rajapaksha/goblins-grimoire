# wireshark — command scenarios for deep research

Every command form across the installed Wireshark 4.4.19 toolset: `dumpcap`
(capture), `tshark` (terminal analysis), and the GUI. Flags from
[`help-dumpcap.txt`](help-dumpcap.txt) and [`help-tshark.txt`](help-tshark.txt).
GUI actions are listed separately and labelled honestly as UI paths.

## How to invoke

```text
@deep-research For Wireshark 4.4, which tshark display filters and statistics
give the fastest evidence of credential exposure in an HTTP capture, and what
are the current pitfalls of TLS decryption via SSLKEYLOGFILE? Cite Wireshark
docs, the display-filter reference, and current TLS 1.3 writeups; return exact
commands and filters.
```

---

## Scenario 1 — find interfaces and capture capability

| # | Command | How to use |
|---|---------|------------|
| 1 | `dumpcap -D` | List capture interfaces with indices |
| 2 | `dumpcap -D -M` | Machine-readable interface list |
| 3 | `tshark -D` | Same list from the analysis tool |
| 4 | `dumpcap -i wlp2s0 -L` | List link-layer types for an interface |
| 5 | `dumpcap -i wlp2s0 -L -M` | Machine-readable link types |
| 6 | `dumpcap -i wlp2s0 --list-time-stamp-types` | Available timestamp sources |
| 7 | `dumpcap -i wlp2s0 -S` | Per-interface capture statistics once per second |
| 8 | `dumpcap -i wlp2s0 -S -M` | Machine-readable statistics |
| 9 | `dumpcap -v` | Version and build info |
| 10 | `tshark -v` | Analysis-tool version |
| 11 | `dumpcap -h` | Capture help |
| 12 | `tshark -h` | Analysis help |

**Research questions**
- Which interface names and timestamp types does this kernel expose, and does
  hardware timestamping actually work on `wlp2s0`?
- Why can `tshark -D` differ from `dumpcap -D` when run as different users?

**Sources**: Wireshark capture-setup docs, dumpcap man page, kernel timestamp
docs, issues on monitor-mode interface naming.

**Output**: a capture command bound to the correct interface index and
timestamp source.

---

## Scenario 2 — plain capture with filters and stop conditions

| # | Command | How to use |
|---|---------|------------|
| 13 | `dumpcap -i wlp2s0 -w cap.pcapng` | Capture until Ctrl-C |
| 14 | `dumpcap -i wlp2s0 -f 'tcp port 443' -w tls.pcapng` | BPF capture filter |
| 15 | `dumpcap -i wlp2s0 -f 'host 10.0.0.5 and not port 22' -w host.pcapng` | Compound BPF |
| 16 | `dumpcap -i wlp2s0 -f 'udp port 53' -d` | Print generated BPF code and exit |
| 17 | `dumpcap -i wlp2s0 -c 1000 -w first1000.pcapng` | Stop after N packets |
| 18 | `dumpcap -i wlp2s0 -a duration:60 -w one-minute.pcapng` | Stop after N seconds |
| 19 | `dumpcap -i wlp2s0 -a filesize:10240 -w small.pcapng` | Stop at N kB |
| 20 | `dumpcap -i wlp2s0 -a packets:5000 -w fivek.pcapng` | Stop at N packets |
| 21 | `dumpcap -i wlp2s0 -a files:10 -w part.pcapng` | Stop after N files |
| 22 | `dumpcap -i wlp2s0 -s 128 -w headers.pcapng` | Snapshot only first N bytes |
| 23 | `dumpcap -i wlp2s0 -p -w cap.pcapng` | Disable promiscuous mode |
| 24 | `dumpcap -i wlp2s0 -B 64 -w cap.pcapng` | 64 MiB kernel buffer |
| 25 | `dumpcap -i wlp2s0 -y LINUX_SLL2 -w cap.pcapng` | Force link-layer type |
| 26 | `dumpcap -i wlp2s0 --time-stamp-type host -w cap.pcapng` | Force timestamp type |
| 27 | `dumpcap -i wlp2s0 -P -w cap.pcap` | Write classic pcap instead of pcapng |
| 28 | `dumpcap -i wlp2s0 -n -w cap.pcapng` | Explicit pcapng (the default) |
| 29 | `dumpcap -i wlp2s0 --capture-comment 'client X baseline' -w cap.pcapng` | Embed a case note |
| 30 | `dumpcap -i wlp2s0 -g -w shared.pcapng` | Group-readable output file |
| 31 | `dumpcap -i wlp2s0 -q -w quiet.pcapng` | Suppress packet counters |
| 32 | `dumpcap -i wlp2s0 -N 100000 -C 67108864 -w cap.pcapng` | Buffer caps inside dumpcap |
| 33 | `dumpcap -i wlp2s0 -t --temp-dir /tmp/opencode -w cap.pcapng` | Thread per interface, custom temp dir |
| 34 | `dumpcap -i wlp2s0 -k 6,11,2412 -w wifi.pcapng` | Set Wi-Fi channel |

**Research questions**
- `-a duration` versus `timeout` in an outer wrapper: which really stops the
  capture on a busy link?
- When does `-s` truncation break TCP reassembly for the analysis I need?

**Sources**: dumpcap man page, pcap-filter(7), Wireshark capture docs, issues
on snapshot length.

**Output**: a capture that stops on its own and is small enough to analyze.

---

## Scenario 3 — long and rotating captures

| # | Command | How to use |
|---|---------|------------|
| 35 | `dumpcap -i wlp2s0 -b duration:3600 -w hourly.pcapng` | New file each hour |
| 36 | `dumpcap -i wlp2s0 -b filesize:102400 -w size.pcapng` | Rotate at 100 MB |
| 37 | `dumpcap -i wlp2s0 -b packets:1000000 -w pkts.pcapng` | Rotate per packet count |
| 38 | `dumpcap -i wlp2s0 -b interval:60 -w aligned.pcapng` | Align file switches to clock multiples |
| 39 | `dumpcap -i wlp2s0 -b files:24 -b duration:3600 -w ring.pcapng` | 24-file ring buffer |
| 40 | `dumpcap -i wlp2s0 -b duration:600 -b printname:stdout -w live.pcapng` | Print each new filename |
| 41 | `dumpcap -i wlp2s0 -a duration:86400 -b files:96 -w long.pcapng` | One-day capture with rotation |

**Research questions**
- Ring-buffer failure modes on full disk — does dumpcap stop or corrupt the
  newest file?
- Cost of `printname` on a high-packet-rate link?

**Sources**: dumpcap man page, issues on ring buffer, Wireshark docs.

**Output**: a rotation policy with a disk budget.

---

## Scenario 4 — read and slice existing captures

| # | Command | How to use |
|---|---------|------------|
| 42 | `tshark -r cap.pcapng` | Read a file, print summaries |
| 43 | `tshark -r -` | Read from STDIN |
| 44 | `tshark -r cap.pcapng -Y 'http.request'` | Display filter on read |
| 45 | `tshark -r cap.pcapng -Y 'ip.addr == 10.0.0.5' -w filtered.pcapng` | Write a filtered subset |
| 46 | `tshark -r cap.pcapng -Y 'tcp.stream eq 3' -w stream3.pcapng` | Isolate one TCP stream |
| 47 | `tshark -r cap.pcapng -2 -R 'http.request'` | Two-pass read filter |
| 48 | `tshark -r cap.pcapng -M 10000` | Session auto-reset for reassembly |
| 49 | `tshark -r cap.pcapng -c 50` | First N packets only |
| 50 | `tshark -r cap.pcapng -Y 'dns' -w dns.pcapng -F pcapng` | Convert while filtering |
| 51 | `tshark -r cap.pcapng -F` | List supported output file types |
| 52 | `tshark -r cap.pcapng -w out.pcapng.gz` | Gzip output by extension |
| 53 | `tshark -r cap.pcapng --compress gzip -w out.pcapng` | Explicit compression |
| 54 | `tshark -r cap.pcapng -C 'CaseProfile'` | Use a config profile |
| 55 | `tshark -r cap.pcapng --global-profile` | Use global rather than personal profile |

**Research questions**
- `-2` with `-R` versus `-Y`: which one changes reassembly results?
- Does `-M` reset state mid-stream and lose a conversation?

**Sources**: tshark man page, Wireshark reassembly docs, issues on read
filters.

**Output**: a per-question extract instead of one giant capture.

---

## Scenario 5 — field extraction for pipelines

| # | Command | How to use |
|---|---------|------------|
| 56 | `tshark -r cap.pcapng -T fields -e ip.src -e ip.dst` | Two columns |
| 57 | `tshark -r cap.pcapng -T fields -e http.host -e http.request.uri -E separator=/t` | Tab-separated |
| 58 | `tshark -r cap.pcapng -T fields -e dns.qry.name -E header=y` | Header row |
| 59 | `tshark -r cap.pcapng -T fields -e tls.handshake.extensions_server_name -E occurrence=a` | All occurrences |
| 60 | `tshark -r cap.pcapng -T fields -e frame.number -e _ws.col.info -E quote=d` | Quoted CSV |
| 61 | `tshark -r cap.pcapng -T fields -e http.cookie -E aggregator=,` | Join multi-values |
| 62 | `tshark -r cap.pcapng -T fields -e ip.src -E bom=y` | BOM for Excel |
| 63 | `tshark -r cap.pcapng -T json` | Full JSON |
| 64 | `tshark -r cap.pcapng -T jsonraw` | Raw JSON values |
| 65 | `tshark -r cap.pcapng -T json --no-duplicate-keys` | Merge duplicate JSON keys |
| 66 | `tshark -r cap.pcapng -T pdml` | XML packet details |
| 67 | `tshark -r cap.pcapng -T psml` | XML packet summaries |
| 68 | `tshark -r cap.pcapng -T ek` | Elasticsearch bulk format |
| 69 | `tshark -r cap.pcapng -T tabs` | Tab-delimited tree |
| 70 | `tshark -r cap.pcapng -T text` | Classic text (default) |
| 71 | `tshark -r cap.pcapng -j 'ip tcp'` | Layer filter for json/pdml/ek |
| 72 | `tshark -r cap.pcapng -J 'http'` | Expand only these protocols |
| 73 | `tshark -r cap.pcapng -T fields -e frame.time_epoch` | Epoch timestamps |
| 74 | `tshark -r cap.pcapng -t ad -u hms` | Absolute date, h:m:s |
| 75 | `tshark -r cap.pcapng -t ud` | Delta since previous packet |

**Research questions**
- Which `-T` formats survive large captures without exhausting memory?
- Precise semantics of `-j` versus `-J` child expansion?

**Sources**: tshark man page, `tshark -G help`, issues on fields output.

**Output**: a field list that answers the question in one pipeline pass.

---

## Scenario 6 — protocol decode and name resolution

| # | Command | How to use |
|---|---------|------------|
| 76 | `tshark -r cap.pcapng -d tcp.port==8888,http` | Decode As |
| 77 | `tshark -r cap.pcapng -d tcp.port==8443,tls` | Force TLS on a port |
| 78 | `tshark -r cap.pcapng --enable-protocol http2` | Enable a protocol |
| 79 | `tshark -r cap.pcapng --disable-protocol arp` | Disable a protocol |
| 80 | `tshark -r cap.pcapng --only-protocols dns,tcp` | Whitelist dissectors |
| 81 | `tshark -r cap.pcapng --disable-all-protocols --enable-protocol http` | Minimal dissection |
| 82 | `tshark -r cap.pcapng --enable-heuristic http2` | Enable a heuristic dissector |
| 83 | `tshark -r cap.pcapng --disable-heuristic http` | Disable one |
| 84 | `tshark -r cap.pcapng -n` | Disable all name resolution |
| 85 | `tshark -r cap.pcapng -N mtndsNvg` | Enable specific resolutions |
| 86 | `tshark -r cap.pcapng -H hosts.txt -w withhosts.pcapng` | Embed hosts-file names |
| 87 | `tshark -r cap.pcapng -o 'tcp.desegment_tcp_streams:FALSE'` | Preference override |
| 88 | `tshark -G fields` | Dump all field names |
| 89 | `tshark -G protocols` | Dump all protocols |
| 90 | `tshark -G help` | List report types |
| 91 | `tshark -r cap.pcapng -K /etc/krb5.keytab` | Kerberos decryption |

**Research questions**
- When does forced `Decode As` produce false positives that poison a report?
- Which resolution flags (`m`, `t`, `n`, `d`, `s`, `N`, `v`, `g`) do what?

**Sources**: tshark man page, Wireshark "Decode As" docs, name-resolution
docs.

**Output**: a decode plan that reflects the real protocol, not a guess.

---

## Scenario 7 — statistics and expert analysis

| # | Command | How to use |
|---|---------|------------|
| 92 | `tshark -r cap.pcapng -q -z io,phs` | Protocol hierarchy |
| 93 | `tshark -r cap.pcapng -q -z io,stat,1` | IO stats per second |
| 94 | `tshark -r cap.pcapng -q -z conv,tcp` | TCP conversations |
| 95 | `tshark -r cap.pcapng -q -z conv,ip` | IP conversations |
| 96 | `tshark -r cap.pcapng -q -z endpoints,ip` | IP endpoints |
| 97 | `tshark -r cap.pcapng -q -z expert` | Expert information |
| 98 | `tshark -r cap.pcapng -q -z http,tree` | HTTP request tree |
| 99 | `tshark -r cap.pcapng -q -z dns,tree` | DNS tree |
| 100 | `tshark -r cap.pcapng -q -z follow,tcp,ascii,0` | Follow TCP stream 0 |
| 101 | `tshark -r cap.pcapng -q -z follow,http,ascii,0` | Follow HTTP stream |
| 102 | `tshark -r cap.pcapng -q -z flow,tcp,network` | Flow graph |
| 103 | `tshark -r cap.pcapng -q -z hosts` | Hosts report |
| 104 | `tshark -r cap.pcapng -q -z plen,tree` | Packet-length tree |
| 105 | `tshark -r cap.pcapng -q -z rtp,streams` | RTP streams |
| 106 | `tshark -r cap.pcapng -q -z sip,stat` | SIP statistics |
| 107 | `tshark -r cap.pcapng -q -z smb,srt` | SMB service-response time |
| 108 | `tshark -r cap.pcapng -q -z tls,stat` | TLS statistics |
| 109 | `tshark -r cap.pcapng -z help` | List all statistics |

**Research questions**
- Which `-z` statistics require two-pass mode to be correct?
- Cost of `expert` on a multi-GB capture?

**Sources**: tshark man page statistics section, Wireshark stats docs.

**Output**: one statistics command per question, not a generic dump.

---

## Scenario 8 — objects, secrets, and evidence export

| # | Command | How to use |
|---|---------|------------|
| 110 | `tshark -r cap.pcapng --export-objects http,/tmp/objects` | Extract HTTP files |
| 111 | `tshark -r cap.pcapng --export-objects smb,/tmp/smbfiles` | Extract SMB files |
| 112 | `tshark -r cap.pcapng --export-objects tftp,/tmp/tftp` | Extract TFTP files |
| 113 | `tshark -r cap.pcapng --export-tls-session-keys keys.log` | Export TLS session keys |
| 114 | `tshark -r cap.pcapng -V` | Full packet tree |
| 115 | `tshark -r cap.pcapng -O http,tls` | Only selected protocol details |
| 116 | `tshark -r cap.pcapng -x` | Hex + ASCII dump |
| 117 | `tshark -r cap.pcapng --hexdump all` | All data sources |
| 118 | `tshark -r cap.pcapng --hexdump frames --hexdump noascii` | Frames only, no ASCII |
| 119 | `tshark -r cap.pcapng -P -w copy.pcapng` | Print summaries while writing |
| 120 | `tshark -r cap.pcapng -S '---' -V` | Custom packet separator |
| 121 | `tshark -r cap.pcapng -l` | Line-buffered live output |
| 122 | `tshark -r cap.pcapng -Q` | Errors only on stderr |
| 123 | `tshark -r cap.pcapng -g -w shared.pcapng` | Group-readable output |
| 124 | `tshark -r cap.pcapng -W n -w resolved.pcapng` | Save name-resolution data |
| 125 | `tshark -r cap.pcapng -X 'lua_script:script.lua'` | Extension option |
| 126 | `tshark -r cap.pcapng -U 'udp.port==5000,gtp'` | PDU export tap |

**Research questions**
- Do exported objects preserve hashes for evidence handling?
- Safety of exporting TLS keys — where should they live?

**Sources**: Wireshark export docs, forensics evidence guidance, TLS key log
docs.

**Output**: an evidence bundle with hashes and a documented chain of custody.

---

## Scenario 9 — GUI workflows (not CLI flags)

| # | Workflow | How to use |
|---|----------|------------|
| 127 | Open capture | File → Open, or drag a `.pcapng` onto the window |
| 128 | Capture options | Capture → Options → interface, promiscuous, filters, output |
| 129 | Display filter bar | Type a filter, green = valid; Enter applies |
| 130 | Follow stream | Right-click packet → Follow → TCP/UDP/TLS/HTTP Stream |
| 131 | Expert information | Analyze → Expert Information |
| 132 | Statistics menu | Statistics → Conversations / Endpoints / Protocol Hierarchy |
| 133 | Decode As | Right-click → Decode As → set protocol for a port |
| 134 | TLS decryption | Preferences → Protocols → TLS → (Pre)-Master-Secret log filename |
| 135 | Export objects | File → Export Objects → HTTP/SMB/TFTP |
| 136 | Coloring rules | View → Coloring Rules; edit severity colors |
| 137 | Profiles | Edit → Configuration Profiles; separate profiles per engagement |
| 138 | IO graph | Statistics → I/O Graph; plot throughput over time |
| 139 | TCP stream graph | Statistics → TCP Stream Graphs → Time/Sequence (tcptrace) |
| 140 | Save filtered | File → Export Specified Packets → Displayed |
| 141 | Capture comments | Packet comment (Ctrl+Alt+C); saved in pcapng |

**Research questions**
- Which GUI-only analyses (tcptrace, IO graph) have no stable tshark
  equivalent?
- Do profile settings leak between engagements if not separated?

**Sources**: Wireshark User's Guide, GUI docs, release notes.

**Output**: a GUI cheat sheet that maps each view to its CLI equivalent.

---

## Scenario 10 — remote, pipe, and wireless capture

| # | Command | How to use |
|---|---------|------------|
| 142 | `dumpcap -i TCP@10.0.0.5:2002 -w remote.pcapng` | Remote capture over TCP |
| 143 | `ssh host dumpcap -i eth0 -w - \| tshark -r -` | Pipe a remote capture |
| 144 | `dumpcap -i - --ifname eth0 --ifdescr 'remote eth0' -w pipe.pcapng` | Name a pipe source |
| 145 | `dumpcap -i wlp2s0 -I -w monitor.pcapng` | Monitor mode, if supported |
| 146 | `dumpcap -i wlp2s0 -I -k 36,5,5180 -w wifi5g.pcapng` | Monitor + channel |
| 147 | `tshark -i wlp2s0 -I -w monitor.pcapng` | Monitor capture via tshark |
| 148 | `tshark -i wlp2s0 -f 'wlan type mgt' -Y 'wlan.fc.type_subtype == 8'` | Beacon frames |
| 149 | `tshark -i eth0 -w - \| dumpcap -i - -w relayed.pcapng` | Relay through a pipe |
| 150 | `tshark -i wlp2s0 -l -T fields -e frame.number -e _ws.col.info` | Live field stream |

**Research questions**
- Which remote-capture transports are authenticated or encrypted, and which
  are plaintext?
- Monitor-mode support on this NIC/driver and the channel set available?

**Sources**: Wireshark remote-capture docs, dumpcap man page, driver docs,
issues on monitor mode.

**Output**: a wireless/remote capture path with explicit trust assumptions.

---

## Complete flag coverage

| Group | Flags | Scenario |
|-------|-------|----------|
| Interface | `-i`, `--ifname`, `--ifdescr`, `-D`, `-L`, `--list-time-stamp-types`, `-S`, `-M` | 1, 10 |
| Capture filter | `-f`, `-d` | 2 |
| Capture tuning | `-s`, `-p`, `-I`, `-B`, `-y`, `--time-stamp-type`, `-k`, `--update-interval` | 2, 10 |
| Stop | `-c`, `-a` (duration/filesize/files/packets) | 2 |
| Output files | `-w`, `-g`, `-b` (duration/filesize/files/packets/interval/printname), `-n`, `-P`, `--capture-comment`, `--temp-dir`, `--compress`, `-F` | 2, 3, 4 |
| Input | `-r` | 4 |
| Processing | `-2`, `-M`, `-R`, `-Y`, `-n`, `-N`, `-H`, `-d`, `--enable-protocol`, `--disable-protocol`, `--only-protocols`, `--disable-all-protocols`, `--enable-heuristic`, `--disable-heuristic`, `-C`, `--global-profile`, `-o`, `-K` | 4, 6 |
| Output format | `-V`, `-O`, `-P`, `-S`, `-x`, `--hexdump`, `-T`, `-j`, `-J`, `-e`, `-E`, `-t`, `-u`, `-l`, `-q`, `-Q`, `-W`, `-X`, `-U`, `--no-duplicate-keys`, `--color`, `--elastic-mapping-filter` | 5, 8 |
| Statistics | `-z` | 7 |
| Export | `--export-objects`, `--export-tls-session-keys` | 8 |
| Diagnostics | `--log-level`, `--log-fatal`, `--log-domains`, `--log-fatal-domains`, `--log-debug`, `--log-noisy`, `--log-file` | 1 |
| Misc | `-N`, `-C`, `-v`, `-h`, `-G` | 1, 6 |

Full per-flag text: [`help-dumpcap.txt`](help-dumpcap.txt),
[`help-tshark.txt`](help-tshark.txt).
