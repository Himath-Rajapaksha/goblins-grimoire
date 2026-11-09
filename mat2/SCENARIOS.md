# mat2 — command scenarios for deep research

Installed build: mat2 0.13.5 (Debian `0.13.5-1.2`), 36 supported MIME types.
Every flag below appears in [`help.txt`](help.txt); output naming comes from
`libmat2/abstract.py` (`output_filename = fname + '.cleaned' + extension`) and
`clean_meta()` in `/usr/bin/mat2`.

> **Scope:** the recipes create and clean throwaway files. Metadata can carry
> personal data and device identifiers; do not process third-party media
> without authorisation.

> **Destructive warning:** `--inplace` renames the cleaned file over the
> original with **no backup**. The default mode leaves the input untouched but
> writes a predictable `<stem>.cleaned<ext>` path, which is symlink-raceable.
> Keep the demo tree private and delete it when finished.

## How to invoke

```text
@deep-research For an authorised metadata-anonymisation task with mat2
0.13.5, explain what each supported format parser removes, how the default
`<stem>.cleaned<ext>` output and `--inplace` differ on disk, and how
`--lightweight`, `--show`, and `--unknown-members` change the result. Assess
the symlink risk created by the predictable output name, the bubblewrap
sandbox boundary, and the gap between "metadata removed" and "anonymous".
Cite the mat2 help text, `/usr/share/doc/mat2/threat_model.md`,
`implementation_notes.md`, and observed output. Separate confirmed behaviour
from inference and state confidence.
```

Create the demo tree and the base images:

```bash
mkdir -p ~/mat2-demo/evidence
cd ~/mat2-demo
date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt
python3 - <<'PY'
from PIL import Image
Image.new('RGB', (32, 24), (200, 10, 10)).save('pic.jpg', quality=90)
Image.new('RGB', (32, 24), (10, 200, 10)).save('pic.png')
PY
exiftool -Artist='Alice' -Comment='secret' pic.jpg >/dev/null
exiftool -Artist='Alice' -Comment='secret' pic.png >/dev/null
sha256sum pic.jpg pic.png > evidence/before.sha256
```

## Scenario 1 — version, formats, dependencies

| # | Command | Purpose |
|---|---------|---------|
| 1 | `mat2 --version` | Print the version |
| 2 | `mat2 --help` | Show the full option surface |
| 3 | `mat2` | No arguments: usage, exit 0 |
| 4 | `mat2 --list` | List the 36 supported MIME types |
| 5 | `mat2 --list \| grep -c '^  - '` | Count the supported formats |
| 6 | `mat2 --check-dependencies` | Report optional dependency availability |
| 7 | `command -v mat2` | Resolve the entry point |
| 8 | `file /usr/bin/mat2` | Confirm it is a Python script |
| 9 | `dpkg -s mat2 \| grep -E 'Version\|Depends'` | Package provenance |
| 10 | `ls /usr/lib/python3/dist-packages/libmat2/` | List the parser modules |
| 11 | `sed -n '1,60p' /usr/share/doc/mat2/threat_model.md` | Read the threat model |
| 12 | `date -u +'%Y-%m-%dT%H:%M:%SZ' > evidence/run-start.txt` | Timestamp the run |

**Research questions**
- Which formats are supported, and which need optional dependencies?
- Is the sandbox always available, or can it be absent?
- What does mat2 explicitly say it does *not* protect against?
- Which parser modules correspond to which format families?

**Sources:** `mat2 --list`; `--check-dependencies`; `threat_model.md`.
**Output:** a format/dependency inventory and the threat-model boundary.

## Scenario 2 — show, default cleaning, lightweight mode

