# trivy — command scenarios for deep research

Every trivy command form, grouped by scenario. Subcommands from
[`help.txt`](help.txt); per-target flags from
[`help-image.txt`](help-image.txt), [`help-filesystem.txt`](help-filesystem.txt),
[`help-repository.txt`](help-repository.txt), [`help-rootfs.txt`](help-rootfs.txt),
[`help-sbom.txt`](help-sbom.txt), [`help-kubernetes.txt`](help-kubernetes.txt),
[`help-config.txt`](help-config.txt), [`help-vm.txt`](help-vm.txt).

## How to invoke

```text
@deep-research For Trivy 0.76, which findings from a `trivy image` scan are
actually reachable in a distroless Go service, and how do I build a VEX
statement that suppresses the rest without hiding real risk? Cite Trivy docs,
OpenVEX/CSAF specs, and Aqua advisories; return exact commands and a sample
.trivyignore.yaml.
```

---

## Scenario 1 — container image scanning

| # | Command | How to use |
|---|---------|------------|
| 1 | `trivy image nginx:latest` | Registry image, default scanners |
| 2 | `trivy image --input app.tar` | Local `docker save` archive |
| 3 | `trivy image --image-src remote nginx:latest` | Registry only, no local runtime |
| 4 | `trivy image --image-src docker,remote nginx:latest` | Priority-ordered sources |
| 5 | `trivy image --docker-host unix:///var/run/docker.sock nginx:latest` | Explicit Docker socket |
| 6 | `trivy image --podman-host unix:///run/podman/podman.sock nginx:latest` | Explicit Podman socket |
| 7 | `trivy image --platform linux/arm64 nginx:latest` | Multi-arch image, pick platform |
| 8 | `trivy image --removed-pkgs alpine:3.20` | Alpine: include removed packages |
| 9 | `trivy image --max-image-size 2GB app:latest` | Refuse oversized images |
| 10 | `trivy image --image-config-scanners misconfig,secret app:latest` | Inspect image config too |
| 11 | `trivy image --scanners vuln,secret,misconfig app:latest` | All content scanners |
| 12 | `trivy image --scanners license --license-full app:latest` | License inventory |
| 13 | `trivy image --pkg-types os app:latest` | OS packages only |
| 14 | `trivy image --pkg-types library --pkg-relationships direct app:latest` | Direct app deps only |
| 15 | `trivy image --distro debian/12 app:latest` | Override distro detection |
| 16 | `trivy image --offline-scan app:latest` | No dependency API calls |
| 17 | `trivy image --sbom-sources oci app:latest` | Reuse SBOM attestations |
| 18 | `trivy image --sbom-sources oci,rekor app:latest` | OCI + Rekor SBOM lookup |

**Research questions**
- Which `--image-src` path is used when the image exists in multiple runtimes?
- Does `--offline-scan` change vulnerability matching quality for language deps?
- For distroless/static images, which packages remain scan targets?

**Sources**: Trivy image docs, Aqua blog posts on distroless scanning, GitHub
issues about `--image-src` precedence and `--offline-scan`.

**Output**: one reproducible image-scan command per runtime, with the chosen
source and scanner set stated in the report.

---

## Scenario 2 — filesystem, repository, rootfs, SBOM, VM

