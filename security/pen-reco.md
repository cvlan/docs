# Pen Testing Tool Recommendations

**Stack:** Rust/axum REST API, React SPA, Docker, mTLS/step-ca, WireGuard

---

## Tier 1 — Run Before Production

**OWASP ZAP** — automated DAST scanner
- Point at cvlan-api on your e2e stack (port 9080)
- Finds auth bypass, header injection, CORS misconfig, SSRF
- Free, scriptable, has API scan mode for OpenAPI specs
- `docker run -t ghcr.io/zaproxy/zaproxy:stable zap-api-scan.py -t http://host:9080/api/v1 -f openapi`

**Trivy** — already used in this audit
- Image CVE scanning, Dockerfile misconfiguration, IaC scanning
- Add to CI: `trivy image --exit-code 1 --severity HIGH,CRITICAL`
- Also scans `Cargo.lock` and `package-lock.json` directly

**cargo-deny** — Rust supply chain
- Advisories, license compliance, duplicate crate detection
- Drop-in CI step, blocks PRs with known vulns

**testssl.sh** — TLS configuration audit
- Critical for your mTLS setup with step-ca
- Checks cipher suites, protocol versions, cert chain, HSTS
- `docker run --rm -ti drwetter/testssl.sh https://your-api:443`

## Tier 2 — Pre-Launch Hardening

**Burp Suite Pro** (~$450/yr) — manual API pen testing
- Best for testing JWT handling (alg confusion, token replay, privilege escalation)
- IDOR testing on your multi-tenant endpoints (`/tenants/{id}/cvlans`, `/nodes`)
- Auth bypass attempts on role boundaries (readonly → tenantadmin escalation)
- The free Community Edition works but lacks the active scanner

**jwt_tool** — JWT-specific attack surface (free)
- Tests alg:none, HMAC/RSA confusion, key brute-force, claim tampering
- Directly relevant since you have 3 JWT secret environments
- `python3 jwt_tool.py <token> -M at -t http://host:9080/api/v1/me`

**Nuclei** — template-based vuln scanning (free)
- 8000+ community templates for misconfigs, default creds, known CVEs
- Good for scanning your nginx (ui), step-ca, postgres exposed ports
- `nuclei -u http://host:9080 -tags cve,misconfig,exposure`

**Docker Bench Security** — CIS benchmark (free)
- Audits your Docker daemon config, container runtime settings
- Checks for privileged containers, host mounts, network mode
- `docker run --rm -v /var/run/docker.sock:/var/run/docker.sock docker/docker-bench-security`

## Tier 3 — Ongoing / Deeper Testing

**sqlmap** — SQL injection confirmation (free)
- Your parameterized queries should hold, but worth confirming
- Run against all API endpoints that accept user input
- Peace-of-mind validation, not expected to find anything

**nmap** — network surface audit (free)
- Verify only expected ports are exposed per container
- Service version detection (check for version disclosure)
- `nmap -sV -p- your-host`

**Certipy** — PKI attack surface (free)
- Relevant for your step-ca certificate authority
- Tests for cert template abuse, relay attacks, ESC1-ESC8 vectors
- Important since compromised CA = compromised entire mesh

**Dockle** — Dockerfile linter (free)
- Catches things Trivy misses: running as root, secrets in layers, missing HEALTHCHECK
- `dockle cvlan-api:latest`

## What to Skip

| Tool | Why Skip |
|------|----------|
| Metasploit | Overkill — you're not testing OS-level exploits |
| Nessus/Qualys | Enterprise scanner, expensive, more suited for VM fleets |
| Cobalt Strike | Red team C2 — not relevant for app-level testing |
| Nikto | Outdated, ZAP covers everything it does and more |

## Suggested Order of Operations