| # | Command | Purpose |
|---|---------|---------|
| 13 | `mat2 --show pic.jpg` | List harmful metadata without removing it |
| 14 | `mat2 --show pic.png` | Same for PNG |
| 15 | `mat2 pic.jpg` | Default: write `pic.cleaned.jpg` |
| 16 | `ls -l pic.jpg pic.cleaned.jpg` | Confirm the input is untouched |
| 17 | `exiftool -Artist -Comment pic.cleaned.jpg` | Confirm the tags are gone |
| 18 | `mat2 --lightweight pic.png` | Remove only superficial metadata |
| 19 | `ls -l pic.png pic.cleaned.png` | Confirm a cleaned copy exists |
| 20 | `exiftool -Artist -Comment pic.cleaned.png` | Check the lightweight result |
| 21 | `sha256sum pic.jpg pic.cleaned.jpg` | Hashes differ because of re-encoding |
| 22 | `cmp -s pic.jpg pic.cleaned.jpg; echo "cmp=$?"` | Confirm the bytes changed |
| 23 | `file pic.cleaned.jpg` | Confirm the output is still a JPEG |
| 24 | `python3 -c "from PIL import Image; print(Image.open('pic.cleaned.jpg').size)"` | Confirm decodability |
| 25 | `mat2 --show pic.cleaned.jpg` | Re-inspect the cleaned file |
| 26 | `exiftool -G1 -a -s pic.cleaned.jpg \| head -12` | Look for surviving tags |

**Research questions**
- Does the default mode modify the input?
- What exactly does `--lightweight` skip?
- Are images re-encoded even when no metadata is found?
- Which tags survive a normal clean?

**Sources:** `mat2 --show`; observed file output; `implementation_notes.md`.
**Output:** a before/after metadata and hash comparison.

## Scenario 3 — in-place cleaning and backup semantics

| # | Command | Purpose |
|---|---------|---------|
| 27 | `cp pic.jpg inplace.jpg` | Prepare a disposable copy |
| 28 | `mat2 --inplace inplace.jpg` | Replace the input, no backup |
| 29 | `ls -l inplace*` | Confirm no `_original` or `.cleaned` file |
| 30 | `exiftool -Artist -Comment inplace.jpg` | Confirm the metadata is gone |
| 31 | `cp pic.jpg keep.jpg` | Prepare a default-mode copy |
| 32 | `mat2 keep.jpg` | Keep the original, write a cleaned copy |
| 33 | `ls -l keep*` | Confirm the pair exists |
| 34 | `stat -c '%n %i %a %U' pic.jpg keep.jpg keep.cleaned.jpg` | Compare inodes and modes |
| 35 | `cp pic.jpg mode.jpg && chmod 640 mode.jpg` | Set a distinctive mode |
| 36 | `mat2 mode.jpg && stat -c '%a' mode.cleaned.jpg` | Check mode inheritance |
| 37 | `cp pic.jpg own.jpg && mat2 --inplace own.jpg` | In-place path on a copy |
| 38 | `stat -c '%n %i %s' pic.jpg own.jpg` | Confirm the inode changed |

**Research questions**
- Does `--inplace` leave any backup at all?
- Does the cleaned file inherit the input's mode?
- Which inode survives an in-place clean?
- Is `--inplace` safe on a file with hard links?

**Sources:** `clean_meta()` in `/usr/bin/mat2`; observed `stat` output.
**Output:** an inode/mode/backup table for both modes.

## Scenario 4 — documents: DOCX, PDF, EPUB

| # | Command | Purpose |
|---|---------|---------|
| 39 | `printf '# Title\n\nBody text.\n' > doc.md` | Source document |
| 40 | `pandoc doc.md -o doc.docx` | Build a DOCX with properties |
| 41 | `mat2 --show doc.docx` | Show per-member ZIP metadata |
| 42 | `mat2 doc.docx` | Clean the DOCX |
| 43 | `ls -l doc.docx doc.cleaned.docx` | Confirm the pair |
| 44 | `unzip -l doc.cleaned.docx` | Inspect the rebuilt package |
| 45 | `exiftool doc.cleaned.docx \| grep -iE 'author\|creator\|lastmodified'` | Look for surviving properties |
| 46 | `printf '<html><head><meta name="author" content="Alice"><title>T</title></head><body><p>Hello</p></body></html>\n' > page.html` | Source HTML |
| 47 | `weasyprint page.html page.pdf` | Build a PDF with an author |
| 48 | `mat2 --show page.pdf` | Show PDF metadata |
| 49 | `mat2 page.pdf` | Clean the PDF (rasterises it) |
| 50 | `pdfinfo page.pdf \| head -12` | Original PDF info |
| 51 | `pdfinfo page.cleaned.pdf \| head -12` | Cleaned PDF info |
| 52 | `pdftotext page.pdf - \| head -3` | Original text is extractable |
| 53 | `pdftotext page.cleaned.pdf - \| head -3` | Cleaned text is gone |

**Research questions**
- Does DOCX cleaning preserve the document parts?
- Does PDF cleaning preserve searchable text?
- Which PDF fields are removed, and which are rewritten?
- Is the cleaned PDF still a valid PDF?