| # | Command | How to use |
|---|---------|------------|
| 19 | `trivy fs .` | Local project directory |
| 20 | `trivy fs --scanners vuln,secret,misconfig .` | All filesystem scanners |
| 21 | `trivy fs --include-dev-deps .` | Include npm/yarn/gradle dev deps |
| 22 | `trivy fs --pkg-types library --pkg-relationships workspace .` | Monorepo workspace deps |
| 23 | `trivy fs --skip-dirs node_modules --skip-files '*.min.js' .` | Trim noise |
| 24 | `trivy fs --file-patterns 'dockerfile:.*Dockerfile.*' .` | Custom config file patterns |
| 25 | `trivy fs --distro debian/12 /mnt/disk` | Scan a mounted disk as a distro |
| 26 | `trivy repo https://github.com/org/repo` | Remote Git repository |
| 27 | `trivy repo /path/to/local/repo` | Local Git repository |
| 28 | `trivy repo --branch main https://github.com/org/repo` | Specific branch |
| 29 | `trivy repo --commit <sha> https://github.com/org/repo` | Specific commit |
| 30 | `trivy repo --tag v1.2.3 https://github.com/org/repo` | Specific tag |
| 31 | `trivy repo --include-dev-deps https://github.com/org/repo` | Include dev deps |
| 32 | `trivy rootfs /mnt/rootfs` | Unpacked root filesystem |
| 33 | `trivy rootfs --distro alpine/3.20 /mnt/rootfs` | Unknown rootfs, force distro |
| 34 | `trivy rootfs --exit-on-eol 3 /mnt/rootfs` | Fail when OS is EOL |
| 35 | `trivy sbom bom.cdx.json` | CycloneDX SBOM |
| 36 | `trivy sbom bom.spdx.json` | SPDX SBOM |
| 37 | `trivy sbom report.cdx.intoto.jsonl` | CycloneDX attestation |
| 38 | `trivy sbom --scanners license bom.cdx.json` | License-only from SBOM |
| 39 | `trivy vm ami:ami-0123456789abcdef0` | AWS AMI (experimental) |
| 40 | `trivy vm ebs:snap-0123456789abcdef0` | AWS EBS snapshot |
| 41 | `trivy vm --aws-region us-east-1 ami:ami-0123456789abcdef0` | Explicit AWS region |

**Research questions**
- `branch`, `commit`, and `tag` are mutually exclusive in practice — which wins
  if combined, and does Trivy fail or silently pick one?
- For `trivy fs` on a monorepo, are workspace relationships honored for all
  ecosystems or only npm/yarn/gradle?
- Is `trivy vm` safe to run against production AMIs (read-only snapshot vs
  live instance)?

**Sources**: repository docs, fs docs, VM docs, issues on monorepo scanning
and AWS permissions.

**Output**: the correct target subcommand for each artifact type, with the
history selector recorded for repo scans.

---

## Scenario 3 — IaC / misconfiguration scanning

| # | Command | How to use |
|---|---------|------------|
| 42 | `trivy config .` | Auto-detect Dockerfile/K8s/Terraform/Helm/etc. |
| 43 | `trivy config --misconfig-scanners dockerfile .` | Dockerfile checks only |
| 44 | `trivy config --misconfig-scanners kubernetes,helm .` | K8s + Helm only |
| 45 | `trivy config --misconfig-scanners terraform .` | Terraform only |
| 46 | `trivy config --misconfig-scanners cloudformation .` | CloudFormation only |
| 47 | `trivy config --misconfig-scanners ansible .` | Ansible only |
| 48 | `trivy config --checks-bundle-repository ghcr.io/aquasecurity/trivy-checks:1 .` | Pin checks bundle |
| 49 | `trivy config --skip-check-update .` | Offline / pinned checks |
| 50 | `trivy config --include-deprecated-checks .` | Include deprecated checks |
| 51 | `trivy config --check-namespaces builtin .` | Limit Rego namespaces |
| 52 | `trivy config --config-check ./checks .` | Custom Rego checks |
| 53 | `trivy config --config-data ./data .` | Rego data files |
| 54 | `trivy config --rego-error-limit 25 .` | Allow more Rego compile errors |
| 55 | `trivy config --trace-rego .` | Verbose Rego query tracing |
| 56 | `trivy config --include-non-failures .` | Show passed checks |
| 57 | `trivy config --tf-vars prod.tfvars .` | Terraform variable values |
| 58 | `trivy config --tf-exclude-downloaded-modules .` | Skip downloaded modules |
| 59 | `trivy config --config-file-schemas schema.json .` | JSON schema hint for Rego |
| 60 | `trivy config --cf-params params.json .` | CloudFormation parameters |
| 61 | `trivy config --helm-values values-prod.yaml .` | Helm values file |
| 62 | `trivy config --helm-set image.tag=1.2.3 .` | Helm value override |
| 63 | `trivy config --helm-set-string env=prod .` | Helm string value |
| 64 | `trivy config --helm-set-file ca=certs/ca.pem .` | Helm value from file |
| 65 | `trivy config --helm-api-versions apps/v1 .` | Helm capability API versions |
| 66 | `trivy config --helm-kube-version 1.30.0 .` | Helm Kubernetes version |
| 67 | `trivy config --ansible-playbook site.yml .` | Explicit playbook |
| 68 | `trivy config --ansible-inventory hosts.ini .` | Ansible inventory |
| 69 | `trivy config --ansible-extra-vars env=prod .` | Ansible extra vars |
| 70 | `trivy config --raw-config-scanners terraform .` | Also scan raw configs |
| 71 | `trivy config --render-cause terraform .` | Show rendered cause |
| 72 | `trivy config --detection-priority comprehensive .` | More findings, more FPs |
| 73 | `trivy config --compliance docker-cis-1.6.0 .` | Compliance report |

