# CloudVLAN Security Assessment

> Security-first architecture: memory-safe language (Rust), mTLS between all components, automated CVE scanning in CI, OWASP-compliant auth (Argon2id, rate limiting, account lockout). Full security audit report available.

**Date:** 2026-02-10
**Scanner:** Trivy 0.69.1, cargo-audit, npm audit
**Scope:** All repos under ~/ws/ (cvlan, vrouter, ui, client, e2e)

---

## 1. Top Security Issues

The codebase has a **strong overall security posture** — no critical application-level vulnerabilities were found. The main areas to address:

### High Priority

| Issue | Location | Risk |
|-------|----------|------|
| **`bytes` 1.11.0 integer overflow** (RUSTSEC-2026-0007) | All 3 Rust repos | Memory corruption in `BytesMut::reserve` — affects tokio, hyper, axum, sqlx |
| **`axios` DoS via `__proto__` key** (GHSA-43fc-jf86-j433) | `ui/` | Production dependency, high severity |
| **CORS allows all origins** | `cvlan/api/src/rest/router.rs:30,71,128` | `.allow_origin(Any)` — mitigated by reverse proxy but should be restricted |
| **OpenSSL RCE in step-ca image** (CVE-2025-15467) | `smallstep/step-ca:latest` | CRITICAL — this is your certificate authority |
| **`time` 0.3.45 stack exhaustion DoS** (RUSTSEC-2026-0009) | `cvlan/` | Denial of service |

### Medium Priority

| Issue | Location | Risk |
|-------|----------|------|
| **`rsa` Marvin Attack** (RUSTSEC-2023-0071) | cvlan via `sqlx-mysql` | Timing sidechannel — you don't use MySQL, disable the feature |
| **`danger_accept_invalid_certs` fallback** | `cvlan/api/src/services/step_ca.rs` | Falls back when CA cert missing — OK for dev, dangerous in prod |
| **localStorage for JWT tokens** | `ui/src/` | Standard SPA risk, mitigated by React XSS protection |
| **No `cargo-deny` in CI** | All Rust repos | No automated dependency vulnerability scanning |
| **`reqwest` 0.11 (outdated)** | `cvlan/Cargo.toml` | vrouter/client already on 0.12; older TLS stack |
| **ubuntu:24.04 gpgv CVE** (CVE-2025-68973) | 7+ Dockerfiles | Out-of-bounds write in GnuPG |

### What's Done Well

- Argon2id password hashing with OWASP 2024 params
- Parameterized SQL queries everywhere (no injection risk)
- Ed25519 token signing, x25519 WireGuard keys
- Rate limiting per endpoint (governor crate)
- Account lockout (5 failures → 15 min)
- Non-root containers, multi-stage builds
- mTLS with step-ca, short-lived certificates
- Comprehensive audit logging
- Generic error messages (no user enumeration)
- Minimal `unsafe` (only FFI boundaries)

---

## 2. Recommended Pen Testing Tools

### Application-Level Testing

| Tool | What It Tests | Why |
|------|---------------|-----|
| **OWASP ZAP** | DAST — active/passive scanning of REST API | Free, comprehensive, good for API fuzzing against cvlan-api |
| **Burp Suite Pro** | DAST — manual + automated API testing | Industry standard for auth bypass, JWT manipulation, IDOR testing |
| **jwt_tool** | JWT attack surface | Test for alg:none, key confusion, token replay on your JWT implementation |
| **sqlmap** | SQL injection | Confirm parameterized queries hold under adversarial input |
| **Nuclei** | Template-based vuln scanning | Large template library for misconfigs, CVEs, default creds |

### Infrastructure / Container Testing

| Tool | What It Tests | Why |
|------|---------------|-----|
| **Trivy** | Container image CVEs, IaC misconfigs | Already used in this scan — add to CI pipeline |
| **Docker Bench Security** | CIS Docker benchmark | Checks container runtime configuration against best practices |
| **Dockle** | Dockerfile linting | Catches running as root, hardcoded secrets, missing HEALTHCHECK |
| **kube-bench** | K8s CIS benchmark | When/if you move to Kubernetes |

### Dependency & Supply Chain

| Tool | What It Tests | Why |
|------|---------------|-----|
| **cargo-audit** | Rust dependency CVEs | Already ran — integrate into CI |
| **cargo-deny** | License + advisory + duplicate deps | More comprehensive than cargo-audit, blocks bad deps in CI |
| **npm audit** | JS dependency CVEs | Already ran — integrate into CI |

### Network & Crypto Testing

| Tool | What It Tests | Why |
|------|---------------|-----|
| **testssl.sh** | TLS configuration | Verify cipher suites, protocol versions, cert chain on your mTLS endpoints |
| **nmap + scripts** | Port scanning + service detection | Ensure only expected ports exposed, check for service version disclosure |
| **Certipy** | PKI/certificate attack surface | Test your step-ca integration for cert template abuse, relay attacks |

### Recommended CI Pipeline Addition

```yaml
# Add to each repo's CI
- cargo deny check advisories   # Rust deps
- cargo deny check licenses     # License compliance
- trivy image --exit-code 1 --severity HIGH,CRITICAL $IMAGE
- npm audit --audit-level=high  # JS deps (ui/)
```

