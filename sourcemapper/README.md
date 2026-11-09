# sourcemapper — recover original source from JavaScript source maps

- **Version:** no version flag in the installed build; built from upstream
  `denandz/sourcemapper`
- **Binary:** `/usr/local/bin/sourcemapper` (Go)
- **Help snapshot:** [`help.txt`](help.txt)
- **Upstream:** https://github.com/denandz/sourcemapper
- **Article:** https://pulsesecurity.co.nz/articles/javascript-from-sourcemaps

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

sourcemapper parses a JavaScript source map (version 3, as emitted by webpack,
Vite, esbuild, and similar bundlers) and writes the original source files to
disk, recreating the directory tree recorded in the map. Modern bundles often
ship a `.map` file, either as a sibling file or embedded as a `data:` URL, and
that map can contain the unminified application code, internal module paths,
comments, and occasionally secrets.

The tool accepts three input modes: a direct map URL or local map path
(`-url`), a JavaScript URL whose `sourceMappingURL` reference is discovered
automatically (`-jsurl`), or a directory of maps processed recursively
(`-dir`). Exactly one of those three may be used per run.

## Use cases

| Situation | Why sourcemapper |
|-----------|------------------|
| A production bundle ships a `.map` | Recover readable application source |
| Reviewing client-side logic | Read the pre-minification modules |
| Finding internal paths | Source trees expose module structure |
| Comparing builds | Extract two versions and diff the trees |
| Batch triage | Walk a directory of downloaded maps |
| Inline maps | Decode `data:` source maps via `-jsurl` |
| Evidence for a report | Preserve extracted source with hashes |

## Requirements & install layout

- Go binary at `/usr/local/bin/sourcemapper`; no runtime dependencies.
- No configuration file.
- `-output` is mandatory; the directory is created if needed.
- Exactly one of `-url`, `-jsurl`, or `-dir` must be supplied.
- `-header` is repeatable and applies to map and JavaScript requests.
- `-proxy` accepts a proxy URL; `-insecure` disables TLS verification.
- Large maps are held in memory while parsing; a multi-megabyte map can need
  several times its size in RAM.

## Basic syntax

```bash
# direct map URL
sourcemapper -url https://example.com/static/js/main.js.map -output ./src

# discover the map from a JavaScript file
sourcemapper -jsurl https://example.com/static/js/main.js -output ./src

# local map file
sourcemapper -url ./main.js.map -output ./src-local

# batch a directory of maps
sourcemapper -dir ./maps -output ./all-src
```

## Flag reference

| Flag | Meaning |
|------|---------|
| `-url <url\|path>` | URL or local path to the source map |
| `-jsurl <url>` | URL of a JavaScript file; locate and fetch its referenced map |
| `-dir <path>` | Directory of `.map` files to process recursively |
| `-output <dir>` | Source output directory; **required** |
| `-header <name: value>` | Header for requests; repeatable (cookies, auth) |
| `-proxy <url>` | Proxy URL for outbound requests |
| `-insecure` | Ignore invalid TLS certificates |
| `-help` | Show usage |

There is no version flag; record the binary hash when provenance matters.

## Recipes

```bash
# 1. Direct map extraction
sourcemapper -url https://example.com/static/js/main.js.map -output ./src-main

# 2. Let sourcemapper find the map reference in the bundle
sourcemapper -jsurl https://example.com/static/js/main.js -output ./src-main

# 3. Authenticated map fetch
sourcemapper -header 'Cookie: session=LAB_VALUE' \
             -header 'Authorization: Bearer LAB_TOKEN' \
             -url https://example.com/static/js/main.js.map -output ./src-auth

# 4. Self-signed lab endpoint through a proxy
sourcemapper -insecure -proxy http://127.0.0.1:8080 \
             -jsurl https://127.0.0.1:8443/app.js -output ./src-lab

# 5. Batch every map in a download directory
sourcemapper -dir ./maps -output ./all-src

# 6. Local map that was extracted from a data: URL
sourcemapper -url ./decoded.js.map -output ./src-decoded

# 7. Preserve raw output and review the tree
find ./src-main -type f | sort | head -50
du -sh ./src-main

# 8. Hash the extracted source for evidence
find ./src-main -type f -print0 | sort -z | xargs -0 sha256sum > src.sha256
```

## Output & pipelines

- Writes original files under `-output`, following the source paths recorded
  in the map. Webpack maps frequently produce a `webpack:/` directory whose
  colon is escaped by the shell; quote paths containing `:`.
- stdout/stderr report retrieval progress, map version, entry count, and bytes
  written.
- Extracted trees feed `grep`, `semgrep`, `trufflehog`, and manual review.
- Use `diff -ru` between two extractions to compare releases.
- Keep the original map file and its hash alongside the extracted tree; the map
  is the authoritative artefact.

## Gotchas & operational security

- The upstream README warns that sourcemapper retrieves **any URL referenced
  as a source map**. A malicious JavaScript file can therefore make the tool
  request an arbitrary URL, including internal addresses. Treat this as an SSRF
  primitive and run it only against trusted targets or inside an isolated
  network.
- Source paths come from the map and may contain unexpected segments. Extract
  into a disposable directory and inspect before copying anything elsewhere.
- Existing files in the output directory may be overwritten.
- A 404 or HTML error page is not a source map; the tool will fail to parse it.
- `data:` maps are best handled with `-jsurl`, which follows the source-map
  spec's linking rules.
- Memory use scales with map size; very large maps can exhaust a small VM.
- `-insecure` disables certificate and hostname checks; never use it against
  third parties.
- A successful extraction does not prove the source is current; compare hashes
  and build metadata.
- Recovered source may contain credentials, internal hostnames, and personal
  data. Handle it as sensitive client material.

## Legal & safety

Use sourcemapper only against applications you own or are authorized to test.
Downloading and reconstructing proprietary source may breach licence terms even
when the map is publicly served. Do not run it against third-party sites, and
isolate it from internal networks because of the arbitrary-URL retrieval
behaviour described above.

## Related tools

- [`subjs/`](../subjs/README.md) — discover JavaScript URLs first
- [`linkfinder/`](../linkfinder/README.md) — endpoints in minified bundles
- [`secretfinder/`](../secretfinder/README.md) — secrets in JavaScript
- [`httpx/`](../httpx/README.md) — verify map and bundle availability
- `trufflehog`, `semgrep` — scan the recovered tree

## References

- https://github.com/denandz/sourcemapper
- https://pulsesecurity.co.nz/articles/javascript-from-sourcemaps
- https://tc39.es/source-map-spec/