**Research questions**
- Which misconfig checks are deprecated in the current checks bundle, and what
  replaced them?
- Does `--helm-set` override `--helm-values`, and are subchart values merged?
- What false-positive rate should be expected from `comprehensive` mode?

**Sources**: misconfiguration docs, trivy-checks repo, Helm docs on values
precedence, issues on Rego namespaces.

**Output**: a pinned checks bundle plus the exact template/values rendering
inputs, so findings are reproducible.

---

## Scenario 4 — Kubernetes cluster scanning

| # | Command | How to use |
|---|---------|------------|
| 74 | `trivy k8s --report summary` | Current context, summary |
| 75 | `trivy k8s --report all` | Per-resource findings |
| 76 | `trivy k8s kind-kind --report summary` | Named kubeconfig context |
| 77 | `trivy k8s --kubeconfig ~/.kube/config --report summary` | Explicit kubeconfig |
| 78 | `trivy k8s --include-namespaces app,prod` | Only these namespaces |
| 79 | `trivy k8s --exclude-namespaces kube-system` | Skip system namespace |
| 80 | `trivy k8s --include-kinds deployment,statefulset` | Only these kinds |
| 81 | `trivy k8s --exclude-kinds node` | Skip node scans |
| 82 | `trivy k8s --exclude-owned` | Skip owned child resources |
| 83 | `trivy k8s --skip-images` | Config only, no image pulls |
| 84 | `trivy k8s --disable-node-collector` | No node-collector job |
| 85 | `trivy k8s --node-collector-imageref ghcr.io/aquasecurity/node-collector:0.3.1` | Pin collector image |
| 86 | `trivy k8s --node-collector-namespace security-scan` | Collector namespace |
| 87 | `trivy k8s --exclude-nodes kubernetes.io/arch:arm64` | Skip labeled nodes |
| 88 | `trivy k8s --tolerations key1=value1:NoExecute` | Collector tolerations |
| 89 | `trivy k8s --k8s-version 1.30.0` | Outdated API validation |
| 90 | `trivy k8s --qps 20 --burst 40` | API client throttling |
| 91 | `trivy k8s --scanners vuln,misconfig,secret` | Scanner selection |
| 92 | `trivy k8s --compliance k8s-cis-1.23` | CIS benchmark report |

**Research questions**
- What RBAC does the node-collector job need, and what does it leave behind?
- Does `--exclude-owned` change coverage for controller-managed pods?
- Which built-in compliance specs ship with 0.76?

**Sources**: Kubernetes docs, trivy k8s docs, node-collector repo, issues on
RBAC and cleanup.

**Output**: a least-privilege cluster scan command with namespace/kind scope
and a cleanup guarantee.

---

## Scenario 5 — severities, filtering, ignore and exit codes