1. **cargo-deny + trivy + npm audit** in CI (automated, catches deps continuously)
2. **testssl.sh** against mTLS endpoints (one-time, validates TLS config)
3. **ZAP API scan** against e2e stack (automated, finds surface-level issues)
4. **jwt_tool** against all 3 JWT environments (targeted, tests your auth)
5. **Burp Suite** manual testing of multi-tenant boundaries (manual, finds logic bugs)
6. **Docker Bench + Dockle** before deployment (one-time, validates container posture)
7. **Nuclei** broad scan of all exposed services (automated, catches known CVEs)

The first 4 are free and can run in a CI pipeline. Burp Pro is the only paid tool worth the investment for this use case.

---

## Risk Assessment: Odds of a 10/10 Vulnerability in the API Layer

### Why a CVSS 10.0 is Unlikely

A CVSS 10.0 requires: network-reachable, no auth needed, no user interaction, scope change, and complete confidentiality + integrity + availability impact. That's essentially unauthenticated RCE.

| Attack Vector | Why It's Hard | Residual Risk |
|---------------|---------------|---------------|
| Buffer overflow → RCE | Rust memory safety, only 2 `unsafe` blocks (both in vrouter, not API) | ~0% from your code |
| SQL injection → data exfil | sqlx parameterized + compile-time checked queries everywhere | ~0% |
| Auth bypass → full access | Proper JWT validation, no alg:none (jsonwebtoken crate rejects it), generic errors | Low |
| Command injection → RCE | No `Command::new()` with user input, no shell exec from API handlers | ~0% |
| Deserialization → RCE | serde is not vulnerable to the Java/Python-style deser RCE class | ~0% |
| SSRF → internal access | step-ca/discovery URLs are config-driven, not user-controlled | Low |
| Path traversal | No file serving from user-controlled paths | ~0% |

### Where a 10.0 *Could* Hide

**1. Dependency — most likely vector (~5-10% odds)**
- The `bytes` integer overflow (RUSTSEC-2026-0007) is the closest thing found. A crafted request hitting `BytesMut::reserve` in the right way could potentially cause memory corruption in tokio/hyper/axum before your code even runs
- A future 0-day in `hyper`, `rustls`, `axum`, or `tokio` could be a 10.0 — you inherit their attack surface

**2. Logic bug in multi-tenant auth (~1-3% odds)**
- Tenant isolation is the highest-value target. If a request can cross tenant boundaries without auth, that's close to a 10.0
- The code looks correct, but logic bugs in authorization (not authentication) are the #1 thing code review misses — they require understanding *intent*, not just *implementation*
- This is where Burp Suite manual testing earns its money

**3. The CORS wildcard + a future XSS (~1% odds)**
- `allow_origin(Any)` means if XSS is ever found in the React app, an attacker can make cross-origin API calls. Today React's built-in escaping prevents this. One `dangerouslySetInnerHTML` changes the equation.

**4. step-ca misconfiguration (~1-2% odds)**
- Not in your code, but in the integration. If cert validation has a bypass path (the `danger_accept_invalid_certs` fallback), an attacker with network position could MITM the control plane

### What a Code Review Can't Find

This was a static code review, not a pen test. The gap matters:

| Method | Finds | Misses |
|--------|-------|--------|
| Code review (what we did) | Architectural flaws, hardcoded secrets, bad crypto, missing validation | Race conditions, timing attacks, complex multi-step logic bugs |
| DAST/fuzzing (ZAP, Burp) | Input handling edge cases, header injection, unexpected 500s | Nothing if the endpoint isn't exercised |
| Manual pen test | Tenant isolation bypass, IDOR, privilege escalation, business logic | Depends entirely on tester skill |
| Formal verification | Everything in scope | Nothing is in scope unless you write the proofs |

### Probability Estimate

| Severity | Odds of Existing Undiscovered |
|----------|-------------------------------|
| CVSS 10.0 | **~3-5%** — almost certainly not in your code, possibly in a dependency |
| CVSS 9.0+ | **~8-12%** — tenant isolation logic bug or dependency 0-day |
| CVSS 7.0+ | **~25-35%** — most likely in auth edge cases or the CORS + future XSS combo |
| CVSS 5.0+ | **~50-60%** — DoS vectors, information disclosure, or medium-severity dep issues |

