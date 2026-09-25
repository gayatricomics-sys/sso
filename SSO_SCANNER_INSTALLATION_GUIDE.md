# SSO Penetration Testing Scanner — Installation & Usage Guide

**Burp Suite Extension for the Complete SSO Pentest Checklist**  
Built for Gayatri's TCBC/D3/DBIQ/MWL SSO Migration Assessment (Target: Sept 30)

---

## Overview

This extension implements the **complete SSO Penetration Testing Checklist** across all 4 stages:

| Stage | Coverage | Tests |
|-------|----------|-------|
| **1. Recon** | Endpoint discovery, metadata exposure, certificate hygiene, trust mapping | Metadata endpoints, cert chain validation, wildcard detection |
| **2. Token Analysis** | XSW (XML Signature Wrapping) variants 1–8, JWT crypto flaws, assertion manipulation | alg confusion, alg=none, exp/aud/tid/iss missing, idtyp=app, signature stripping, NameID swaps |
| **3. Flow Logic** | redirect_uri validation, state parameter, consent/prompt bypass, SAML flow | Subdomain/path traversal/open-redirect variations, CSRF (missing state), implicit flow, scope escalation, unsolicited assertions |
| **4. Session/Lifecycle** | SLO, session timeouts, token revocation, cookie flags, token storage | Logout validation, idle expiry, refresh token rotation, Secure/HttpOnly/SameSite auditing, localStorage XSS |

**Key Features:**
- ✅ **Auto-Population from Proxy Traffic** — discovers endpoints, tokens, SAML assertions, and credentials automatically as you browse
- ✅ **Zero Dummy Values** — all parameters auto-filled from real observed traffic
- ✅ **Native Burp Integration** — issues appear in Burp Dashboard, Issues tab, HTML reports, and context menus
- ✅ **Full Checklist Coverage** — all test cases organized by stage in the UI
- ✅ **De-Duplication** — each endpoint scanned only once per 60 seconds to avoid noise

---

## Installation

### Requirements
- Burp Suite Professional or Community Edition
- Jython 2.7 standalone JAR

### Step 1: Obtain Jython 2.7

Download from: https://www.jython.org/download

```bash
# Extract the JAR
unzip jython-installer-2.7.x.jar
# Or use the pre-built jar
ls jython.jar
```

### Step 2: Install Extension in Burp

1. Open **Burp Suite** → **Extensions** → **Installed** tab
2. Click **Add**
3. Select **Extension type**: Python
4. Select **Jython 2.7 standalone JAR**
5. Choose the Jython JAR file from Step 1
6. Click **Select file** for the extension source code
7. Choose: `/path/to/sso_pentest_scanner.py`
8. Click **Next** → **Close**

**Expected output in Burp console:**
```
SSO Pentest Scanner loaded (all stages, auto-population enabled).
```

### Step 3: Verify Extension Loaded

- Open the **SSO Pentest (Full)** tab in Burp (appears in the top tab bar)
- You should see 7 tabs: Auto-Discovered Config, Stage 1, Stage 2, Stage 3, Stage 4, Test Results, Scanner Issues

---

## Workflow: How to Use

### Phase 1: Proxy & Auto-Population (5–10 minutes)

1. **Open the "Auto-Discovered Config" tab**
2. **Browse SSO flows** through Burp Proxy:
   - Login page (`/login`, `/signin`)
   - OAuth authorize endpoint (`/oauth2/v2.0/authorize`)
   - SAML ACS (`/acs`, `/assert`)
   - ADFS wsignin.0 (`/adfs/ls/`)
   - Logout (`/logout`, `/sign_out`)
3. **Watch the extension capture**:
   - Endpoint URLs (authorize, token, ACS, logout)
   - Parameter values (client_id, scope, nonce, redirect_uri, wtrealm, wreply)
   - Credentials (username, password from login forms)
   - Assertions (SAML responses, JWT tokens)
4. **Click "Refresh from Proxy"** to see what's been discovered

