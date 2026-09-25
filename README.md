# SSO Penetration Testing Scanner for Burp Suite

**Automated Security Assessment & Vulnerability Scanner for Single Sign-On (OAuth 2.0, OpenID Connect, SAML 2.0, ADFS, JWT)**

```
Target Assessment: TCBC / D3 / DBIQ / MWL SSO Migration Assessment
Target Completion: September 30, 2026
Status: ✅ Ready for Production Deployment
Repository: https://github.com/gayatricomics-sys/sso.git
```

---

## 📋 Table of Contents

- [Overview](#overview)
- [Repository Files](#repository-files)
- [Prerequisites](#prerequisites)
- [Installation Instructions](#installation-instructions)
- [Operating Workflow & Instructions](#operating-workflow--instructions)
- [Extension UI Breakdown](#extension-ui-breakdown)
- [Test Coverage & Vulnerabilities](#test-coverage--vulnerabilities)
- [Reporting & Evidence Export](#reporting--evidence-export)
- [Troubleshooting & FAQ](#troubleshooting--faq)

---

## 🔍 Overview

The **SSO Penetration Testing Scanner** (`sso_pentest_scanner.py`) is a native Burp Suite extension written in Jython (Python 2.7). It automates vulnerability scanning across all 4 stages of the SSO Penetration Testing Checklist for OAuth 2.0, OIDC, SAML 2.0, ADFS, and JWT implementations.

### Key Capabilities
- 🔍 **Auto-Discovery**: Automatically extracts endpoints, client IDs, SAML assertions, JWTs, and redirect URIs directly from Burp Proxy traffic without requiring manual entry.
- 🧪 **48 Automated Tests**: Scans for XML Signature Wrapping (XSW 1–8), JWT `alg=none` / HMAC algorithm confusion, `redirect_uri` bypasses, state parameter omission (CSRF), and missing security flags.
- 📊 **Native Burp Integration**: Integrates directly with Burp's Scanner, Issues Tab, Dashboard, Repeater, and HTML Report Generator.
- ⚡ **De-duplication Logic**: Prevents scan flooding by de-duplicating active scan requests (60-second window per endpoint).
- 🛠️ **7 Tab Custom UI**: Custom tabbed interface providing full visibility into discovered parameters, stage-specific tests, and execution results.

---

## 📁 Repository Files

| File | Purpose |
|------|---------|
| [`sso_pentest_scanner.py`](file:///Users/gayatri/Documents/sso/sso_pentest_scanner.py) | **Main Burp Extension** (664 lines, Jython 2.7 compatible) |
| [`README.md`](file:///Users/gayatri/Documents/sso/README.md) | Project Overview & Comprehensive Operating Instructions |
| [`SSO_SCANNER_INSTALLATION_GUIDE.md`](file:///Users/gayatri/Documents/sso/SSO_SCANNER_INSTALLATION_GUIDE.md) | Step-by-step setup, technical workflow, and vulnerability remediation guide |
| [`SSO_PENTEST_CHECKLIST_REFERENCE.md`](file:///Users/gayatri/Documents/sso/SSO_PENTEST_CHECKLIST_REFERENCE.md) | Complete 90-item SSO testing checklist matrix & tab mapping |
| [`ARCHITECTURE.md`](file:///Users/gayatri/Documents/sso/ARCHITECTURE.md) | Extension internal architecture and code structure documentation |
| [`DELIVERY_SUMMARY.md`](file:///Users/gayatri/Documents/sso/DELIVERY_SUMMARY.md) | Deliverable summary and deployment readiness report |
| [`INDEX.txt`](file:///Users/gayatri/Documents/sso/INDEX.txt) | Quick text index of project files |

---

## ⚙️ Prerequisites

Before installing the extension, ensure you have:

1. **Burp Suite** (Community Edition or Professional, v2020.1 or newer).
2. **Jython 2.7 Standalone JAR**:
   - Download `jython-standalone-2.7.3.jar` (or 2.7.2+) from [jython.org Downloads](https://www.jython.org/download).
   - Save the `.jar` file to a permanent directory on your computer (e.g., `~/BurpExtensions/jython-standalone-2.7.3.jar`).

---

## 🚀 Installation Instructions

### Step 1: Configure Python Environment in Burp Suite
1. Open Burp Suite.
2. Go to **Extensions** (or **Extender**) → **Options**.
3. Under **Python Environment**:
   - Click **Select file** next to *Location of Jython standalone JAR file*.
   - Browse and select your downloaded `jython-standalone-2.7.3.jar`.

### Step 2: Load the Scanner Extension
1. Clone or download this repository:
   ```bash
   git clone https://github.com/gayatricomics-sys/sso.git
   cd sso
   ```
2. In Burp Suite, navigate to **Extensions** → **Installed** (or **Extender** → **Extensions**).
3. Click **Add**.
4. Set **Extension type** to **Python**.
5. Click **Select file** and select [`sso_pentest_scanner.py`](file:///Users/gayatri/Documents/sso/sso_pentest_scanner.py).
6. Click **Next**.
7. Confirm that the extension output pane shows:
   ```text
   [+] SSO Pentest Scanner v1.0 Loaded Successfully!
   [+] Native Burp Scanner integration active.
   [+] Custom UI Tabs initialized.
   ```
8. Verify that a new main tab titled **"SSO Pentest (Full)"** appears in the top menu bar of Burp Suite.

---

## 📖 Operating Workflow & Instructions

Follow this 4-phase workflow during your penetration test:

```
┌─────────────────┐     ┌──────────────────┐     ┌─────────────────┐     ┌──────────────────┐
│  Phase 1:       │ ──► │  Phase 2:        │ ──► │  Phase 3:       │ ──► │  Phase 4:        │
│  Auto-Discovery │     │  Passive Scans   │     │  Active Scans   │     │  Manual Verification
└─────────────────┘     └──────────────────┘     └─────────────────┘     └──────────────────┘
```

### Phase 1: Auto-Discovery & Configuration (5–10 min)
1. Ensure Burp Intercept is OFF, but Proxy is actively logging HTTP history.
2. Open your target application in a browser configured to use Burp Proxy.
3. Perform standard login flows (OAuth 2.0, OpenID Connect, SAML 2.0, or ADFS login/logout).
4. Go to **SSO Pentest (Full)** → **Auto-Discovered Config** tab in Burp.
5. Click **Refresh from Proxy**.
6. Review auto-populated parameters (Authorize Endpoints, Token Endpoints, SAML ACS URLs, Client IDs, JWT Tokens).

### Phase 2: Passive Scanning (Automatic)
- Passive scanning runs automatically in the background on all HTTP traffic passing through Burp Proxy.
- It detects missing security flags (`HttpOnly`, `Secure`, `SameSite`), exposed JWT tokens in URL parameters or `localStorage`, and weak/missing `state` parameters.
- Check findings under **Extensions** → **SSO Pentest (Full)** → **Scanner Issues** or Burp's main **Dashboard** / **Target** → **Issues**.

### Phase 3: Active Scanning (10–15 min)
1. Go to Burp's **Proxy** → **HTTP History** or **Target** tab.
2. Locate key SSO requests (e.g., SAML ACS `POST` request, OAuth Authorization GET, Token POST).
3. **Right-click** the target request and select:
   > **Actively scan this request (SSO Pentest)**
4. The scanner will generate and send test payloads for:
   - **XSW 1–8 Variants** (XML Signature Wrapping on SAML Responses).
   - **`redirect_uri` Validation Bypasses** (wildcards, path traversal, open redirectors).
   - **JWT Cryptographic Flaws** (`alg=none`, HMAC algorithm confusion).
   - **CSRF & Flow Bypasses** (missing `state`, `prompt=none`, missing `tid`).

### Phase 4: Manual Verification in Repeater (10–20 min)
1. For any issue reported by the scanner, right-click the item and send it to **Repeater**.
2. Verify responses manually:
   - **SAML XSW**: Check if signature verification passes when elements are cloned/moved.
   - **`redirect_uri`**: Check if tokens or auth codes are leaked to unauthorized domains.
   - **JWT**: Verify if modified JWT tokens with `alg=none` are accepted by the server.

---

## 🖥️ Extension UI Breakdown

The extension provides 7 specialized tabs inside the **SSO Pentest (Full)** menu:

1. **Auto-Discovered Config**: Displays all live endpoints, client credentials, SAML assertions, and tokens intercepted from Proxy traffic.
2. **Stage 1: Recon**: Displays endpoint discovery status, metadata endpoints (`.well-known/openid-configuration`), certificate validation, and trust mapping.
3. **Stage 2: Token Analysis**: Shows results for XML Signature Wrapping (XSW 1–8), JWT algorithm checks, signature enforcement, and claim validation.
4. **Stage 3: Flow Logic**: Displays tests for `redirect_uri` validation, `state` CSRF mitigation, consent page bypass, and ADFS-specific logic.
5. **Stage 4: Session Lifecycle**: Shows single log-out (SLO) tests, session timeout enforcement, token revocation, and cookie flag security.
6. **Test Results**: Provides a summary table of all automated test runs, payload statuses, and target URLs.
7. **Scanner Issues**: Native table view listing all discovered vulnerabilities categorized by severity (Critical, High, Medium, Low, Information).

---

## 🛡️ Test Coverage & Vulnerabilities

| Stage | Focus Area | Total Tests | Automated | Manual |
|-------|------------|-------------|-----------|--------|
| **Stage 1: Recon** | Endpoint discovery, metadata, TLS/certificates, trust mapping | 13 | 7 | 6 |
| **Stage 2: Token Analysis** | XSW 1–8, JWT crypto (`alg=none`), SAML response manipulation | 32 | 18 | 14 |
| **Stage 3: Flow Logic** | `redirect_uri` bypasses, `state` CSRF, consent, ADFS rules | 25 | 13 | 12 |
| **Stage 4: Session Lifecycle** | Single Log-Out (SLO), cookie security, token storage, revocation | 20 | 10 | 10 |
| **TOTAL** | **Full 4-Stage SSO Checklist Coverage** | **90** | **48 (53%)** | **42 (47%)** |

### Key Vulnerabilities Detected

- **Critical**: `alg=none` JWT acceptance, SAML Assertion accepted without signature, `idtyp=app` privilege escalation on user endpoints, raw tokens exposed in `localStorage`.
- **High**: XSW 1–8 signature wrapping bypasses, missing `state` parameter (OAuth CSRF), JWT HMAC key confusion, missing `HttpOnly` flag on session cookies, missing Microsoft tenant ID (`tid`).
- **Medium**: `redirect_uri` open redirect / domain bypass, missing JWT expiry (`exp`) or audience (`aud`) claims, `prompt=none` authorization bypass, SameSite attribute missing.

---

## 📊 Reporting & Evidence Export

1. Open **Burp Dashboard** → **Issues** pane (or **Target** → **Site Map**).
2. Filter issues by selecting the target host.
3. Select the SSO issues generated by **SSO Pentest Scanner**.
4. Right-click and choose **Report selected issues** (or **Generate HTML Report** in Burp Pro).
5. Cross-reference findings with [`SSO_PENTEST_CHECKLIST_REFERENCE.md`](file:///Users/gayatri/Documents/sso/SSO_PENTEST_CHECKLIST_REFERENCE.md) for detailed remediation guidance.

---

## ❓ Troubleshooting & FAQ

| Problem | Cause | Resolution |
|---------|-------|------------|
| **Extension fails to load in Burp** | Jython JAR not configured | Ensure `jython-standalone-2.7.3.jar` is specified under **Extensions** → **Options** → **Python Environment**. |
| **"SSO Pentest (Full)" tab not showing** | Loading error | Check **Extensions** → **Installed** → **Errors** tab for Python traceback details. |
| **No endpoints in Auto-Config tab** | Proxy traffic not intercepted/refreshed | Drive SSO traffic through Burp Proxy first, then click **Refresh from Proxy**. |
| **Active scan does not trigger** | Duplicate request within 60 seconds | Wait 60 seconds or target a unique endpoint URL/parameter set. |

---

## 📝 License & Project Contact

- **Project**: TCBC / D3 / DBIQ / MWL SSO Migration Security Assessment
- **Maintained By**: Security Assessment Team (Gayatri & Sarath)
- **Reference**: For full technical details, consult [`SSO_SCANNER_INSTALLATION_GUIDE.md`](file:///Users/gayatri/Documents/sso/SSO_SCANNER_INSTALLATION_GUIDE.md).