The codebase is well above average for security. The biggest risk isn't what's in the code — it's what's in the dependency tree. The `bytes` fix and adding `cargo-deny` to CI are the highest-ROI actions.

---

## Industry Reality: CI Integration vs Ad-Hoc Scanning

### How Common is CI Integration by Tool Category

**Almost always in CI (>80% of mature teams):**

| Tool | Why CI is Standard |
|------|-------------------|
| Dependency scanners (cargo-audit, npm audit, Trivy, Snyk) | Fast (<30s), deterministic, zero false positives on known CVEs |
| SAST linters (clippy, eslint-security, semgrep) | Already in most CI pipelines, just enable security rules |
| Container scanning (Trivy, Snyk Container) | Runs against built image, <2min, blocks deployment of known-vulnerable images |
| License compliance (cargo-deny, license-checker) | Legal requirement for many companies, trivial to automate |
| Secret detection (gitleaks, trufflehog) | Pre-commit hook or CI gate, catches accidental credential commits |

**Sometimes in CI (~30-50% of companies):**

| Tool | Reality |
|------|---------|
| DAST scanners (ZAP, Nuclei) | Larger companies run ZAP in CI against staging on every merge to main. Most startups run it weekly/monthly on a cron. The blocker is needing a running environment — adds 5-10min to pipeline. |
| Dockerfile linting (Dockle, hadolint) | Easy to add but many teams skip it. Those who do usually gate on it. |
| testssl.sh | Rarely in CI. Usually run once at infrastructure changes. Some teams run weekly via cron. |

**Almost never in CI (<10%):**

| Tool | Why |
|------|-----|
| Burp Suite | Manual tool. Requires a human. Run quarterly or before major releases. Burp Enterprise exists ($4k+/yr) but rare. |
| jwt_tool | Manual/semi-automated. Run before launch, after auth changes. |
| sqlmap | Slow, noisy, high false-positive rate in automated mode. Run before launch. |
| nmap | Infrastructure tool, not app CI. Run at deploy-time or on a schedule. |
| Certipy | Highly specialized. Run when PKI changes. |
| Pen testing firms | Quarterly or annual engagement. SOC2/ISO27001 typically require annual. |

### What Companies Actually Do by Stage

| Stage | Typical Practice |
|-------|-----------------|
| **Pre-seed / MVP** | Nothing. Maybe npm audit because GitHub nags them. |
| **Seed / Series A** | Dependabot/Snyk free tier, maybe eslint security rules |
| **Series B** | Dependency scanning + container scanning in CI, annual pen test for SOC2 |
| **Series C+** | Full CI pipeline (deps + SAST + DAST + container), quarterly pen tests, bug bounty program |
| **Enterprise / Public** | All of the above + dedicated AppSec team, Burp Enterprise, WAF, RASP |

### Context: This Codebase is 3 Weeks Old

Thinking about pen testing at pre-seed with 3 weeks of code is uncommon — and frankly unnecessary right now. Most companies at this stage have zero security tooling and accumulate tech debt that becomes expensive to fix post-Series A when SOC2 auditors come knocking.

What you've already done (Argon2id, parameterized queries, non-root containers, rate limiting, mTLS) puts you ahead of most Series B companies in terms of security architecture. The fact that the foundations are correct means retrofitting will be minimal.

**What actually matters at this stage:**

1. **Don't ship known vulnerabilities** — `cargo-deny` + `trivy` + `npm audit` in CI. Half a day of setup, runs forever. This is the only thing worth doing now.
2. **Don't think about pen tests until you have users.** A pen test on code that's still changing daily is wasted money. The findings will be stale in a week.
3. **Don't hire an AppSec firm until SOC2 forces you.** That's typically Series A/B when enterprise customers start asking for compliance reports.
4. **Do keep the good habits you already have.** The security architecture decisions made in week 1-3 (Rust, parameterized queries, Argon2, mTLS) are 10x harder to retrofit than to build in from the start.

