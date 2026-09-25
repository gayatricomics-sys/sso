# SSO Pentest Scanner — Architecture & Data Flow

## Extension Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                   BURP SUITE INTEGRATION                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │         BurpExtender (Main Class)                        │  │
│  │  ├─ registerExtenderCallbacks()                          │  │
│  │  ├─ _build_ui() → 7 Tabs                                │  │
│  │  └─ Extension Name: "SSO Pentest Scanner (Full Checklist)"│  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│         ┌────────────────┼────────────────┐                    │
│         │                │                │                    │
│         ▼                ▼                ▼                    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│  │   ITab      │ │ IHttpList   │ │ IScannerCheck│             │
│  │  (UI Tabs)  │ │ (Auto-Pop)  │ │  (Passive)  │             │
│  └─────────────┘ └─────────────┘ └─────────────┘             │
│         │                │                │                    │
└─────────│────────────────│────────────────│────────────────────┘
          │                │                │
          ▼                ▼                ▼
    [UI Layer]      [Discovery]       [Scanner]
```

## Component Interactions

### 1. AUTO-POPULATION (IHttpListener)

```
Browser Traffic
    │
    ├─ POST /login → username, password
    ├─ GET /oauth2/v2.0/authorize?client_id=...&scope=...&nonce=...
    ├─ POST /oauth2/v2.0/token (response: access_token, id_token)
    ├─ POST /acs (SAML response)
    ├─ GET /adfs/ls/?wtrealm=...&wreply=...
    │
    ▼ (captured by processHttpMessage)
┌────────────────────────────┐
│    SSOConfig Instance      │
│  ├─ endpoints{}            │
│  │  ├─ oauth_authorize     │
│  │  ├─ oauth_token         │
│  │  ├─ saml_acs            │
│  │  ├─ wsfed               │
│  │  └─ logout              │
│  ├─ values{}               │
│  │  ├─ client_id           │
│  │  ├─ scope               │
│  │  ├─ nonce               │
│  │  ├─ redirect_uri        │
│  │  ├─ wtrealm             │
│  │  └─ username            │
│  └─ assertions{}           │
│     ├─ samlresponse        │
│     └─ jwt_token           │
└────────────────────────────┘
    │
    ▼ (displayed in UI)
┌─────────────────────────────┐
│  Auto-Config Tab            │
│ "Refresh from Proxy" button  │
│  Shows:                      │
│  - Discovered endpoints      │
│  - Auto-populated values     │
│  - Captured assertions       │
└─────────────────────────────┘
```

### 2. PASSIVE SCANNING (IHttpListener → doPassiveScan)

```
HTTP Response
    │
    ├─ JWT in body/header
    │  ├─ Decode JWT (base64url)
    │  ├─ Analyze header (alg, kid, jku)
    │  ├─ Analyze payload (exp, aud, iss, sub, tid, idtyp)
    │  └─ Generate issue (if alg=none, missing claims, etc.)
    │
    ├─ Set-Cookie header
    │  ├─ Parse flags (Secure, HttpOnly, SameSite)
    │  ├─ Detect missing flags
    │  └─ Generate issue
    │
    ├─ JavaScript code
    │  ├─ Search for localStorage.setItem(...token)
    │  ├─ Search for sessionStorage.setItem(...token)
    │  └─ Generate issue (XSS-vulnerable storage)
    │
    └─ OAuth authorize URL
       ├─ Check for ?state= parameter
       ├─ Check for &state=&  (empty)
       └─ Generate issue (CSRF)

    │
    ▼
┌────────────────────────────┐
│   ScanIssue (IScanIssue)   │
│  ├─ URL                    │
│  ├─ Name                   │
│  ├─ Severity               │
│  ├─ Confidence             │
│  ├─ Detail                 │
│  └─ HTTP Evidence          │
└────────────────────────────┘
    │
    ▼