**Expected in Config panel:**
```
AUTO-DISCOVERED SSO CONFIGURATION
============================================================

ENDPOINTS:
  oauth_authorize: https://idp.example.com/oauth2/v2.0/authorize (last seen: 2 seconds ago)
  login: https://idp.example.com/login (method: POST, last seen: 15 seconds ago)
  saml_acs: https://app.example.com/Saml2/Acs (last seen: 8 seconds ago)
  wsfed: https://idp.example.com/adfs/ls/ (last seen: 12 seconds ago)

AUTO-POPULATED VALUES:
  client_id: d4e5f6a7-b8c9-4d1e-a2f3-b4c5d6e7f8a9
  scope: openid profile email offline_access
  nonce: 0b1c2d3e4f5a6b7c
  redirect_uri: https://app.example.com/auth/callback
  wtrealm: urn:microsoft:adfs:claimsxray
  wreply: https://idp.example.com/adfs/ls/
  username: testuser@example.com

CAPTURED ASSERTIONS:
  samlresponse: 2048 bytes
```

### Phase 2: Run Passive Scans (2–3 minutes)

1. **Open the "Scanner Issues" tab**
2. **Initiate a Burp Crawl or proxy browsing** of the SSO flows
3. **The extension runs passively** on every HTTP response:
   - ✓ Decodes and analyzes JWTs for `alg=none`, missing `exp`, missing `aud`, Microsoft `tid` checks, `idtyp=app`
   - ✓ Audits Set-Cookie headers for missing `Secure`, `HttpOnly`, `SameSite` flags
   - ✓ Detects OAuth authorize URLs and flags missing `state` parameter
   - ✓ Identifies and flags tokens stored in localStorage

**Expected issues** (visible in Burp Dashboard and Issues tab):
- `JWT weakness: alg=none (no signature)` [CRITICAL]
- `JWT weakness: No exp claim -- token may never expire` [MEDIUM]
- `Cookie 'session_id' missing Secure, HttpOnly, SameSite` [MEDIUM]
- `OAuth: Missing/weak state` [HIGH]
- `JWT/token stored in localStorage (XSS-vulnerable)` [HIGH]

### Phase 3: Run Active Scans (10–15 minutes)

1. **Open Burp's Active Scanner** (Scanner → Active Scan tab)
2. **Select a specific SSO endpoint** (login page, authorize URL, ACS, etc.)
3. **Right-click → "Actively scan this request (SSO Pentest)"**
   - Or let Burp's Audit + Crawl run it automatically
4. **The extension fires active checks**:
   - XSW 1–8 variants generated against SAML ACS endpoints
   - redirect_uri validation tests (subdomain, path traversal, open redirect, HTTP downgrade, @-trick)
   - state parameter bypass attempts
   - prompt=none, response_type=token implicit flow, scope escalation tests
   - OAuth/OIDC token endpoint tests
   - ADFS WS-Federation wreply/wtrealm mismatch tests
   - Session logout + re-access tests (SLO validation)
   - Cookie lifecycle and revocation validation

**Expected issues**:
- `XSW variant: XSW1-sig-stripped` [MEDIUM]
- `XSW variant: XSW2-duplicate-unsigned-first` [MEDIUM]
- `XSW variant: XSW3-reference-uri-change` [MEDIUM]
- `XSW variant: XSW4-signature-comment` [MEDIUM]
- `XSW variant: XSW5-namespace-injection` [MEDIUM]
- `XSW variant: XSW6-custom-id-reference` [MEDIUM]
- `XSW variant: XSW7-namespace-confusion` [MEDIUM]
- `XSW variant: XSW8-xslt-transform` [MEDIUM]
- `OAuth Flow: redirect_uri test (manual verification needed)` [MEDIUM]
- `OAuth: Missing/weak state` [HIGH]
- `STAGE 1: Recon needed on metadata exposure` [INFO]
- `Session: Cookie flag audit` [LOW]

### Phase 4: Manual Verification in Repeater

1. **For each generated XSW variant issue**:
   - Open the **original request** in Repeater
   - Replace the `SAMLResponse` parameter with the mutated variant (shown in the issue detail)
   - Send the request and observe if the SP accepts it (any successful auth = vulnerability)
   
2. **For redirect_uri tests**:
   - Modify the `redirect_uri` parameter in the authorize URL with each variant
   - Observe if the IdP redirects you to an unexpected domain (indicates lax validation)

3. **For OAuth/OIDC tests**:
   - Try removing `state` parameter from authorize request
   - Try `prompt=none` to force silent authentication
   - Try `response_type=token` (implicit flow) and observe token in fragment
   - Try scope escalation (add admin/superuser scopes to the request)

---

## Checklist Mapping: Which Tab Runs Which Tests

### Stage 1: Recon
**Tab:** "Stage 1: Recon"

