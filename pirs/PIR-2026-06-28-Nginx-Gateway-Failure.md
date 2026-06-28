# Post-Incident Report (PIR)

**Incident ID:** INC-2026-0628-001
**Date:** 2026-06-28
**Severity:** P1
**Status:** Resolved

## Incident Summary

On 2026-06-28 at 23:21 IST, Uptime Kuma detected that the Nginx gateway on vpn-server (140.245.218.252) was unreachable. All users were unable to connect through the gateway. The alert fired as ECONNREFUSED on port 80. Investigation confirmed the Nginx process had stopped. No CPU spike or memory exhaustion was detected in Prometheus — it was a clean process failure. The service was restarted at 18:20 UTC and all users were restored. Total downtime was 31 minutes.

## Timeline

| Time (IST/UTC) | Event |
|----------------|-------|
| 17:48 UTC | Nginx process stopped — incident begins |
| 23:21 IST | Uptime Kuma detected ECONNREFUSED on port 80 — alert fired |
| 23:23 IST | P1 ticket #4 created in GitHub Issues |
| 23:24 IST | Customer acknowledgement sent |
| 23:29 IST | systemctl status confirmed nginx inactive (dead) since 17:48 UTC |
| 23:30 IST | Prometheus CPU checked — no spike, confirmed process failure not resource issue |
| 23:31 IST | Internal investigation note added to ticket |
| 23:33 IST | 15-minute update posted to ticket |
| 18:20 UTC | Nginx restarted — service active (running) |
| 18:21 UTC | Uptime Kuma confirmed green — service restored |
| 23:51 IST | Resolution notice sent to customer |
| 23:52 IST | Ticket closed as resolved |

## Root Cause

The Nginx process stopped unexpectedly on vpn-server. No resource exhaustion (CPU, memory, disk) was detected in Prometheus at the time of failure — the process simply exited cleanly. This was a simulated gateway crash (sudo systemctl stop nginx) but in a real environment this could be caused by a failed config reload, OOM kill, or manual error. There was no auto-restart configured for Nginx, so once it stopped it stayed down until manually restarted.

## Impact Assessment

- **Users affected:** All users (100%)
- **Duration:** 31 minutes (17:48 — 18:20 UTC)
- **Business impact:** Complete gateway outage — no users could connect during the incident window. SSH and VM reachability were unaffected. No data loss occurred.

## Resolution Steps

1. Uptime Kuma alert detected ECONNREFUSED on port 80 at 23:21 IST.
2. P1 ticket created and customer acknowledged within 3 minutes.
3. systemctl status nginx confirmed process was inactive since 17:48 UTC.
4. Prometheus CPU graph checked — no spike, ruled out resource exhaustion.
5. Nginx restarted with sudo systemctl start nginx.
6. Uptime Kuma confirmed green — service restored at 18:20 UTC.
7. Resolution notice sent to customer and ticket closed.

## Prevention Actions

- [ ] Configure Nginx auto-restart on failure — add `Restart=always` to nginx.service — Owner: Infrastructure Team — Due: 2026-07-05
- [ ] Set up Uptime Kuma alert notification (email/webhook) so on-call is paged automatically — Owner: Support Team — Due: 2026-07-05

## Open Items

- [ ] Write a quick runbook for Nginx process failure — what to check and how to restart — Owner: Support Team — Due: 2026-07-07

