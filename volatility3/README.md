# Volatility 3 — memory forensics framework

- **Version:** Volatility 3 Framework 2.28.2
- **Binary:** `/usr/local/bin/vol` (entry point)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/volatilityfoundation/volatility3
- **Docs:** https://volatility3.readthedocs.io/

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Volatility 3 analyzes RAM images, crash dumps, hibernation files, and live
memory acquisitions. It identifies the operating system and kernel from the
image, builds a symbol table (ISF) for that kernel, and runs plugins that walk
kernel structures to recover processes, network connections, loaded modules,
registry hives, files, and injected code. On Linux it can also recover bash
history, environment variables, and page-cache files.

## Use cases

| Situation | Why Volatility 3 |
|-----------|------------------|
| Incident response | Process, network, and module triage from RAM |
| Malware analysis | `malfind`, hidden modules, process spoofing |
| Rootkit hunting | `check_syscall`, `check_idt`, `hidden_modules` |
| Linux forensics | `bash`, `envars`, `lsof`, `pagecache` |
| Windows forensics | `pslist`, `netscan`, `registry`, `cmdline` |
| Timeline building | `timeliner` |
| Evidence extraction | `dumpfiles`, `module_extract`, renderers |

## Requirements & install layout

- Installed from pipx at `/opt/pipx`; entry point `/usr/local/bin/vol`.
- Python 3.13 environment; no root needed to analyze an image you can read.
- Symbol tables (ISF JSON) are downloaded on demand and cached under
  `~/.cache/volatility3`; use `--offline` in air-gapped labs.
- Some plugins require additional dependencies (yara, pycryptodome, etc.); the
  help shows several plugins could not be loaded on this host.
- `-f` is shorthand for `--single-location=file://`.

## Basic syntax

```bash
# identify the image
vol -f memory.raw banners.Banners
vol -f memory.raw windows.info.Info
vol -f memory.raw linux.pslist.PsList

# process and network triage
vol -f memory.raw windows.pslist.PsList
vol -f memory.raw windows.netscan.NetScan

# JSON output
vol -f memory.raw -r json windows.pslist.PsList > ps.json

# Linux plugins
vol -f memory.raw linux.bash.Bash
vol -f memory.raw linux.malfind.Malfind
```

## Flag reference

### Global

| Flag | Meaning |
|------|---------|
| `PLUGIN` | Plugin to run, e.g. `windows.pslist.PsList` |
| `-f, --file <file>` | Memory image |
| `-c, --config <json>` | Load configuration |
| `-e, --extend <k=v>` | Extend configuration |
| `-p, --plugin-dirs <paths>` | Additional plugin paths (`;`-separated) |
| `-s, --symbol-dirs <paths>` | Additional symbol paths |
| `-v` | Increase verbosity |
| `-l, --log <file>` | Log to file |
| `-o, --output-dir <dir>` | Output directory for generated files |
| `-q, --quiet` | Remove progress output |
| `--parallelism [processes\|threads\|off]` | Parallelism mode |
| `--filters <spec>` | Filter rows (`+col,pattern`) |
| `--hide-columns <cols>` | Hide columns by prefix |
| `-r, --renderer <fmt>` | `quick, none, csv, pretty, json, jsonl, mermaid` |
| `--single-location <uri>` | Base location for stacking |
| `--stackers <list>` | Stackers to use |
| `--single-swap-locations <uris>` | Swap layer URIs |
| `--cache-path <dir>` | Cache location (default `~/.cache/volatility3`) |
| `--clear-cache` | Clear short-term cache |
| `--offline` | Do not look up symbols online |
| `-u, --remote-isf-url <url>` | Custom ISF source |
| `--write-config` / `--save-config <file>` | Persist configuration |

### Plugin families (selection)