Tests:
- Endpoint & Metadata Discovery
  - Passive: scans for `/metadata`, `/.well-known/openid-configuration`, `/FederationMetadata/2007-06/FederationMetadata.xml`
  - Active: validates endpoints are accessible and return valid metadata
  
- Certificate & Cryptographic Hygiene
  - Checks HTTPS certificate chain, expiry date, pinning
  - Detects wildcard certificates and short-lived certs
  
- Trust Relationship Mapping
  - Identifies SP ↔ IdP trust bindings (endpoint pairs, certificate associations)
  - Checks for certificate pinning at SP

**How to trigger:**
- Auto-triggered during Crawl or Audit if OAuth2/SAML endpoints are discovered
- Manual: Right-click any OAuth/SAML endpoint → "Actively scan..."

---

### Stage 2: Assertion/Token Analysis
**Tab:** "Stage 2: Token Analysis"

**Passive Tests** (no user action needed):
- JWT Cryptographic Analysis
  - `alg=none` (no signature) → CRITICAL
  - HMAC algorithms (HS256) → HIGH (alg confusion)
  - Missing `exp` claim → MEDIUM (no expiration)
  - Missing `aud` claim → MEDIUM (no audience validation)
  - Microsoft tokens missing `tid` → HIGH (tenant confusion risk)
  - `idtyp=app` on user endpoint → CRITICAL (app-only token)

- SAML Assertion Hygiene
  - Checks for unsigned assertions
  - Detects assertions without NotOnOrAfter (expiry)
  - Identifies assertions with weak signature algorithms (SHA-1)

**Active Tests** (triggered on SAML endpoints):
- XML Signature Wrapping (XSW) 1–8
  - **XSW1:** Signature stripped from assertion
  - **XSW2:** Unsigned attacker assertion placed before valid signed one
  - **XSW3:** Reference URI changed to point to attacker-controlled element
  - **XSW4:** Signature element commented out
  - **XSW5:** Namespace injection to confuse canonicalization
  - **XSW6:** Custom assertion ID injected and referenced
  - **XSW7:** Namespace prefix confusion (ds:Signature vs. Signature)
  - **XSW8:** XSLT transforms to reorder/extract elements

- SAML Assertion Manipulation
  - Signature value blanked (but structure intact)
  - Audience removed or swapped
  - Issuer swapped
  - InResponseTo (SAML request ID match) removed
  - Expiry extended to 2099
  - NameID changed to privileged user (admin@internal.local)

- OAuth/OIDC Token-Level Flaws
  - JWT alg confusion (HS256 vs RS256)
  - JWT kid (Key ID) injection
  - JWT jku (JWKS URL) injection
  - Token refresh token reuse

**How to trigger:**
- Passive: automatic on all responses (watch the Scanner Issues tab)
- Active: Right-click a SAML ACS request → "Actively scan..."

---

### Stage 3: Flow Logic Validation
**Tab:** "Stage 3: Flow Logic"

**Tests:**

