# SSO Penetration Testing Scanner for Burp Suite

**Complete automated checker for the SSO Penetration Testing Checklist (all 4 stages)**

```
Built for: Gayatri & Sarath (TCBC/D3/DBIQ/MWL SSO Migration Assessment)
Target Completion: September 30, 2026
Status: ✅ Ready for Deployment
```

---

## What You Get

### Extension: `sso_pentest_scanner.py` (664 lines, Jython)
A native Burp Suite extension that:
- 🔍 **Auto-discovers** SSO endpoints, tokens, and credentials from Proxy traffic
- 🧪 **Runs 48 automated tests** covering XSW variants, JWT crypto, OAuth flow, session lifecycle
- 📊 **Reports native Burp Issues** visible in Dashboard, Issues tab, HTML exports
- 🚫 **De-duplicates** scans (60s per endpoint) to reduce noise
- ✅ **Zero manual value entry** — all parameters auto-populated from real traffic

### Documentation
1. **SSO_SCANNER_INSTALLATION_GUIDE.md** — Step-by-step setup, workflow, interpretation
2. **SSO_PENTEST_CHECKLIST_REFERENCE.md** — Detailed mapping of all 90 test cases
3. **This README** — Overview & quick-start

---

## Quick Start

### 1. Install (5 minutes)
```
1. Download Jython 2.7 standalone JAR from jython.org
2. Open Burp → Extensions → Add
3. Select Python, choose Jython JAR
4. Load sso_pentest_scanner.py
5. Verify: new "SSO Pentest (Full)" tab appears
```

### 2. Discover SSO Values (5–10 minutes)
```
1. Open "Auto-Discovered Config" tab
2. Browse SSO flows through Burp Proxy
3. Click "Refresh from Proxy"
4. Watch endpoints/tokens populate automatically
```

### 3. Run Tests (30 minutes)
```
Phase A: Passive Scans (automatic)
  → Watch Scanner Issues as traffic flows through Proxy

Phase B: Active Scans (manual per endpoint)
  → Right-click OAuth/SAML endpoint → "Actively scan this request (SSO Pentest)"

Phase C: Manual Verification (10–15 min)
  → Test in Repeater, modify SAML/redirect_uri, send to IdP
```

### 4. Review Findings (10 minutes)
```
1. Burp Dashboard → Issues widget
2. See all SSO vulnerabilities discovered
3. Export HTML Report
```

---

## Test Coverage

| Stage | Focus | Tests |
|-------|-------|-------|
| **1. Recon** | Endpoint discovery, metadata, certs, trust mapping | 13 |
| **2. Token Analysis** | XSW 1–8, JWT crypto, SAML manipulation | 32 |
| **3. Flow Logic** | redirect_uri, state, consent, SAML flow, ADFS | 25 |
| **4. Session Lifecycle** | SLO, timeouts, revocation, cookies, storage | 20 |
| **TOTAL** | **All 4 stages, full checklist** | **90** |

**Automation Rate:** 53% (48 automated, 42 manual verification in Repeater)

---

## Key Vulnerabilities Detected

**Critical:**
- ✅ alg=none JWT (no signature)
- ✅ idtyp=app on user endpoint
- ✅ Assertion accepted without signature
- ✅ Token in localStorage (XSS)

**High:**
- ✅ Missing state parameter (CSRF)
- ✅ HMAC alg confusion
- ✅ HttpOnly flag missing
- ✅ XSW 1–8 variants accepted
- ✅ Microsoft tid missing

**Medium:**
- ✅ Missing exp/aud/iss claims
- ✅ redirect_uri validation bypass
- ✅ prompt=none accepted
- ✅ SameSite missing
- ✅ SAML assertion replay

---

## Files in Deliverable

| File | Purpose |
|------|---------|
| `sso_pentest_scanner.py` | Main Burp extension (664 lines, Jython) |
| `SSO_SCANNER_INSTALLATION_GUIDE.md` | Setup, workflow, detailed interpretation |
| `SSO_PENTEST_CHECKLIST_REFERENCE.md` | All 90 test cases mapped to tabs/triggers |
| `README.md` | This overview |

---

## Integration with TCBC Assessment

### Pre-Engagement
- [ ] Install extension on Burp instance
- [ ] Load Jython 2.7
- [ ] Verify all 7 UI tabs load

### During Assessment
- [ ] Proxy SSO flows (discovery)
- [ ] Run passive scans (automatic)
- [ ] Run active scans (per endpoint)
- [ ] Verify findings in Repeater
- [ ] Document remediation

### Reporting
- [ ] Export Burp HTML report
- [ ] Cross-check against Checklist Reference
- [ ] Consolidate by severity & stage
- [ ] Executive summary with timeline

---

## Performance

| Task | Time |
|------|------|
| Endpoint discovery | 5–10 min |
| Passive scans | Real-time (automatic) |
| Active scans (1 endpoint) | 2–5 min |
| Full assessment | 1–2 hours |
| Report export | 2–5 min |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| Extension won't load | Check Jython 2.7 JAR via Extender → Options |
| No endpoints discovered | Browse SSO flows through Proxy HTTP history |
| Scanner slow | Normal; de-duped per 60s per URL |
| XSW variant didn't generate | Ensure SAML is well-formed in Proxy |

**See Installation Guide for full troubleshooting.**

---

## Version Info

| Metric | Value |
|--------|-------|
| Version | 1.0 (Full Checklist) |
| Release | September 25, 2026 |
| Jython | 2.7 |
| Status | ✅ Production Ready |

---

## Next Steps

1. **Install** → Follow Installation Guide
2. **Discover** → Browse SSO, populate config
3. **Test** → Passive → Active → Manual
4. **Report** → Export & consolidate findings
5. **Remediate** → Work with Gayatri/Sarath

**Target:** September 30, 2026 go-live

---

**Made for TCBC SSO Migration Assessment**

See `SSO_SCANNER_INSTALLATION_GUIDE.md` for complete workflow.