| Family | Examples |
|--------|----------|
| `banners`, `frameworkinfo`, `isfinfo`, `layerwriter`, `timeliner`, `vmscan`, `regexscan` | Framework utilities |
| `linux.pslist`, `linux.psscan`, `linux.pstree`, `linux.psaux`, `linux.proc.Maps` | Processes and memory |
| `linux.bash`, `linux.envars`, `linux.lsof`, `linux.library_list`, `linux.elfs` | User activity |
| `linux.malfind`, `linux.malware.*` | Injection and rootkits |
| `linux.pagecache.Files`, `RecoverFs`, `InodePages` | File recovery |
| `linux.sockstat`, `linux.sockscan`, `linux.ip.Addr` | Network |
| `windows.pslist`, `psscan`, `pstree`, `cmdline`, `envars`, `handles`, `dlllist` | Processes |
| `windows.netscan`, `netstat`, `malfind`, `ldrmodules`, `callbacks`, `ssdt` | Network and kernel |
| `windows.registry.*`, `windows.amcache`, `windows.scheduled_tasks` | Registry/artifacts |
| `windows.dumpfiles`, `windows.memmap`, `windows.pedump` | Extraction |

## Recipes

```bash
# 1. Identify the image type before anything else
vol -f memory.raw banners.Banners
vol -f memory.raw windows.info.Info

# 2. Windows process triage
vol -f memory.raw windows.pslist.PsList
vol -f memory.raw windows.psscan.PsScan
vol -f memory.raw windows.pstree.PsTree

# 3. Network state
vol -f memory.raw windows.netscan.NetScan

# 4. Injected code and hidden modules
vol -f memory.raw windows.malfind.Malfind
vol -f memory.raw windows.ldrmodules.LdrModules

# 5. Linux process and shell history
vol -f memory.raw linux.pslist.PsList
vol -f memory.raw linux.bash.Bash
vol -f memory.raw linux.envars.Envars

# 6. Recover files from page cache
vol -f memory.raw linux.pagecache.Files
vol -f memory.raw -o /evidence linux.pagecache.RecoverFs

# 7. JSON for correlation
vol -f memory.raw -r json windows.pslist.PsList > ps.json

# 8. Filter output without post-processing
vol -f memory.raw --filters "+Process,chrome" windows.pslist.PsList

# 9. Offline lab with pre-seeded symbols
vol --offline -s /opt/vol-symbols -f memory.raw linux.pslist.PsList

# 10. Timeline
vol -f memory.raw timeliner.Timeliner
```

## Output & pipelines

- Default renderer is a table; `-r json|jsonl|csv` for machine parsing.
- `-o` collects dumped files, extracted modules, and recovered filesystems.
- `timeliner` output feeds forensic timelines.
- JSON output feeds SIEM, `jq`, and correlation scripts.
- `windows.dumpfiles` and `linux.pagecache.RecoverFs` write evidence you must
  hash and store securely.

## Gotchas & operational security

- Plugin names are case-sensitive (`windows.pslist.PsList`).
- The framework may download ISF symbol files; use `--offline` and a seeded
  symbol dir in air-gapped or sensitive environments.
- Several plugins in this build failed to load (see the tail of `help.txt`),
  usually due to missing optional Python packages. Check `vol -vv` when a
  plugin is unavailable.
- Memory images can be huge; analysis is I/O- and RAM-intensive.
- `malfind` flags executable/private memory but produces false positives on
  JIT engines and packed software.
- Volatility parses untrusted memory; run it in an isolated analysis VM.
- Extracted files may be malware; never execute them on the analysis host.
- Memory contains credentials and personal data; handle per policy.

## Legal & safety

Analyze only memory images you are authorized to possess and examine. RAM
captures contain credentials, personal data, and malware; store them encrypted,
limit access, and follow chain-of-custody rules.

## Related tools

- [`exiftool`](https://exiftool.org/) — metadata on extracted files
- `strings`, `binwalk` — quick image triage
- [`testssl/`](../testssl/README.md), [`sslyze/`](../sslyze/README.md) — not
  related but often in the same IR toolkit
- `yara` — scan extracted memory and files

## References

- https://github.com/volatilityfoundation/volatility3
- https://volatility3.readthedocs.io/
- https://github.com/volatilityfoundation/volatility3/wiki