**Sources:** `mat2 --show`; `pdfinfo`; `pdftotext`; observed output.
**Output:** a document-format before/after record.

## Scenario 5 — archives and unknown-member policies

| # | Command | Purpose |
|---|---------|---------|
| 54 | `zip -q arch.zip pic.jpg doc.md` | Build a ZIP with one unsupported member |
| 55 | `tar cf arch.tar pic.jpg doc.md` | Build a TAR with the same members |
| 56 | `mat2 arch.zip` | Default `abort` policy |
| 57 | `ls -l arch.zip arch.cleaned.zip 2>&1` | Confirm no cleaned ZIP |
| 58 | `mat2 --unknown-members omit arch.zip` | Drop the unsupported member |
| 59 | `unzip -l arch.cleaned.zip` | Inspect the rebuilt archive |
| 60 | `mat2 --unknown-members keep arch.zip` | Keep the member, accept leakage |
| 61 | `unzip -l arch.cleaned.zip` | Compare the two policies |
| 62 | `mat2 --unknown-members abort arch.zip` | Explicit abort |
| 63 | `mat2 arch.tar` | TAR abort on the unsupported member |
| 64 | `mat2 --unknown-members omit arch.tar` | TAR omit policy |
| 65 | `tar tf arch.cleaned.tar` | Inspect the cleaned TAR |
| 66 | `zipinfo arch.cleaned.zip \| head -8` | Compare member timestamps |
| 67 | `unzip -p arch.cleaned.zip pic.jpg \| exiftool - \| head -8` | Check the cleaned member's metadata |

**Research questions**
- Which policy is the safe default, and why?
- Does `keep` preserve the original member bytes?
- Are archive timestamps normalised?
- Does `omit` silently lose data the user expected?

**Sources:** `mat2 --unknown-members`; `zipinfo`; observed warnings.
**Output:** a three-policy archive comparison.

## Scenario 6 — EPUB and recursive directories

| # | Command | Purpose |
|---|---------|---------|
| 68 | `printf '# Book\n\nText.\n' > book.md` | Source for an EPUB |
| 69 | `pandoc book.md -o book.epub` | Build an EPUB with extra members |
| 70 | `mat2 book.epub` | Default abort on the Apple display-options member |
| 71 | `ls -l book.epub book.cleaned.epub 2>&1` | Confirm no cleaned EPUB |
| 72 | `mat2 --unknown-members omit book.epub` | Drop the unknown member |
| 73 | `ls -l book.epub book.cleaned.epub` | Confirm the cleaned EPUB |
| 74 | `unzip -l book.cleaned.epub` | Inspect the package |
| 75 | `mkdir -p tree/sub && cp pic.jpg tree/a.jpg && cp pic.png tree/sub/b.png` | Build a tree |
| 76 | `mat2 tree` | Recurse and write `.cleaned` files |
| 77 | `find tree -type f \| sort` | Show the output layout |
| 78 | `mat2 tree` | Run again on the already-cleaned tree |
| 79 | `find tree -name '*cleaned.cleaned*'` | Show the double-cleaning quirk |
| 80 | `find tree -name '*.cleaned.*' -delete` | Remove generated copies |
| 81 | `mat2 --inplace tree && find tree -type f \| sort` | In-place recursive clean |

**Research questions**
- Does a directory argument recurse by default?
- What happens on a second run over the same tree?
- Does `--inplace` preserve the relative layout?
- Which EPUB members are considered unknown?

**Sources:** `mat2` observed output; `unzip -l`.
**Output:** a recursive-clean layout and the double-cleaning caveat.

## Scenario 7 — symlink risk, sandbox, verification

| # | Command | Purpose |
|---|---------|---------|
| 82 | `mkdir -p sym && cp pic.png sym/p.png` | Prepare a private directory |
| 83 | `printf 'ORIGINAL\n' > sym/target.txt` | Create a victim file |
| 84 | `ln -s target.txt sym/p.cleaned.png` | Plant a predictable output symlink |
| 85 | `mat2 sym/p.png` | Write through the symlink |
| 86 | `file sym/target.txt` | Show the victim was overwritten |
| 87 | `cat sym/target.txt \| head -c 16 \| xxd` | Show PNG bytes in the victim |
| 88 | `mat2 --no-sandbox pic.jpg` | Disable bubblewrap for a trusted file |
| 89 | `mat2 --show pic.cleaned.jpg` | Re-inspect the cleaned image |
| 90 | `exiftool -G1 -a -s pic.cleaned.jpg \| head -12` | Look for residual tags |
| 91 | `strings pic.cleaned.jpg \| grep -i alice; echo "alice=$?"` | Search for the author string |
| 92 | `strings pic.cleaned.jpg \| grep -i secret; echo "secret=$?"` | Search for the comment |
| 93 | `stat -c '%n %y' pic.jpg pic.cleaned.jpg` | Compare filesystem timestamps |

