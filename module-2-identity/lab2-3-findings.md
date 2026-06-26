# Lab 2.3 — TOTP MFA: Generate and Verify Time-Based OTPs

## My Setup

- **Python:** 3.10
- **Libraries:** pyotp 2.10.0, qrcode 8.2, pillow 12.2.0
- **Authenticator App:** Google Authenticator and Authy
- **TOTP Secret:** `2EVNOLNQDJCZK5O6AIUYIBBUPGBGSL43`
- **Issuer:** InstaSafe Lab
- **User:** testuser@instasafe.local

---

## Part 1 — TOTP Secret Generated and QR Code Created

I installed the required libraries and ran the TOTP generator script:

```bash
pip3 install pyotp qrcode pillow
```

The script generated a random Base32 secret, created a provisioning URI in the `otpauth://` format, and saved a QR code PNG to `/tmp/totp_qr.png`.

**Terminal output showing secret generation:**
![TOTP secret generated](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-1-image.png)

**Generated QR code:**
![QR code](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-2-image.png)

The provisioning URI format is:
```
otpauth://totp/InstaSafe%20Lab:testuser%40instasafe.local?secret=2EVNOLNQDJCZK5O6AIUYIBBUPGBGSL43&issuer=InstaSafe%20Lab
```

This URI encodes everything the authenticator app needs — the algorithm (TOTP), the account name, the issuer, and the shared secret. When scanned, the app stores the secret locally and uses it with the current time to generate OTPs independently — no network connection needed after setup.

---

## Part 2 — Matching OTP: Terminal vs Authenticator App

I scanned the QR code with Google Authenticator first, then added the same secret manually to Authy for screenshot purposes (Google Authenticator blocks screenshots as a security feature — which is itself a good security practice worth noting).

I ran a quick OTP check:

```bash
python3 -c "import pyotp; t=pyotp.TOTP('2EVNOLNQDJCZK5O6AIUYIBBUPGBGSL43'); print(f'Current OTP: {t.now()}')"
```

**Terminal showing OTP `201633`:**
![Terminal OTP](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-3-image.png)

**Authy showing the same OTP `201633` for InstaSafe Lab:**
![Authy OTP match](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-4-image.jpeg)

Both show `201633` at the same moment. Neither side communicated with the other — they independently computed the same OTP using the shared secret and the current Unix timestamp. This is exactly how TOTP works per RFC 6238.

---

## Part 3 — OTP Rotation After 31 Seconds

I ran the verification script which:
1. Generated the current OTP
2. Verified it returns `True`
3. Verified a wrong OTP (`000000`) returns `False`
4. Waited 31 seconds
5. Generated a new OTP and confirmed it's different

**OTP rotation output:**
![OTP rotation](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-5-image.png)

Results:
- **First OTP:** `233439` → `verify()` returned `True`
- **Wrong OTP:** `000000` → `verify()` returned `False`
- **After 31 seconds:** `602053` → completely different
- **Are they different:** `True`

This proves TOTP rotates every 30 seconds. The old code `233439` is now invalid — even if someone intercepted it, it's useless after the window expires. This is what makes TOTP significantly more secure than static passwords or SMS OTP (which has no automatic expiry).

---

## Part 4 — What Causes TOTP Failure in an Enterprise?

TOTP failures are one of the most common MFA support issues. When a user says "my OTP isn't working", there are three main root causes:

---

### Root Cause 1 — Clock Drift (Most Common)

TOTP is entirely time-based. Both the server and the user's phone compute the OTP using the current Unix timestamp divided into 30-second windows. If the phone clock is even 90 seconds off from the server clock, they'll be in completely different time windows and generate different codes.

This happens more often than expected — phones that are in airplane mode for a long time, phones with manual time settings, or phones in regions with incorrect timezone configurations can all drift.

**Diagnostic step:** Ask the user to check their phone's time settings. Go to **Settings → General → Date & Time → Set Automatically** (iOS) or **Settings → System → Date & Time → Use network-provided time** (Android). Then immediately try the OTP again. If it works, clock drift was the cause. On the server side, check NTP sync status: `timedatectl status` — look for `System clock synchronized: yes`.

---

### Root Cause 2 — Wrong Secret / Re-enrollment Needed

If a user sets up TOTP on a new phone without properly migrating their authenticator app, or if the server secret was regenerated (e.g. after a security incident), the secret on the phone and the secret on the server won't match. They'll generate completely different OTPs even with a perfectly synced clock.

This also happens when a user accidentally scans the wrong QR code, or scans it twice creating two entries that rotate at different offsets.