| # | Command | How to use |
|---|---------|------------|
| 93 | `trivy image --severity HIGH,CRITICAL app:latest` | Only high-risk findings |
| 94 | `trivy image --severity UNKNOWN,LOW app:latest` | Triage the long tail |
| 95 | `trivy image --ignore-unfixed app:latest` | Only fixable vulns |
| 96 | `trivy image --ignore-status will_not_fix,fix_deferred app:latest` | Status filters |
| 97 | `trivy image --ignore-status end_of_life app:latest` | Hide EOL packages |
| 98 | `trivy image --vuln-severity-source debian app:latest` | Prefer Debian severities |
| 99 | `trivy image --vuln-severity-source nvd,ghsa,osv app:latest` | Source priority |
| 100 | `trivy image --ignorefile .trivyignore app:latest` | Custom ignore file |
| 101 | `trivy image --ignorefile '' app:latest` | Disable ignore file |
| 102 | `trivy image --ignore-policy policy.rego app:latest` | Rego-based ignore logic |
| 103 | `trivy image --show-suppressed app:latest` | Audit what was suppressed |
| 104 | `trivy image --vex ./openvex.json app:latest` | Apply VEX statements |
| 105 | `trivy image --vex repo app:latest` | VEX from configured repos |
| 106 | `trivy image --vex oci app:latest` | VEX from OCI attestations |
| 107 | `trivy image --skip-vex-repo-update --vex repo app:latest` | Offline VEX |
| 108 | `trivy image --exit-code 1 app:latest` | Fail on any finding |
| 109 | `trivy image --exit-code 1 --severity HIGH,CRITICAL app:latest` | Fail on high-risk |
| 110 | `trivy image --exit-on-eol 2 app:latest` | Distinct EOL exit code |

**Research questions**
- Does `--severity` filter before or after VEX suppression?
- How do `.trivyignore` expiry dates and `--ignore-policy` interact?
- Which severity source best matches the distro's own security tracker?

**Sources**: filtering docs, VEX docs, `.trivyignore` docs, distro security
trackers, issues on severity-source behavior.

**Output**: one documented risk policy (status + severity + VEX) that a CI
gate can enforce.

---

## Scenario 6 — reporting, formats and conversion

| # | Command | How to use |
|---|---------|------------|
| 111 | `trivy image -f table app:latest` | Human summary (default) |
| 112 | `trivy image -f json -o out.json app:latest` | Machine-readable JSON |
| 113 | `trivy image -f sarif -o out.sarif app:latest` | GitHub/GitLab code scanning |
| 114 | `trivy image -f cyclonedx -o out.cdx app:latest` | CycloneDX SBOM |
| 115 | `trivy image -f spdx -o out.spdx app:latest` | SPDX tag-value |
| 116 | `trivy image -f spdx-json -o out.spdx.json app:latest` | SPDX JSON |
| 117 | `trivy image -f github app:latest` | GitHub Actions annotations |
| 118 | `trivy image -f cosign-vuln -o out.vuln app:latest` | cosign attestation format |
| 119 | `trivy image -f template -t report.tpl -o out.txt app:latest` | Custom Go template |
| 120 | `trivy image -o out.txt app:latest` | Redirect default table |
| 121 | `trivy image --list-all-pkgs -f json app:latest` | Every package in JSON |
| 122 | `trivy image --list-all-pkgs=false -f json app:latest` | Vulnerable packages only |
| 123 | `trivy image --dependency-tree app:latest` | Why a package is installed |
| 124 | `trivy image --table-mode summary app:latest` | Summary table only |
| 125 | `trivy image --table-mode detailed app:latest` | Detailed table only |
| 126 | `trivy image --table-mode summary,detailed app:latest` | Both tables |
| 127 | `trivy image --report summary --compliance docker-cis-1.6.0 app:latest` | Compliance summary |
| 128 | `trivy image --report all --compliance docker-cis-1.6.0 app:latest` | Full compliance report |
| 129 | `trivy convert -f cyclonedx -o out.cdx result.json` | Convert saved JSON |
| 130 | `trivy convert -f table result.json` | Re-render JSON as table |
| 131 | `trivy convert --scanners vuln,secret result.json` | Label scanners for summary |
| 132 | `trivy convert --dependency-tree result.json` | Dependency tree while converting |

**Research questions**
- Which formats are lossy or omit fields needed for a VEX workflow?
- Is `cosign-vuln` stable enough for `cosign attest` in CI?
- How do custom templates access nested package/vulnerability fields?

**Sources**: report formats docs, SARIF/CycloneDX/SPDX specs, cosign docs,
community templates.

**Output**: one authoritative JSON artifact kept for audit plus derived
formats generated by `trivy convert`.

---

## Scenario 7 — secrets and licenses