**The practical pre-seed security checklist is short:**
- [x] Memory-safe language (Rust) — done
- [x] Proper password hashing — done
- [x] Parameterized queries — done
- [x] Non-root containers — done
- [ ] Dependency scanning in CI — ~2 hours of work
- [ ] gitleaks pre-commit hook — ~30 minutes
- Everything else can wait until you have paying customers.

---

## Controlled DDoS Testing

The API already has rate limiting (`governor`) and load shedding (`LoadShedder` in `hardening.rs`). These need to be validated under real pressure.

### Layer 7 (Application Layer) — Most Relevant

**vegeta** — HTTP load testing (Go, single binary)
- Best for testing rate limiter and load shedder thresholds
- Constant rate attack mode finds the exact breaking point
- `echo "GET http://localhost:9080/api/v1/health" | vegeta attack -rate=5000/s -duration=30s | vegeta report`
- Can replay realistic traffic patterns from a file

**k6** (Grafana) — scriptable load testing (JS scripts, Go engine)
- Best for multi-endpoint scenarios (auth → create → poll lifecycle)
- Can model realistic user behavior alongside spike traffic
- Has stages: ramp to 10k users over 30s, hold 60s, spike to 50k
- Free and open source, paid cloud version for distributed attacks

**wrk2** — constant-throughput HTTP benchmarking
- Corrects coordinated omission (most benchmarks lie about latency under load)
- Good for finding true p99 when the server is saturated
- `wrk2 -t4 -c400 -d60s -R50000 http://localhost:9080/api/v1/health`

**oha** — Rust HTTP load generator
- Already in the Rust ecosystem, lightweight
- Real-time TUI showing latency distribution as it runs
- `oha -z 30s -c 500 -q 10000 http://localhost:9080/api/v1/health`

### Layer 4 (TCP/Connection Layer)

**hping3** — TCP SYN flood simulation
- Tests how the stack handles connection exhaustion
- `hping3 -S --flood -p 9080 target` (SYN flood)
- Good for validating kernel-level protections (SYN cookies, connection limits)

**iperf3** — bandwidth saturation
- Not an attack tool, but useful for finding the network ceiling
- Tells you "my link saturates at X Gbps" so you know what to expect

### Layer 7 Protocol-Specific

**slowloris** — slow HTTP attack
- Holds connections open with partial headers, exhausts connection pool
- `LoadShedder` max_in_flight (5000) + request timeout (30s) should defend against this — test it
- `slowhttptest -c 5000 -H -g -o output -i 10 -r 200 -t GET -u http://target:9080/api/v1/health`

**RUDY (R-U-Dead-Yet)** — slow POST body
- Sends POST data one byte at a time
- Tests whether `RequestBodyLimitLayer` and timeouts actually kill slow requests

### What to Validate

| Defense | What to Validate | Tool |
|---------|-----------------|------|
| Rate limiter (10 req/s auth) | Send 100 req/s to `/auth/login`, confirm 429s start at ~10 | vegeta |
| Load shedder (5000 in-flight) | Open 6000 concurrent connections, confirm 503s | k6 or wrk2 |
| Request timeout (30s) | Send slowloris/RUDY, confirm connections killed at 30s | slowhttptest |
| Body limit (1MB) | Send 2MB POST bodies, confirm 413 rejection | curl loop |
| Global RPS (50k) | Blast 60k req/s, confirm graceful degradation not crash | vegeta or oha |
| Recovery | Spike to 50k, drop to 0, confirm latency returns to baseline in <5s | k6 with stages |

### Recommendation for Pre-Seed

Start with **vegeta + slowhttptest**. Two tools, covers 90% of what matters:

- vegeta validates rate limiter and load shedder hold at configured thresholds
- slowhttptest validates timeouts actually kill zombie connections

Both are single binaries, no setup, run against the e2e stack. Add k6 later for realistic multi-step user flows under load.
