# Post-Incident Report (PIR)

**Incident ID:** INC-2026-0408-001
**Date:** 2026-04-08
**Severity:** P2
**Status:** Resolved

## Incident Summary

On 2026-04-08, 25 users from the finance team of a financial institution couldn't receive SMS OTP for MFA login. Email OTP was still working so it wasn't a full MFA outage. I checked Kaleyra and saw REJECTED codes for all 25 numbers — that pointed to a carrier issue, not a user issue. Escalated to L2 who found that our SMS sender ID was blacklisted by TRAI because another sender sharing the same route had violated DND rules. We switched to a backup sender ID and all 25 users were able to login by 11:45 IST.

## Timeline

| Time (IST) | Event |
|------------|-------|
| 09:15 | Customer reported SMS OTP not working for 25 finance users |
| 09:18 | Ticket #1001 created — P2-High |
| 09:20 | Sent first response to customer |
| 09:25 | Checked phone numbers on file — all correct |
| 09:35 | Opened Kaleyra dashboard — saw REJECTED codes for all 25 numbers |
| 10:30 | Escalated to L2 — suspected carrier-level block |
| 11:30 | L2 confirmed: sender ID flagged by TRAI due to DND violation |
| 11:40 | L2 switched to backup sender ID — tested on 3 numbers, OTP delivered |
| 11:45 | All 25 users confirmed OTP working — ticket closed |

## Root Cause

Our Kaleyra SMS route used a shared sender ID. Another business on the same route got flagged by TRAI for sending messages to DND-registered numbers. TRAI blacklisted the sender ID, which blocked our OTP messages too even though we didn't do anything wrong. No alert fired on our end because we didn't have sender ID monitoring set up in Kaleyra.

## Impact Assessment

- **Users affected:** 25 (finance department)
- **Duration:** 2 hours 30 minutes (09:15 — 11:45 IST)
- **Business impact:** Finance users couldn't log in during morning hours. Email OTP worked as a workaround but users had to be told about it manually which caused delays.

## Resolution Steps

1. Checked Kaleyra dashboard — confirmed REJECTED codes across all 25 numbers, ruled out individual phone issues.
2. Escalated to L2 with investigation findings.
3. L2 contacted Kaleyra and found the shared sender ID was blacklisted by TRAI.
4. L2 switched to backup sender ID on a different route.
5. Tested OTP delivery on 3 numbers — all worked within 30 seconds.
6. Informed customer — all 25 users confirmed working by 11:45 IST.

## Prevention Actions

- [ ] Set up sender ID monitoring alert in Kaleyra — Owner: L2 Network Team — Due: 2026-04-15
- [ ] Add backup sender ID steps to L1 runbook so L1 can fix this without escalating — Owner: Support Lead — Due: 2026-04-22
- [ ] Request a dedicated sender ID from Kaleyra so we're not sharing with other businesses — Owner: Vendor Management — Due: 2026-05-01

## Open Items

- [ ] Write KB article on SMS OTP bulk failure due to sender ID blacklisting — Owner: L1 Team Lead — Due: 2026-04-15
- [ ] Check if other customers are on the same Kaleyra route — Owner: L2 Network Team — Due: 2026-04-18