| # | Command | How to use |
|---|---------|------------|
| 133 | `trivy fs --scanners secret .` | Secret scan |
| 134 | `trivy fs --scanners secret --secret-config my-secret.yaml .` | Custom secret rules |
| 135 | `trivy fs --scanners secret --secret-config '' .` | Disable custom rules |
| 136 | `trivy image --scanners secret app:latest` | Secrets baked into image |
| 137 | `trivy fs --scanners license .` | License inventory |
| 138 | `trivy fs --scanners license --license-full .` | Scan headers and license files |
| 139 | `trivy fs --scanners license --ignored-licenses GPL-3.0,AGPL-3.0 .` | Flag copyleft |
| 140 | `trivy fs --scanners license --license-confidence-level 0.8 .` | Looser classifier |
| 141 | `trivy image --scanners license --license-full app:latest` | Image license audit |

**Research questions**
- What built-in secret rules exist, and what is the false-positive profile on
  test fixtures and lockfiles?
- How does the license classifier score confidence, and which licenses are
  normalized?
- Are secret findings ever sent off-host for validation?

**Sources**: secret scanning docs, license docs, trivy source, issues about
FP rates.

**Output**: a secret-scan rule set and license policy with documented
exceptions, not ad-hoc ignores.

---

## Scenario 8 — database, cache, air-gap, registry auth, client/server

| # | Command | How to use |
|---|---------|------------|
| 142 | `trivy image --skip-db-update app:latest` | Use cached DB |
| 143 | `trivy image --skip-java-db-update app:latest` | Use cached Java DB |
| 144 | `trivy image --download-db-only` | Pre-fetch DB, no scan |
| 145 | `trivy image --download-java-db-only` | Pre-fetch Java DB |
| 146 | `trivy image --db-repository ghcr.io/aquasecurity/trivy-db:2 app:latest` | Pin DB repo |
| 147 | `trivy image --java-db-repository ghcr.io/aquasecurity/trivy-java-db:1 app:latest` | Pin Java DB repo |
| 148 | `trivy image --cache-dir /var/cache/trivy app:latest` | Shared cache dir |
| 149 | `trivy image --cache-backend redis://127.0.0.1:6379 app:latest` | Redis cache |
| 150 | `trivy image --cache-backend redis://cache:6379 --cache-ttl 24h app:latest` | Redis with TTL |
| 151 | `trivy image --cache-backend redis://cache:6379 --redis-tls --redis-ca ca.pem --redis-cert cert.pem --redis-key key.pem app:latest` | TLS Redis |
| 152 | `trivy image --offline-scan --skip-db-update app:latest` | Fully air-gapped |
| 153 | `trivy image --no-progress app:latest` | Clean logs |
| 154 | `trivy registry login reg.example.com --username ci --password-stdin` | Registry auth |
| 155 | `trivy registry logout reg.example.com` | Remove credentials |
| 156 | `trivy image --username ci --password "$TOKEN" reg.example.com/app:latest` | Inline credentials (avoid) |
| 157 | `trivy image --password-stdin reg.example.com/app:latest` | Password from stdin |
| 158 | `trivy image --registry-token "$TOKEN" reg.example.com/app:latest` | Registry token |
| 159 | `trivy server --listen 127.0.0.1:4954` | Server mode |
| 160 | `trivy image --server http://127.0.0.1:4954 app:latest` | Client mode |
| 161 | `trivy image --server https://trivy.internal:4954 --token "$T" app:latest` | Token auth |
| 162 | `trivy image --server https://trivy.internal:4954 --token "$T" --token-header X-Trivy-Token app:latest` | Custom token header |
| 163 | `trivy image --server https://trivy.internal:4954 --custom-headers 'X-Env: prod' app:latest` | Extra client headers |
| 164 | `trivy image --server https://trivy.internal:4954 --cacert ca.pem app:latest` | Private CA |

**Research questions**
- Which database artifacts must be mirrored for a fully air-gapped install
  (vuln DB, Java DB, checks bundle, VEX repos)?
- Is Redis cache safe to share across scanners with different DB versions?
- What exactly does server mode centralize, and what still runs client-side?

**Sources**: air-gap docs, client/server docs, cache docs, registry auth docs.

**Output**: a mirrored artifact list and a client/server topology with auth and
TLS specified.

---

## Scenario 9 — management, utility and extensions