**Research questions**
- Can a predictable output name overwrite an arbitrary file?
- What does the sandbox isolate, and what does `--no-sandbox` expose?
- Do strings survive in the re-encoded image?
- Are filesystem timestamps changed?

**Sources:** observed symlink test; `threat_model.md`; `strings`.
**Output:** a demonstrated symlink write and a residual-metadata check.

## Scenario 8 — unsupported input, exit codes, cleanup

| # | Command | Purpose |
|---|---------|---------|
| 94 | `printf 'hello\n' > note.txt` | A supported plain-text file |
| 95 | `mat2 note.txt` | Clean the text file |
| 96 | `ls -l note.txt note.cleaned.txt` | Confirm the output |
| 97 | `printf '\x00\x01\x02' > blob.bin` | An unsupported binary |
| 98 | `mat2 blob.bin` | Unsupported: message, no output |
| 99 | `ls -l blob*` | Confirm no cleaned file |
| 100 | `mat2 /nonexistent` | Missing path |
| 101 | `mat2 /nonexistent >/dev/null 2>&1; echo "exit=$?"` | Confirm exit 255 |
| 102 | `mkdir -p empty-dir && mat2 empty-dir` | Empty directory: walked, exit 0 |
| 103 | `mat2 empty-dir >/dev/null 2>&1; echo "exit=$?"` | Confirm the recursive success |
| 104 | `mat2 >/dev/null 2>&1; echo "exit=$?"` | No arguments exits 0 |
| 105 | `rm -rf ~/mat2-demo` | Remove the demo tree |
| 106 | `test ! -e ~/mat2-demo; echo "removed=$?"` | Verify cleanup |

**Research questions**
- Which failures are silent successes?
- Does an unsupported file stop a directory walk?
- Is exit 255 distinguishable from a normal error?
- What evidence proves the demo tree is gone?

**Sources:** observed exit codes; `clean_meta()`.
**Output:** an exit-code table and a cleaned workspace.

## Flag coverage checklist

| Group / flag | Scenario |
|--------------|----------|
| `-h`, `--help` | 1 |
| `-v`, `--version` | 1 |
| `-l`, `--list` | 1 |
| `--check-dependencies` | 1 |
| `-s`, `--show` | 2, 4, 7 |
| `-L`, `--lightweight` | 2 |
| `--inplace` | 3, 6 |
| `--unknown-members abort\|omit\|keep` | 5, 6 |
| `--no-sandbox` | 7 |
| `-V`, `--verbose` | 2 (implicit in status output) |

Full text: [`help.txt`](help.txt); design notes in
`/usr/share/doc/mat2/implementation_notes.md` and `threat_model.md`.

## Scenario index

| Scenario | Task | Commands |
|----------|------|----------|
| 1 | Version, formats, dependencies | 12 |
| 2 | Show, default cleaning, lightweight mode | 14 |
| 3 | In-place cleaning and backup semantics | 12 |
| 4 | Documents: DOCX, PDF, EPUB | 15 |
| 5 | Archives and unknown-member policies | 14 |
| 6 | EPUB and recursive directories | 14 |
| 7 | Symlink risk, sandbox, verification | 12 |
| 8 | Unsupported input, exit codes, cleanup | 13 |
| **Total** | | **106** |

## Legal and safety

mat2 is an anonymisation aid, not a guarantee. It does not remove watermarks,
steganography, visible identifiers, or stylometric traces, and it does not
change filesystem timestamps or ownership. The predictable `.cleaned` output
name is a real symlink hazard: demonstrated on this host by overwriting an
unrelated file through a planted symlink. Clean untrusted trees in a private
directory, keep backups before `--inplace`, and validate the result with
`exiftool` before publishing. Delete the demo tree when finished.
