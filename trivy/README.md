# Trivy — vulnerability / misconfiguration / secret scanner

- **Version:** 0.76.2
- **Binary:** `/usr/local/bin/trivy`
- **Help snapshots:** [`help.txt`](help.txt) plus
  `help-image.txt`, `help-filesystem.txt`, `help-repository.txt`,
  `help-rootfs.txt`, `help-sbom.txt`, `help-kubernetes.txt`,
  `help-config.txt`, `help-vm.txt`, `help-server.txt`, `help-clean.txt`,
  `help-convert.txt`, `help-module.txt`, `help-plugin.txt`,
  `help-vex-repo.txt`, `help-registry.txt`
- **Docs:** https://trivy.dev/latest/docs/
- **Source:** https://github.com/aquasecurity/trivy

> **Deep-research scenarios:** [`SCENARIOS.md`](SCENARIOS.md)

## What it does

Trivy scans container images, filesystems, Git repositories, rootfs, SBOMs,
Kubernetes clusters, VMs, and IaC for:
- **Vulnerabilities** in OS packages and language dependencies,
- **Misconfigurations** (Dockerfile, Kubernetes, Terraform, Helm, CloudFormation, Ansible),
- **Secrets** (API keys, tokens, private keys),
- **Licenses**.

It downloads OCI-hosted vulnerability databases and checks bundles.

## Use cases

| Situation | Command |
|-----------|---------|
| Scan a container image | `trivy image nginx:latest` |
| Scan a project directory | `trivy fs .` |
| Scan a remote Git repo | `trivy repo https://github.com/org/repo` |
| Scan IaC only | `trivy config .` |
| Scan a Kubernetes cluster | `trivy k8s --report summary cluster` |
| Scan an SBOM | `trivy sbom sbom.json` |
| Secrets only | `trivy fs --scanners secret .` |
| CI gate on HIGH/CRITICAL | `--severity HIGH,CRITICAL --exit-code 1` |
| Air-gapped | `--offline-scan --skip-db-update` |

## Commands

| Command | Purpose |
|---------|---------|
| `trivy image` | Container image scan |
| `trivy filesystem` (`fs`) | Local directory/filesystem |
| `trivy repository` (`repo`) | Remote Git repository |
| `trivy rootfs` | Unpacked root filesystem |
| `trivy sbom` | SBOM scan |
| `trivy config` | IaC misconfiguration scan |
| `trivy kubernetes` (`k8s`) | Cluster scan |
| `trivy vm` | VM image scan (experimental) |
| `trivy server` / `client` | Client/server mode |
| `trivy module` / `plugin` / `vex` | Management/extension |
| `trivy clean` / `convert` | Cache cleanup / report conversion |

> There is no standalone `trivy secret` or `trivy license` command. Use
> `--scanners secret` or `--scanners license` with `fs`/`image`/`repo`.

## Common flags (most subcommands)

### Report
| Flag | Meaning |
|------|---------|
| `-f, --format <fmt>` | `table` (default), `json`, `sarif`, `cyclonedx`, `spdx`, `spdx-json`, `github`, `template` |
| `-o, --output <file>` | Write report |
| `-s, --severity <list>` | `UNKNOWN,LOW,MEDIUM,HIGH,CRITICAL` |
| `--exit-code <n>` | Exit code when findings exist (CI gating) |
| `--ignore-unfixed` | Hide vulns with no fix |
| `--vuln-type <list>` | `os`, `library` |
| `--scanners <list>` | `vuln,misconfig,secret,license` |
| `--list-all-pkgs` | Include all packages in JSON |
| `--dependency-tree` | Show dependency paths |
| `--ignorefile <path>` | `.trivyignore`-style file |
| `--show-suppressed` | Include suppressed findings |
| `--pkg-types <list>` | `os`, `library` |
| `--detection-priority` | `precise` or `comprehensive` |

### Scan behavior
| Flag | Meaning |
|------|---------|
| `--skip-dirs <glob>` | Exclude directories |
| `--skip-files <glob>` | Exclude files |
| `--offline-scan` | No DB/network updates |
| `--scanners` | Select scan types |
| `--compliance <spec>` | Compliance report (e.g. `docker-cis`) |
| `--include-non-failures` | Show passed checks (misconfig) |
| `--misconfig-scanners <list>` | Limit IaC scanners |
| `--helm-*`, `--ansible-*`, `--tf-exclude-downloaded-modules` | IaC tuning |