| # | Command | How to use |
|---|---------|------------|
| 165 | `trivy module install <repository>` | Install a WASM module |
| 166 | `trivy module uninstall <repository>` | Remove a module |
| 167 | `trivy module --module-dir ./modules install <repository>` | Custom module dir |
| 168 | `trivy image --enable-modules custom-module app:latest` | Enable modules for a scan |
| 169 | `trivy plugin search sbom` | Search the plugin index |
| 170 | `trivy plugin list` | Installed plugins |
| 171 | `trivy plugin install <plugin>` | Install a plugin |
| 172 | `trivy plugin info <plugin>` | Plugin metadata |
| 173 | `trivy plugin run <plugin> -- <args>` | Run a plugin |
| 174 | `trivy plugin uninstall <plugin>` | Remove a plugin |
| 175 | `trivy plugin update` | Refresh plugin index |
| 176 | `trivy plugin upgrade` | Upgrade all plugins |
| 177 | `trivy vex repo init` | Create VEX repo config |
| 178 | `trivy vex repo list` | List VEX repositories |
| 179 | `trivy vex repo download` | Fetch VEX repos |
| 180 | `trivy clean --all` | Remove every cache |
| 181 | `trivy clean --vuln-db` | Remove vulnerability DB |
| 182 | `trivy clean --java-db` | Remove Java DB |
| 183 | `trivy clean --checks-bundle` | Remove checks bundle |
| 184 | `trivy clean --scan-cache` | Remove image/VM analysis cache |
| 185 | `trivy clean --vex-repo` | Remove VEX repositories |
| 186 | `trivy completion bash` | Bash completion |
| 187 | `trivy completion zsh` | Zsh completion |
| 188 | `trivy completion fish` | Fish completion |
| 189 | `trivy completion powershell` | PowerShell completion |
| 190 | `trivy version` | Version text |
| 191 | `trivy version -f json` | Version as JSON |
| 192 | `trivy --generate-default-config` | Write `trivy-default.yaml` |

**Research questions**
- Which plugin/module interfaces are stable in 0.76, and which are
  experimental?
- Does `trivy clean --all` ever remove evidence needed for an audit?
- What does `--generate-default-config` emit for fields not exposed as flags?

**Sources**: plugin docs, module docs, CLI source, release notes.

**Output**: a versioned extension and cleanup policy; no destructive cleanup
during an active audit.

---

## Scenario 10 — performance, network and global flags

| # | Command | How to use |
|---|---------|------------|
| 193 | `trivy image --parallel 8 app:latest` | More scan goroutines |
| 194 | `trivy image --parallel 0 app:latest` | Auto-detect parallelism |
| 195 | `trivy image --timeout 20m app:latest` | Raise global timeout |
| 196 | `trivy image --insecure reg.local/app:latest` | Skip TLS verification (lab only) |
| 197 | `trivy image --cacert ca.pem reg.local/app:latest` | Trust private CA |
| 198 | `trivy image --disable-telemetry app:latest` | No anonymous usage data |
| 199 | `trivy image --skip-version-check app:latest` | No update notices |
| 200 | `trivy image -q app:latest` | Quiet output |
| 201 | `trivy image -d app:latest` | Debug logging |
| 202 | `trivy image --rekor-url https://rekor.internal app:latest` | Private Rekor |
| 203 | `trivy image -c trivy.yaml app:latest` | Explicit config file |
| 204 | `trivy image -c '' app:latest` | Disable config file |
| 205 | `trivy --version` | Global version |
| 206 | `trivy --help` | Command list |

**Research questions**
- Which parallelism knob dominates: `--parallel`, `--qps`, or image pull
  concurrency?
- Does `--disable-telemetry` also stop version checks and DB calls?
- What settings are only available through `trivy.yaml`?

**Sources**: performance docs, telemetry docs, config file reference, source.

**Output**: a config file plus explicit flags for CI, with network egress
documented.

---

## Complete flag coverage

Every flag from the snapshots above appears at least once. Flags marked
"per-target" exist only on some subcommands; check the relevant help file.