**redirect_uri Validation** (7 variants):
1. **Subdomain swap:** `https://attacker.example.com/callback`
2. **@-trick:** `https://example.com%40attacker.com/callback`
3. **Path traversal:** `https://example.com/callback/../evil`
4. **Open redirect:** `https://example.com/redirect?to=https://attacker.com`
5. **HTTP downgrade:** `http://example.com/callback` (from https://example.com/authorize)
6. **Parent domain:** `https://parent.example.com/callback` (if child was registered)
7. **Subdomain wildcard:** `https://*.example.com/callback`

**state Parameter** (CSRF):
- Missing `state` parameter → HIGH (CSRF in authorization code flow)
- Static `state` value → HIGH (predictable, not unique per request)
- Empty `state` → MEDIUM (not validated)

**Authorization Request & Consent:**
- `prompt=none` accepted (silent auth, no user interaction) → may bypass MFA
- `response_type=token` (implicit flow) → token in fragment (logged in browser history)
- Scope escalation: request `admin`, `superuser`, or `.all` scopes
- PKCE omitted on public clients
- nonce parameter omitted (OIDC, can allow ID token reuse)

**SAML Flow Logic:**
- Unsolicited responses accepted (no corresponding AuthnRequest)
- InResponseTo validation disabled (response not tied to request)
- Assertion replay (same assertion accepted twice)

**Microsoft/Entra ID Specific:**
- Tenant confusion via `/common/` endpoint
- `/consumers/` multi-tenant endpoint misconfiguration
- `response_mode=fragment` for code flow (token in fragment)
- `/adminconsent` endpoint allowing consent elevation

**How to trigger:**
- Auto-triggered on OAuth authorize and SAML authorize endpoints during Crawl/Audit
- Manual: Right-click authorize URL → "Actively scan..."

---

### Stage 4: Session/Identity Lifecycle
**Tab:** "Stage 4: Session Lifecycle"

**Tests:**

**Single Logout (SLO):**
- Verify logout endpoint exists (`/logout`, `/sign_out`, `/logout?`)
- Confirm that after logout, accessing protected resource returns 401 (not 200)
- Check if token revocation happens (access_token unusable after logout)
- Verify ID token revocation (no further token refresh possible)

**Session Timeout & Expiry:**
- Measure SP idle timeout vs. IdP timeout
- Verify no token refresh after logout
- Check if refresh tokens are rotated on each use
- Detect missing absolute timeout (token never expires)

**Token Revocation:**
- POST access_token to `/token/revoke` endpoint
- Attempt to use token after revocation (should fail)
- Check if refresh token revocation cascades (all tokens revoked)

**Session Cookie Hygiene:**
- `Secure` flag missing (can be sent over HTTP) → MEDIUM
- `HttpOnly` flag missing (XSS can steal cookie) → HIGH
- `SameSite` missing or not set to `Strict` → MEDIUM (CSRF risk)
- Session cookie domain too broad (`Domain=.example.com` instead of `app.example.com`)
- Session cookie path too broad (`Path=/` instead of `/app/`)
- Cookie expires in far future or no max-age

**Token Storage:**
- Tokens in localStorage (XSS-vulnerable) → HIGH
- Tokens in sessionStorage (XSS-vulnerable) → HIGH
- Tokens in secure HTTP-only cookies (safe) → OK
- Tokens in URL fragment (logged in browser history) → MEDIUM

**How to trigger:**
- Passive: automatic on all Set-Cookie and localStorage/sessionStorage JS (watch Scanner Issues)
- Active: Right-click logout endpoint → "Actively scan..."

---

## Understanding Auto-Population

The extension captures SSO values in real-time as you browse:

```
Browsing SSO flow:
  1. Visit https://app.example.com
  2. Redirected to https://idp.example.com/oauth2/v2.0/authorize?client_id=...&redirect_uri=...&scope=...
     └─ Extension captures: client_id, redirect_uri, scope
  3. Login: POST to https://idp.example.com/login with username=user@example.com&password=...
     └─ Extension captures: username (not password, for safety)
  4. Receive authorization code, redirected to https://app.example.com/auth/callback?code=...&state=...
     └─ Extension captures: code, state
  5. Backend exchanges code for token at https://idp.example.com/oauth2/v2.0/token
     └─ Extension captures: access_token, id_token, refresh_token (in response)
  6. Receive JWT token in response headers or body
     └─ Extension decodes and checks alg, exp, aud, iss, tid, idtyp claims

Captured in Config:
  Endpoints: oauth_authorize, oauth_token, login, logout, saml_acs, wsfed, ...
  Values: client_id, scope, nonce, redirect_uri, wtrealm, wreply, username, access_token, id_token, ...
  Assertions: base64-encoded SAML responses, JWT tokens
```

### No Manual Value Entry Needed
- All parameters auto-filled from traffic
- No dummy values or placeholders
- If a value is unknown, the test is skipped (not run with dummy data)

### Validation
- Each captured endpoint is verified for accessibility
- Each token is decoded and checked for freshness (exp claim < now)
- Each SAML assertion is parsed and checked for validity (NotOnOrAfter, signature)

---

## Interpreting Findings

### Severity Levels (Burp Standard)

| Level | Meaning | Action |
|-------|---------|--------|
| **Critical** | Allows complete authentication bypass or privilege escalation | FIX IMMEDIATELY |
| **High** | Allows significant authentication weakness or data exposure | FIX BEFORE PRODUCTION |
| **Medium** | Allows partial authentication weakness, requires follow-up | FIX SOON |
| **Low** | Informational, best practice deviation | CONSIDER FIXING |
| **Info** | Reconnaissance data, no direct vulnerability | DOCUMENT |

### Common Findings & Their Meaning

**Finding:** `alg=none (no signature)`  
**Severity:** Critical  
**What it means:** JWT has no signature. Attacker can forge any JWT.  
**Remediation:** Force RS256 or other signing algorithm. Validate `alg` header.

**Finding:** `XSW1-sig-stripped`  
**Severity:** Medium (indicates code accepted, but requires manual validation that SP actually uses the forged assertion)  
**What it means:** SAML signature can be removed and assertion still accepted.  
**Remediation:** Implement strict signature validation; reject unsigned assertions.

**Finding:** `Cookie 'session_id' missing HttpOnly`  
**Severity:** High  
**What it means:** JavaScript (including XSS) can read the session cookie.  
**Remediation:** Add `HttpOnly` flag to all authentication cookies.

**Finding:** `OAuth: Missing state`  
**Severity:** High  
**What it means:** CSRF attack possible; attacker can trick user into authorizing malicious app.  
**Remediation:** Always include and validate `state` parameter (unique per request, stored in session).

**Finding:** `STAGE 1: Recon needed on metadata exposure`  
**Severity:** Info  
**What it means:** Metadata endpoints discovered; verify they're not overly exposed (e.g., no sensitive claims revealed).  
**Remediation:** Review metadata endpoints; restrict access if needed.

---

## Extending the Scanner

### Adding a Custom Test

Edit `sso_pentest_scanner.py` in the relevant Stage class:

```python
class Stage3FlowLogic(object):
    @staticmethod
    def test_custom_feature(oauth_url):
        """Test a custom flow feature."""
        issues = []
        if 'custom_param=vulnerable' in oauth_url:
            issues.append(("high", "Custom feature: vulnerable setting detected"))
        return issues

    # In doActiveScan, add:
    for issue_data in Stage3FlowLogic.test_custom_feature(url):
        issues.append(ScanIssue(service, info.getUrl(), [...],
                               "Custom: " + issue_data[0], ...))
```

Then reload the extension (Extender → Extensions → Unload, then Load again).

---

## Troubleshooting

### Extension doesn't load
- **Issue:** "No module named burp"
- **Solution:** Ensure you're using Burp's Jython environment, not system Python

### No endpoints discovered
- **Issue:** "Auto-Discovered Config shows (No endpoints discovered yet)"
- **Solution:** Ensure you're browsing SSO traffic through Burp Proxy. Check Proxy → HTTP history for SSO requests.

### Scanner runs slowly
- **Issue:** Extension performs many checks per request
- **Solution:** This is expected. The de-duplication (60s per URL) reduces re-scanning.

### False positives in JWT analysis
- **Issue:** Extension flags test/demo JWTs with alg=none
- **Solution:** This is correct behavior; verify in Repeater that the SP actually validates JWTs before dismissing.

---

## Reporting & Evidence

All issues discovered by the extension appear in:

1. **Burp Dashboard** (Summary card showing issue counts)
2. **Issues tab** (detailed view with severity, confidence, evidence)
3. **HTML Report** (Burp → Export → Full HTML Report)
4. **Context menu** (Right-click request → view evidence, send to Repeater)

Each issue includes:
- **Name:** Test case identifier (e.g., "XSW1-sig-stripped")
- **Severity:** Critical / High / Medium / Low / Info
- **Confidence:** Firm / Tentative
- **Issue Detail:** What was found, why it's a problem
- **Evidence:** HTTP request/response that triggered the finding
- **URL:** Specific endpoint tested

---

## Timeline for TCBC SSO Assessment

| Phase | Duration | Action |
|-------|----------|--------|
| **Setup** | 30 min | Install extension, load Jython, verify in Burp |
| **Discovery** | 30 min | Browse SSO flows through Proxy, watch auto-population |
| **Passive Scans** | 15 min | Let Crawler run, monitor passive findings |
| **Active Scans** | 30 min | Run Active Scanner on each endpoint type |
| **Manual Verification** | 1 hr | Validate XSW/redirect_uri/flow tests in Repeater |
| **Reporting** | 1 hr | Consolidate findings, export HTML report, draft executive summary |
| **Remediation Follow-up** | Ongoing | Track fixes, re-test before go-live |

**Total assessment time: ~4 hours (plus remediation/re-test cycles)**

---

## Support

For issues or enhancements:
- Check the Burp Extensions documentation: https://portswigger.net/burp/documentation/desktop/extensions
- Verify Jython 2.7 is the correct version (not Jython 3.x)
- Re-sync from transcript if needed for context on implementation details

**Contact:** Gayatri / Sarath (TCBC) for assessment coordination.

---

**Version:** 1.0 (Full Checklist Implementation)  
**Built:** Sept 25, 2026  
**Engagement:** TCBC/D3/DBIQ/MWL SSO Migration (Target: Sept 30)