### Database / cache
| Flag | Meaning |
|------|---------|
| `--db-repository` | OCI repo for trivy-db |
| `--java-db-repository` | OCI repo for Java DB |
| `--skip-db-update` / `--skip-java-db-update` | Don't update |
| `--download-db-only` / `--download-java-db-only` | Pre-fetch |
| `--cache-dir <dir>` | Cache location |
| `--cache-backend` | `fs` or Redis |
| `--no-progress` | Quiet progress |
| `--timeout <dur>` | Global timeout (default 5m) |
| `--insecure` | Allow insecure registry TLS |
| `--quiet` | Suppress logs |

### Image-specific
`--input <tar>`, `--image-src docker|containerd|podman|remote`,
`--platform <os/arch>`, `--docker-host`, `--podman-host`,
`--removed-pkgs`, `--image-config-scanners misconfig,secret`,
`--max-image-size`.

### Kubernetes-specific
Context is a positional argument, not a flag: `trivy k8s [flags] [CONTEXT]`.
Use `--kubeconfig`, `--include-namespaces`, `--exclude-namespaces`,
`--include-kinds`, `--exclude-kinds`, `--report summary|all`, `--scanners`.
There is no `--context`, `--namespace`, or `--components` flag in 0.76.

## Recipes

```bash
# Container image, all severities, table
trivy image nginx:latest

# Only HIGH/CRITICAL, fail CI on findings
trivy image --severity HIGH,CRITICAL --exit-code 1 myapp:latest

# JSON report
trivy image -f json -o image.json myapp:latest

# SARIF for GitHub code scanning
trivy image -f sarif -o trivy.sarif myapp:latest

# Local project: vulns + secrets + misconfig
trivy fs --scanners vuln,secret,misconfig .

# Secrets only, excluding tests
trivy fs --scanners secret --skip-dirs tests .

# IaC only (Terraform/K8s/Dockerfile/Helm)
trivy config .

# Remote repository
trivy repo https://github.com/org/repo

# SBOM generation + scan
trivy image -f cyclonedx -o sbom.json myapp:latest
trivy sbom sbom.json

# Kubernetes cluster summary
trivy k8s --report summary cluster

# Ignore unfixed vulnerabilities
trivy image --ignore-unfixed --severity HIGH,CRITICAL myapp:latest

# Air-gapped with pre-downloaded DB
trivy image --offline-scan --skip-db-update myapp:latest

# Client/server mode
trivy server --listen 0.0.0.0:4954
trivy image --server http://127.0.0.1:4954 myapp:latest
```

## Ignore file

`.trivyignore`:

```
# CVE with justification
CVE-2023-12345
# Expiry date support
CVE-2024-99999 exp:2026-12-31
```

Use `--ignorefile custom.ignore`.

## Output formats

| Format | Use |
|--------|-----|
| `table` | Human terminal |
| `json` | Automation / custom processing |
| `sarif` | GitHub/GitLab code scanning |
| `cyclonedx` | SBOM (CycloneDX) |
| `spdx` / `spdx-json` | SBOM (SPDX) |
| `github` | GitHub Actions annotations |
| `template` | Custom Go template |

## Gotchas

- First run downloads the vulnerability DB (hundreds of MB); use `--skip-db-update` after.
- `--exit-code` is required for CI failure; findings alone do not fail the build.
- Image scans need a container runtime or `--image-src remote` for registry-only.
- `--ignore-unfixed` hides real risk; document the choice.
- Secret scanning is regex/entropy-based — expect false positives in test fixtures.
- `trivy config` scans IaC, not live clusters; use `trivy k8s` for live.
- Scan results depend on DB freshness; pin versions in CI for reproducibility.
- Private registries need auth (`docker login` or `TRIVY_USERNAME`/`TRIVY_PASSWORD`).

## References

- Docs: https://trivy.dev/latest/docs/
- Configuration: https://trivy.dev/latest/docs/configuration/
- CI examples: https://trivy.dev/latest/docs/tutorials/integrations/
- Source: https://github.com/aquasecurity/trivy