| Group | Flags | Scenario |
|-------|-------|----------|
| Global | `--cacert`, `--cache-dir`, `-c/--config`, `-d/--debug`, `--generate-default-config`, `--insecure`, `-q/--quiet`, `--timeout`, `-v/--version`, `-h/--help` | 10 |
| Target selection | `image`, `filesystem`/`fs`, `repository`/`repo`, `rootfs`, `sbom`, `config`/`conf`, `kubernetes`/`k8s`, `vm` | 1–4 |
| Image | `--docker-host`, `--image-config-scanners`, `--image-src`, `--input`, `--max-image-size`, `--platform`, `--podman-host`, `--removed-pkgs` | 1 |
| Repository | `--branch`, `--commit`, `--tag` | 2 |
| VM | `--aws-region` | 2 |
| Package scope | `--include-dev-deps`, `--pkg-types`, `--pkg-relationships`, `--removed-pkgs` | 1, 2 |
| Scanners | `--scanners`, `--detection-priority`, `--distro`, `--file-patterns`, `--offline-scan`, `--parallel`, `--sbom-sources`, `--rekor-url`, `--skip-dirs`, `--skip-files`, `--skip-version-check`, `--disable-telemetry` | 1–4, 10 |
| Severity / status | `--severity`/`-s`, `--ignore-unfixed`, `--ignore-status`, `--vuln-severity-source` | 5 |
| Ignore / VEX | `--ignorefile`, `--ignore-policy`, `--show-suppressed`, `--vex`, `--skip-vex-repo-update` | 5 |
| Exit codes | `--exit-code`, `--exit-on-eol` | 5 |
| Report | `-f/--format`, `-o/--output`, `--template`/`-t`, `--list-all-pkgs`, `--dependency-tree`, `--table-mode`, `--compliance`, `--report`, `--output-plugin-arg` | 6 |
| Secrets | `--secret-config` | 7 |
| License | `--license-full`, `--ignored-licenses`, `--license-confidence-level` | 7 |
| Misconfig / IaC | `--ansible-extra-vars`, `--ansible-inventory`, `--ansible-playbook`, `--checks-bundle-repository`, `--config-file-schemas`, `--helm-api-versions`, `--helm-kube-version`, `--helm-set`, `--helm-set-file`, `--helm-set-string`, `--helm-values`, `--include-non-failures`, `--misconfig-scanners`, `--raw-config-scanners`, `--render-cause`, `--tf-exclude-downloaded-modules`, `--tf-vars`, `--cf-params`, `--check-namespaces`, `--config-check`, `--config-data`, `--include-deprecated-checks`, `--rego-error-limit`, `--skip-check-update`, `--trace-rego` | 3 |
| Kubernetes | `--burst`, `--disable-node-collector`, `--exclude-kinds`, `--exclude-namespaces`, `--exclude-nodes`, `--exclude-owned`, `--include-kinds`, `--include-namespaces`, `--k8s-version`, `--kubeconfig`, `--node-collector-imageref`, `--node-collector-namespace`, `--qps`, `--skip-images`, `--tolerations` | 4 |
| Database | `--db-repository`, `--java-db-repository`, `--download-db-only`, `--download-java-db-only`, `--skip-db-update`, `--skip-java-db-update`, `--no-progress` | 8 |
| Cache | `--cache-backend`, `--cache-ttl`, `--redis-ca`, `--redis-cert`, `--redis-key`, `--redis-tls` | 8 |
| Registry | `--username`, `--password`, `--password-stdin`, `--registry-token` | 8 |
| Client/server | `--server`, `--listen` (server), `--token`, `--token-header`, `--custom-headers` | 8 |
| Modules | `--enable-modules`, `--module-dir` | 9 |
| Clean | `-a/--all`, `--checks-bundle`, `--java-db`, `--scan-cache`, `--vuln-db`, `--vex-repo` | 9 |

Subcommands: `image` (`i`), `filesystem` (`fs`), `repository` (`repo`),
`rootfs`, `sbom`, `config` (`conf`), `kubernetes` (`k8s`), `vm`, `server`
(`s`), `module` (`m`), `plugin` (`p`), `vex repo`, `clean`, `convert`
(`conv`), `completion`, `registry`, `version`, `help`.

`help-secret.txt` and `help-license.txt` are root help snapshots: there is no
standalone `trivy secret` or `trivy license` subcommand.

Full per-flag text: the `help-*.txt` files linked at the top.