┌────────────────────────────┐
│  Burp Issues Tab           │
│  (Visible to analyst)      │
└────────────────────────────┘
```

### 3. ACTIVE SCANNING (IScannerCheck → doActiveScan)

```
Scanner Right-Click on Endpoint
    │
    └─ Endpoint type detected:
    
       ├─ /authorize (OAuth)
       │  └─ Stage 3: Flow Logic tests
       │     ├─ redirect_uri variants (7)
       │     ├─ state bypass
       │     ├─ prompt=none
       │     ├─ response_type=token
       │     └─ scope escalation
       │
       ├─ /acs (SAML)
       │  ├─ Stage 2: XSW 1–8 variants
       │  ├─ Signature stripping
       │  ├─ NameID manipulation
       │  └─ Audience/Issuer swaps
       │
       ├─ /token (Token endpoint)
       │  ├─ Stage 2: Token analysis
       │  └─ Signature bypass variants
       │
       ├─ /adfs/ls/ (ADFS)
       │  ├─ Stage 3: WS-Fed tests
       │  ├─ wreply/wtrealm mismatch
       │  └─ wctx XSS reflection
       │
       └─ /logout (Logout)
          ├─ Stage 4: SLO validation
          ├─ Post-logout access test
          └─ Token revocation test

    │
    ▼
┌────────────────────────────┐
│   Test Generation          │
│  (generate_variants)       │
│  ├─ XSW 1–8 variants       │
│  ├─ redirect_uri variants  │
│  ├─ state bypass variants  │
│  └─ flow logic variants    │
└────────────────────────────┘
    │
    ▼
┌────────────────────────────┐
│   ScanIssue (IScanIssue)   │
│  ├─ Name: "XSW1-sig-stripped"
│  ├─ Severity: "Medium"     │
│  ├─ Detail: "XSW variant   │
│  │  generated for manual   │
│  │  testing in Repeater"   │
│  └─ HTTP Evidence          │
└────────────────────────────┘
    │
    ▼
┌────────────────────────────┐
│  Burp Issues Tab           │
│  (Analyst verifies in      │
│   Repeater)                │
└────────────────────────────┘
```

## Test Case Stages

```
┌─────────────────────────────────────────────────────────────┐
│              STAGE 1: RECONNAISSANCE (13 tests)             │
├─────────────────────────────────────────────────────────────┤
│ • Endpoint discovery (metadata, authorize, token)           │
│ • Certificate analysis (expiry, wildcard, pinning)         │
│ • Trust relationship mapping                                │
│                                                             │
│ Trigger: Auto-population + manual review                   │
│ Output: Endpoints list, cert info, IdP trust chain        │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│       STAGE 2: TOKEN & ASSERTION ANALYSIS (32 tests)       │
├─────────────────────────────────────────────────────────────┤
│ Passive:                                                    │
│ • JWT alg=none, HMAC, missing claims (exp/aud/iss)        │
│ • Microsoft tid/idtyp/ver checks                           │
│ • SAML signature algorithm strength                        │
│                                                             │
│ Active:                                                     │
│ • XSW 1–8 variants (signature wrapping bypass)             │
│ • SAML assertion manipulation (audience, issuer, NameID)   │
│ • JWT alg confusion, kid/jku injection                     │
│                                                             │
│ Trigger: Passive on all responses, Active on SAML/JWT      │
│ Output: Signature bypass issues, crypto weaknesses         │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│        STAGE 3: AUTHORIZATION FLOW LOGIC (25 tests)        │
├─────────────────────────────────────────────────────────────┤
│ • redirect_uri validation (7 variants)                      │
│ • state parameter bypass (CSRF)                             │
│ • Consent/prompt manipulation (prompt=none)                │
│ • Implicit flow detection (response_type=token)            │
│ • SAML flow logic (unsolicited, InResponseTo, replay)      │
│ • ADFS/WS-Fed specifics (wreply, wtrealm, wctx)            │
│ • Microsoft Entra ID (/common/, /consumers/, /adminconsent)│
│                                                             │
│ Trigger: Active on OAuth/SAML/ADFS authorize endpoints     │
│ Output: Flow bypass issues, redirect/CSRF issues           │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│    STAGE 4: SESSION & IDENTITY LIFECYCLE (20 tests)        │
├─────────────────────────────────────────────────────────────┤
│ Passive:                                                    │
│ • Session cookie flags (Secure, HttpOnly, SameSite)        │
│ • Token storage hygiene (localStorage vs. cookies)         │
│ • Missing expiration (token never expires)                 │
│                                                             │
│ Active/Manual:                                              │
│ • Single Logout (SLO) validation                           │
│ • Session timeout measurement (idle vs. absolute)          │
│ • Token revocation (post-logout)                           │
│ • Refresh token rotation                                   │
│                                                             │
│ Trigger: Passive on all responses, Manual on logout URL    │
│ Output: Session management issues, XSS/CSRF risks          │
└─────────────────────────────────────────────────────────────┘
```

## Data Flow: Discovery to Report

```
┌─────────────────────────────────────────────────────────────┐
│ Step 1: BROWSER TRAFFIC → BURP PROXY                        │
├─────────────────────────────────────────────────────────────┤
│  User browses SSO flow:                                     │
│  • Login (POST /login)                                      │
│  • Authorize (GET /authorize?client_id=...&redirect_uri=...) │
│  • Token exchange (POST /token, response with JWT)          │
│  • ACS (POST /acs with SAMLResponse)                        │
│  • Logout (GET /logout)                                    │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 2: AUTO-POPULATION (IHttpListener.processHttpMessage)  │
├─────────────────────────────────────────────────────────────┤
│  Extension intercepts:                                      │
│  • Parses URLs → extracts endpoints                         │
│  • Parses request/response bodies                           │
│  • Decodes Base64 assertions                               │
│  • Regex searches for JWT, credentials, tokens             │
│  • Updates SSOConfig instance                              │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3a: PASSIVE SCANNING (on all responses)                │
├─────────────────────────────────────────────────────────────┤
│  • Decode JWTs, analyze claims                             │
│  • Parse Set-Cookie headers, check flags                   │
│  • Grep for localStorage.setItem() JS patterns             │
│  • Check OAuth URL for state parameter                     │
│  • Generate ScanIssue for each finding                     │
└─────────────────────────────────────────────────────────────┘
         │                                  │
         │                                  ▼
         │                         ┌──────────────────────┐
         │                         │  Passive Issues      │
         │                         │  • JWT weaknesses    │
         │                         │  • Cookie flags      │
         │                         │  • XSS storage       │
         │                         │  • CSRF (state)      │
         │                         └──────────────────────┘
         │
         ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 3b: ACTIVE SCANNING (manual trigger per endpoint)      │
