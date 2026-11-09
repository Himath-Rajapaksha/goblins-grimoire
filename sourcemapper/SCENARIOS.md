# sourcemapper — command scenarios for deep research

Installed build: sourcemapper from upstream `denandz/sourcemapper`; the binary
exposes no version flag. Flags come from `help.txt`.

## How to invoke

```text
@deep-research For sourcemapper in 2026, how often do production webpack, Vite,
and esbuild bundles still ship source maps, which discovery path (-url, -jsurl,
-dir) gives the most reliable extraction, and how should the upstream
arbitrary-URL retrieval behaviour be contained? Cite the source-map spec, the
Pulse Security article, and bug-bounty reports; return a decision tree and a
containment checklist.
```

---

## Scenario 1 — direct map extraction

| # | Command | How to use |
|---|---------|------------|
| 1 | `sourcemapper -help` | Confirm usage |
| 2 | `sourcemapper -url https://example.com/main.js.map -output ./src` | URL mode |
| 3 | `sourcemapper -url ./main.js.map -output ./src-local` | Local file mode |
| 4 | `find ./src -type f \| head` | Inspect tree |
| 5 | `du -sh ./src` | Measure output |

**Research questions**
- Does the map use absolute, relative, or webpack-style source paths?
- Which files are original source versus generated glue?

**Sources:** source-map spec, extraction log, output tree.
**Output:** a recovered source tree and size profile.

---

## Scenario 2 — discovery from a JavaScript file

| # | Command | How to use |
|---|---------|------------|
| 6 | `sourcemapper -jsurl https://example.com/app.js -output ./src-js` | Auto-discovery |
| 7 | `curl -s https://example.com/app.js \| grep -o 'sourceMappingURL=[^ ]*'` | Inspect reference |
| 8 | `sourcemapper -jsurl https://example.com/app.js -output ./src-inline` | Inline `data:` map |
| 9 | `curl -sI https://example.com/app.js.map` | Check map availability |
| 10 | `sourcemapper -jsurl https://example.com/app.js -output ./src-retry` | Repeat for reproducibility |

**Research questions**
- Which `sourceMappingURL` forms appear (absolute, relative, `data:`)?
- Does the bundle advertise a map that the server refuses to serve?

**Sources:** bundle contents, HTTP headers, source-map linking spec.
**Output:** the map discovery method that works for this target.

---

## Scenario 3 — authenticated and proxied retrieval

| # | Command | How to use |
|---|---------|------------|
| 11 | `sourcemapper -header 'Cookie: session=LAB' -url ... -output ./src` | Cookie auth |
| 12 | `sourcemapper -header 'Authorization: Bearer LAB' -url ... -output ./src` | Bearer auth |
| 13 | `sourcemapper -proxy http://127.0.0.1:8080 -url ... -output ./src` | Intercept via proxy |
| 14 | `sourcemapper -insecure -jsurl https://127.0.0.1:8443/app.js -output ./src` | Lab TLS |
| 15 | `sourcemapper -header 'A: 1' -header 'B: 2' -url ... -output ./src` | Repeated headers |

**Research questions**
- Which headers are required for the map but absent from the bundle request?
- How should session material be kept out of shell history and process lists?

**Sources:** HTTP responses, proxy history, auth documentation.
**Output:** authenticated extraction without credential leakage.

---

## Scenario 4 — batch processing and comparison

| # | Command | How to use |
|---|---------|------------|
| 16 | `mkdir -p maps && cp ./*.map maps/` | Collect maps |
| 17 | `sourcemapper -dir ./maps -output ./all-src` | Batch mode |
| 18 | `find ./all-src -maxdepth 2 -type d \| sort` | Review structure |
| 19 | `sourcemapper -dir ./maps-v2 -output ./all-src-v2` | Second release |
| 20 | `diff -ru ./all-src ./all-src-v2 \| head -100` | Compare releases |

**Research questions**
- How are bad or empty maps reported, and does one failure stop the batch?
- Which source changes between builds indicate feature or security work?

**Sources:** batch logs, diff output, release metadata.
**Output:** a cross-version source comparison.

---

## Scenario 5 — containment and evidence

| # | Command | How to use |
|---|---------|------------|
| 21 | `unshare -n -- sourcemapper -url ./local.map -output ./src-isolated` | Offline isolation |
| 22 | `grep -rniE 'api[_-]?key\|secret\|token' ./src \| head` | Sensitive strings |
| 23 | `find ./src -type f -print0 \| sort -z \| xargs -0 sha256sum > src.sha256` | Hash tree |
| 24 | `tar -czf src-evidence.tgz ./src src.sha256` | Package evidence |
| 25 | `sha256sum ./local.map src-evidence.tgz` | Hash map and bundle |

**Research questions**
- Which URLs did the tool request, and were any outside the target?
- What must be logged to prove the extraction stayed in scope?

**Sources:** proxy logs, hashes, egress logs.
**Output:** a contained, hashed evidence package.

## Flag coverage checklist

| Flag | Scenario |
|------|----------|
| `-url` | 1, 3, 5 |
| `-jsurl` | 2, 3 |
| `-dir` | 4 |
| `-output` | 1–5 |
| `-header` | 3 |
| `-proxy` | 3 |
| `-insecure` | 3 |
| `-help` | 1 |

## Scenario index

| Scenario | Commands |
|----------|----------|
| 1 — direct map | 5 |
| 2 — discovery | 5 |
| 3 — auth/proxy | 5 |
| 4 — batch/diff | 5 |
| 5 — containment | 5 |
