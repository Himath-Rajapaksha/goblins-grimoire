# metagoofil — document metadata harvesting

- **Version:** metagoofil 1.5.0 (opsdisk Python 3 fork)
- **Binary:** `/usr/local/bin/metagoofil`
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/opsdisk/metagoofil

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

metagoofil searches for documents published by a target domain, downloads the
selected file types, and extracts metadata from them — author names, email
addresses, software versions, paths, printer names, and creation dates. Old
Office and PDF files frequently leak internal usernames, machine names, and
network paths that are useful for username generation and social engineering.

## Use cases

| Situation | Why metagoofil |
|-----------|----------------|
| Username harvesting | Author fields reveal internal naming patterns |
| Software fingerprinting | Producer fields expose Office/PDF versions |
| Path disclosure | Metadata leaks internal shares and directories |
| Social engineering prep | Names, emails, and roles for pretexting |
| Document inventory | See what the organization publishes |
| OPSEC review | Show clients what their own files leak |

## Requirements & install layout

- Installed via pipx from the opsdisk fork; entry point `/usr/local/bin/metagoofil`.
- Python 3 environment; uses `googlesearch` and `requests`.
- No API key; Google may block or CAPTCHA aggressive use.
- Searches are rate-limited by `-e`; keep the delay respectful.
- Downloads land in `./data` unless `-o` is set.

## Basic syntax

```bash
# search and list PDF/DOC/XLS/PPT links (no download)
metagoofil -d example.com -t pdf,doc,xls,ppt

# download and extract metadata
metagoofil -d example.com -t pdf,docx,xlsx,pptx -w

# save links and download to a directory
metagoofil -d example.com -t pdf -f links.txt -o evidence -w

# all 17,576 three-letter extensions (slow, noisy)
metagoofil -d example.com -t ALL -w
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-d <domain>` | Domain to search (required) |
| `-t <types>` | File types: `pdf,doc,xls,ppt,odp,ods,docx,xlsx,pptx` or `ALL` |
| `-w` | Download files, not just list links |
| `-f [file]` | Save HTML links; no argument auto-names |
| `-o <dir>` | Download directory (default `./data`) |
| `-l <n>` | Max search results (default 100) |
| `-n <n>` | Max downloads per file type (default 100) |
| `-e <sec>` | Delay between searches (default 30) |
| `-i <sec>` | URL timeout (default 15) |
| `-r <n>` | Downloader threads (default 8) |
| `-u [ua]` | User-Agent; default randomises |
| `-h, --help` | Help |

## Recipes

```bash
# 1. Recon only: see what is published
metagoofil -d example.com -t pdf,doc,xls,ppt -l 50

# 2. Save the link list, do not download
metagoofil -d example.com -t pdf -f links.txt -l 100

# 3. Download and extract metadata
metagoofil -d example.com -t pdf,docx,xlsx,pptx -w -o evidence

# 4. Polite, slow run against a sensitive target
metagoofil -d example.com -t pdf -w -e 60 -l 30 -n 10 -r 2

# 5. Broad extension sweep (expect noise and blocks)
metagoofil -d example.com -t ALL -w -l 200

# 6. Custom user agent for a lab
metagoofil -d example.com -t pdf -w -u "Mozilla/5.0 (X11; Linux x86_64)"

# 7. Extract text metadata with exiftool afterwards
exiftool -r -Author -Creator -Producer -CompanyName evidence/ > metadata.txt

# 8. Pull email addresses from the metadata report
grep -Eio '[a-z0-9._%+-]+@[a-z0-9.-]+\.[a-z]{2,}' metadata.txt | sort -u
```

## Output & pipelines

- Console report lists each file and its extracted metadata fields.
- `-f` writes the discovered links; `-o` stores downloads.
- `-w` triggers metadata extraction after download.
- Authors/emails feed username generators (`kerbrute`, `netexec`) and
  phishing pretexts.
- Software versions feed vulnerability research.

## Gotchas & operational security

- Google custom search via `googlesearch` breaks when Google changes markup or
  serves CAPTCHAs; expect empty runs.
- The `ALL` extension mode is extremely slow and noisy.
- Downloaded documents can contain malware; handle in an isolated VM and never
  open them on the assessment host.
- Metadata parsing of legacy `.doc`/`.xls` needs `antiword`/`catdoc` helpers;
  missing helpers yield incomplete results.
- Author fields are frequently the original template author, not the target.
- Respect copyright: downloading documents is not the same as being licensed
  to read them.

## Legal & safety

Only harvest documents that are publicly accessible and within scope. Metadata
is personal data; store it securely and retain it only as long as the
engagement requires. Do not redistribute obtained documents.

## Related tools

- [`theharvester/`](../theharvester/README.md), [`spiderfoot/`](../spiderfoot/README.md)
- [`recon-ng/`](../recon-ng/README.md)
- [`sherlock/`](../sherlock/README.md), [`holehe/`](../holehe/README.md)
- [`exiftool`](https://exiftool.org/) — deeper metadata extraction

## References

- https://github.com/opsdisk/metagoofil
- https://github.com/laramies/metagoofil (original)