├─────────────────────────────────────────────────────────────┤
│  User right-clicks endpoint in Proxy:                       │
│  "Actively scan this request (SSO Pentest)"                │
│                                                             │
│  Extension:                                                │
│  • Detects endpoint type (OAuth, SAML, ADFS, logout)       │
│  • Generates test variants per endpoint type               │
│  • Creates ScanIssue for each variant                      │
│  • Issues show "manual testing required in Repeater"       │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 4: MANUAL VERIFICATION (Repeater)                      │
├─────────────────────────────────────────────────────────────┤
│  Analyst:                                                  │
│  1. Opens issue → reads variant details                    │
│  2. Copies request to Repeater                             │
│  3. Modifies request (SAMLResponse, redirect_uri, etc.)    │
│  4. Sends request                                          │
│  5. Observes response (200/302 = vuln, 403 = safe)         │
│  6. Documents finding (PASSED or FAILED)                   │
└─────────────────────────────────────────────────────────────┘
                          │
                          ▼
┌─────────────────────────────────────────────────────────────┐
│ Step 5: REPORTING & CONSOLIDATION                           │
├─────────────────────────────────────────────────────────────┤
│  • Export Burp HTML Report                                 │
│  • Cross-check against Checklist Reference                 │
│  • Count tests passed/failed per stage                     │
│  • Consolidate by severity (Critical, High, Medium, Low)   │
│  • Write executive summary & timeline                      │
│  • Deliver to Gayatri/Sarath for remediation               │
└─────────────────────────────────────────────────────────────┘
```

## Class Structure

```
BurpExtender
├── IBurpExtender (main entry point)
├── ITab (UI panels)
├── IHttpListener (auto-population)
├── IScannerCheck (passive + active scanning)
├── IContextMenuFactory (right-click menu)
│
├─ SSOConfig
│  ├─ endpoints{} (detected OAuth/SAML endpoints)
│  ├─ values{} (client_id, scope, nonce, redirect_uri, etc.)
│  ├─ assertions{} (SAML responses, JWT tokens)
│  └─ methods: update_from_request(), get_best_value()
│
├─ Stage1Recon
│  ├─ check_metadata_exposure()
│  └─ check_certificate_pinning()
│
├─ Stage2TokenAnalysis
│  ├─ analyze_jwt()
│  └─ test_xsw_variants()
│
├─ XSWVariantGenerator
│  └─ generate() → [(variant_name, mutated_xml), ...]
│
├─ Stage3FlowLogic
│  ├─ test_redirect_uri_validation()
│  ├─ test_state_parameter()
│  ├─ test_prompt_and_consent()
│
├─ Stage4SessionLifecycle
│  ├─ check_cookie_flags()
│  └─ check_token_storage()
│
├─ ScanIssue (IScanIssue)
│  ├─ getUrl()
│  ├─ getIssueName()
│  ├─ getSeverity()
│  ├─ getConfidence()
│  ├─ getIssueDetail()
│  └─ getHttpMessages()
│
└─ Helper Functions
   ├─ b64url_decode/encode()
   ├─ rand_str()
   ├─ parse_url()
   └─ sev_map() (severity → Burp enum)