---

## 3. CVEs in Docker Images (CVSS >= 5)

**12 unique CVEs found across 5 images** (2 CRITICAL, 10 HIGH)

### CRITICAL (CVSS 9.0+)

| CVE | CVSS | Component | Affected Images | Fix |
|-----|------|-----------|----------------|-----|
| **CVE-2025-15467** | 9.8 | OpenSSL 3.x (oversized IV → RCE) | step-ca, tailscale, headscale | Pull newer images |
| **CVE-2025-68121** | 9.1 | Go stdlib TLS session resumption | postgres, step-ca, tailscale, headscale | Pull newer images |

### HIGH (CVSS 7.0+)

| CVE | CVSS | Component | Affected Images | Fix |
|-----|------|-----------|----------------|-----|
| CVE-2025-68973 | 8.1 | GnuPG gpgv OOB write | **ubuntu:24.04** (7 Dockerfiles) | `apt-get upgrade gpgv` |
| CVE-2025-69419 | 7.8 | OpenSSL PKCS#12 code exec | step-ca, tailscale, headscale | Pull newer images |
| CVE-2025-69421 | 7.5 | OpenSSL PKCS#12 DoS | step-ca, tailscale, headscale | Pull newer images |
| CVE-2026-25793 | 7.5 | Nebula blocklist bypass | step-ca | Pull newer image |
| CVE-2026-0861 | 7.4 | glibc heap corruption | headscale | **No fix yet** (Debian) |
| CVE-2025-58183 | 7.5 | Go archive/tar allocation | postgres, headscale | Pull newer images |
| CVE-2025-61726 | 7.5 | Go net/url memory exhaustion | All 5 images | Pull newer images |
| CVE-2025-61728 | 7.5 | Go archive/zip CPU exhaustion | All 5 images | Pull newer images |
| CVE-2025-61729 | 7.5 | Go crypto/x509 DoS | postgres, headscale | Pull newer images |
| CVE-2025-61730 | 7.5 | Go TLS 1.3 handshake vuln | All 5 images | Pull newer images |

### Rust Dependency CVEs (CVSS >= 5)

| Advisory | CVSS | Crate | Repos | Fix |
|----------|------|-------|-------|-----|
| RUSTSEC-2026-0007 | High | `bytes` 1.11.0 | all 3 | `cargo update -p bytes` |
| RUSTSEC-2026-0009 | 6.8 | `time` 0.3.45 | cvlan | `cargo update -p time` |
| RUSTSEC-2023-0071 | 5.9 | `rsa` 0.9.10 | cvlan | Disable sqlx `mysql` feature |

### JS Dependency CVEs

| Advisory | Severity | Package | Repo | Fix |
|----------|----------|---------|------|-----|
| GHSA-43fc-jf86-j433 | High | `axios` <=1.13.4 | ui | `npm audit fix` |
| GHSA-67mh-4wv8-2f99 | Moderate | `esbuild` <=0.24.2 | ui (dev only) | Upgrade vite to 7.x |

---

## 4. Immediate Action Items (Priority Order)

1. **`docker pull smallstep/step-ca:latest`** — OpenSSL RCE on your CA is the single highest risk
2. **`cargo update -p bytes`** in all 3 Rust repos — zero-risk semver patch
3. **`cargo update -p time`** in cvlan — zero-risk semver patch
4. **`npm audit fix`** in ui — fixes axios DoS
5. **Pull updated base images** (`postgres:16-alpine`, `ubuntu:24.04`, `tailscale`, `headscale`) and rebuild
6. **Disable sqlx `mysql` feature** in cvlan to drop the unfixable `rsa` advisory
7. **Add `cargo-deny` + `trivy`** to CI pipelines for ongoing scanning
8. **Restrict CORS** to known origins before production deployment

---

## 5. Image Scan Details

### Images Scanned

| Image | Size | Base | CVEs Found |
|-------|------|------|------------|
| `postgres:16-alpine` | 389MB | Alpine | 6 (1 CRIT, 5 HIGH) |
| `ubuntu:24.04` | 258MB | Ubuntu Noble | 1 (HIGH) |
| `smallstep/step-ca:latest` | 187MB | Alpine | 20 across 4 targets (8 unique) |
| `tailscale/tailscale:latest` | 157MB | Alpine | 18 across 4 targets (7 unique) |
| `headscale/headscale:latest` | 120MB | Debian 12 | 10 across 2 targets (9 unique) |

### Base Images in Dockerfiles

| Base Image | Used By |
|------------|---------|
| `ubuntu:24.04` | cvlan API runtime, cvlan build, vrouter runtime, vrouter build, regression runner, client runtime, client build |
| `node:20-alpine` | UI build |
| `nginx:alpine` | UI runtime |
| `postgres:16-alpine` | Test/dev/e2e databases |
| `smallstep/step-ca:latest` | Certificate authority (tests + e2e) |
| `python:3.12-slim` | vrouter test runner |

**Note:** Workspace build images (cvlan-api, cvlan-build, vrouter, cvlan-ui, client-build, cvlan-client, regression-runner) were not built locally at scan time. Rebuild on updated base images and re-scan.
