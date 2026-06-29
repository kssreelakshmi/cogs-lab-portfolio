# Change Plan — CHANGE-001

**Change ID:** CHANGE-001
**Type:** Normal Change
**Status:** PENDING APPROVAL
**Date:** 2026-06-29
**Author:** Sreelakshmi

## Description

Update Nginx TLS configuration to prefer TLS 1.3 and disable TLS 1.2. TLS 1.3 has faster handshakes and removes legacy cipher vulnerabilities present in TLS 1.2.

## Justification

TLS 1.3 is the current industry standard. TLS 1.2 is still widely supported but has known weaknesses in certain cipher suites. Upgrading to TLS 1.3-only aligns the lab environment with current best practice and mirrors what enterprise security teams require on production gateways.

## Scope

- **System:** vpn-server (140.245.218.252)
- **File:** /etc/nginx/sites-available/lab-tls
- **Impact:** TLS 1.2 clients will be rejected after change

## Execution Steps

1. Backup current Nginx TLS config
2. Update ssl_protocols line to TLSv1.3 only
3. Test config with `sudo nginx -t`
4. Reload Nginx with `sudo systemctl reload nginx`
5. Verify TLS 1.3 working with openssl
6. Verify TLS 1.2 is rejected with openssl

## Rollback Trigger

- `nginx -t` fails after config change
- `openssl s_client -tls1_3` does not confirm TLS 1.3
- Site becomes unreachable after reload

## Rollback Steps

1. `sudo cp /etc/nginx/sites-available/lab-tls.backup /etc/nginx/sites-available/lab-tls`
2. `sudo nginx -t && sudo systemctl reload nginx`
3. Verify service restored with `curl -k https://localhost`

## Rollback Time

2 minutes

## Maintenance Window

Immediate — lab environment, no customer impact

## Sign-Off

- [ ] Change plan reviewed
- [ ] Backup confirmed before execution
- [ ] Rollback plan tested

---

## Execution Log

### Change Executed — 2026-06-29

**Step 1 — Backup taken:**
sudo cp /etc/nginx/sites-available/lab-tls /etc/nginx/sites-available/lab-tls.backup

**Step 2 — Config updated:** ssl_protocols changed from TLSv1.2 TLSv1.3 to TLSv1.3 only

**Step 3 — nginx -t result:** syntax ok — test successful

**Step 4 — Nginx restarted:** sudo systemctl restart nginx

**Step 5 — Verification:**
- TLS 1.3: New, TLSv1.3, Cipher is TLS_AES_256_GCM_SHA384 ✅
- TLS 1.2: Cipher 0000 — connection refused ✅

**Status: COMPLETED**

---

## Rollback Log

### Rollback Triggered — 2026-06-29

**Trigger:** Deliberate bad config — ssl_protocolz typo introduced

**nginx -t result:** unknown directive "ssl_protocolz" — test FAILED

**Rollback executed:**
sudo cp /etc/nginx/sites-available/lab-tls.backup /etc/nginx/sites-available/lab-tls
sudo nginx -t && sudo systemctl reload nginx

**nginx -t after rollback:** syntax ok — test successful

**Service verified:** curl -k https://localhost returned COGS Lab page ✅

**Rollback time:** Under 2 minutes

**Status: ROLLED BACK SUCCESSFULLY**