```

## UI Tab Layout

```
┌─────────────────────────────────────────────────────────────┐
│          SSO Pentest Scanner — Main Panel                   │
├─────────────────────────────────────────────────────────────┤
│ [1]Auto-Config  [2]Stage1  [3]Stage2  [4]Stage3  [5]Stage4   │
│ [6]Results      [7]Issues                                   │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│ Tab [1] AUTO-DISCOVERED CONFIG                             │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ [Refresh from Proxy] button                             │ │
│ │                                                         │ │
│ │ ENDPOINTS:                                              │ │
│ │   oauth_authorize: https://idp.../authorize (2s ago)   │ │
│ │   saml_acs: https://app.../Acs (8s ago)                │ │
│ │                                                         │ │
│ │ AUTO-POPULATED VALUES:                                  │ │
│ │   client_id: d4e5f6a7-...                              │ │
│ │   redirect_uri: https://app.../callback                │ │
│ │   username: testuser@example.com                        │ │
│ │                                                         │ │
│ │ CAPTURED ASSERTIONS:                                    │ │
│ │   samlresponse: 2048 bytes                             │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [2] STAGE 1: RECON                                     │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Endpoint & Metadata Discovery                          │ │
│ │ • Discover authorize, token, metadata endpoints        │ │
│ │ • Validate endpoint accessibility                      │ │
│ │ • Certificate & Cryptographic Hygiene                 │ │
│ │ • Check HTTPS cert chain, expiry, pinning              │ │
│ │ • Trust Relationship Mapping                           │ │
│ │ • Identify SP ↔ IdP trust bindings                    │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [3] STAGE 2: TOKEN ANALYSIS                            │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ XSW 1–8, JWT crypto flaws, SAML manipulation           │ │
│ │ • alg=none, HMAC confusion, missing claims             │ │
│ │ • XSW signature wrapping (8 variants)                  │ │
│ │ • Audience/Issuer/NameID swaps                         │ │
│ │ • Microsoft tid/idtyp checks                           │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [4] STAGE 3: FLOW LOGIC                                │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ redirect_uri, state, consent, SAML/ADFS flow           │ │
│ │ • redirect_uri variants (7 tests)                      │ │
│ │ • state parameter bypass (CSRF)                        │ │
│ │ • prompt=none, implicit flow, scope escalation         │ │
│ │ • ADFS wreply/wtrealm mismatch                         │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [5] STAGE 4: SESSION/LIFECYCLE                         │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ SLO, timeouts, revocation, cookies, storage             │ │
│ │ • Verify logout clears session (SLO)                   │ │
│ │ • Measure timeout (idle vs. absolute)                  │ │
│ │ • Check token revocation (post-logout)                 │ │
│ │ • Audit cookie flags (Secure, HttpOnly, SameSite)      │ │
│ │ • Check token storage (localStorage vs. cookies)       │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [6] TEST RESULTS                                       │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Test Name | Severity | Status | Description             │ │
│ │ ──────────┼──────────┼────────┼─────────────────────── │ │
│ │ JWT:alg=none | HIGH | FAILED | Found in response       │ │
│ │ XSW1:sig-stripped | MED | PASSED | Manual verify needed │ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
│ Tab [7] SCANNER ISSUES                                     │
│ ┌─────────────────────────────────────────────────────────┐ │
│ │ Severity | Issue | Details                             │ │
│ │ ──────────┼──────────┼────────────────────────────────┐│ │
│ │ HIGH | JWT weakness: alg=none | Header: {"alg":"none"} ││ │
│ │ MED | XSW1-sig-stripped | Manual testing in Repeater   ││ │
│ │ MED | Cookie missing HttpOnly | Set-Cookie: id=x       ││ │
│ │ HIGH | State parameter missing | /authorize?... (no &state) ││ │
│ └─────────────────────────────────────────────────────────┘ │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

---

**Diagram Key:**
- `[n]` = Tab number
- `{}` = Python dictionary/class
- `()` = Method/function
- `→` = Data flow
- `├─` = Child element
- `▼` = Sequential flow