**Diagnostic step:** Check the server logs for the specific error. Most TOTP implementations log whether the failure is a `CLOCK_DRIFT` error (codes are off by one window) or an `INVALID_TOKEN` error (codes don't match at all across multiple windows). If it's `INVALID_TOKEN` consistently, the secret is wrong and the user needs to re-enroll — delete the entry from their authenticator app and scan a fresh QR code.

---

### Root Cause 3 — OTP Reuse Within the Same Window

RFC 6238 recommends that servers reject a TOTP code that has already been used within the same 30-second window, even if it's technically still valid. This prevents replay attacks. Some implementations enforce this strictly.

If a user tries to log in, fails for another reason (wrong password, network issue), then immediately tries again with the same OTP within the same 30-second window, the server may reject the second attempt even though the OTP itself is correct — because it was already "seen".

**Diagnostic step:** Ask the user to wait for the OTP to rotate (watch the timer in the authenticator app) and then try with the fresh new code. If it works, the previous code was being replayed. Check the server's `allow_reuse` or `valid_window` configuration — some implementations allow a window of ±1 period (90 seconds total) to account for clock drift, which also means a recently used code could be accepted or rejected depending on the config.

---

## How TOTP Works — The Algorithm

Understanding this makes every TOTP support issue easier to diagnose:

```
OTP = HOTP(secret, T)
where T = floor(current_unix_time / 30)
```

Both the phone and the server compute this independently. They only need:
1. The **shared secret** (embedded in the QR code at enrollment)
2. The **current time** (from their respective clocks)

No network call is made when generating an OTP. This is why clock sync is so critical — it's the only external dependency.

---

## TOTP vs SMS OTP — Why TOTP is More Secure

| Factor | TOTP | SMS OTP |
|---|---|---|
| **Network required** | No — works offline | Yes — requires mobile signal |
| **Expiry** | 30 seconds | Varies, often minutes |
| **Intercept risk** | Very low — no transmission | High — SS7 attacks, SIM swapping |
| **Phishing resistance** | Moderate | Low |
| **Clock dependency** | Yes — must be synced | No |
| **Cost** | Free | Per-SMS cost for provider |

TOTP's main vulnerability is that the code can still be phished in real time — an attacker can trick a user into entering their OTP on a fake site and immediately relay it. This is why hardware keys (FIDO2/WebAuthn) are considered more secure than TOTP for high-value accounts.

---

## What I Learned

**TOTP has no network component after setup.** The QR code is scanned once, the secret is stored on the phone, and from that point both sides compute OTPs independently. This means TOTP works in airplane mode — but it also means clock sync is the single most critical dependency.

**The 30-second window is a design tradeoff.** Shorter windows are more secure but less usable. Most implementations also allow ±1 window (90 seconds total) to handle minor clock drift without breaking the UX.

**Google Authenticator blocking screenshots is itself a security feature.** It prevents malware or screen recording apps from capturing OTP codes. This is why Authy (which allows screenshots but requires account backup) is sometimes preferred in enterprise environments — it's a tradeoff between security and recoverability.

**TOTP failure diagnosis is straightforward once you understand the algorithm.** If both sides have the same secret and the same time, they will always generate the same OTP. If the OTP is wrong, either the secret is wrong or the clock is wrong — it's always one of those two.

---

## Final Checklist

| Criteria | Status |
|---|---|
| pyotp, qrcode, pillow installed | ✅ |
| TOTP secret generated | ✅ `2EVNOLNQDJCZK5O6AIUYIBBUPGBGSL43` |
| QR code created and saved | ✅ `/tmp/totp_qr.png` |
| QR code scanned with Google Authenticator | ✅ |
| Terminal OTP matches Authy OTP (`201633`) | ✅ |
| Wrong OTP rejected (`000000` → False) | ✅ |
| OTP rotation demonstrated (233439 → 602053) | ✅ |
| 3 TOTP failure root causes written | ✅ |
| 1 diagnostic step per root cause | ✅ |

---

## Part 5 — VM Clock Sync Verification

The lab guide specifically asks to run the OTP check on the VM and compare it with the phone — this simulates what a support engineer does when a user reports TOTP not working.

I SSH'd into the OCI VM and ran:

```bash
python3 -c "import pyotp; s='2EVNOLNQDJCZK5O6AIUYIBBUPGBGSL43'; t=pyotp.TOTP(s); print(f'VM OTP: {t.now()}')"
```

**VM terminal showing OTP `740428`:**
![VM OTP](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-7-image.png)

**Google Authenticator on phone showing `740428` for InstaSafe Lab:**
![Phone OTP match](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-2-3-6-image.png)

Both the OCI VM and the phone independently generated `740428` at the same moment. This confirms:
- The VM's system clock is properly synced (NTP is working)
- The phone clock is in sync
- The shared secret is identical on both sides

If these had shown different codes, it would mean clock drift — the most common cause of TOTP failures in enterprise environments. The diagnostic step would be to check NTP sync on the server (`timedatectl status`) and enable automatic time on the phone.
