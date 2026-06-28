
# Lab 5.1 — Kill the Gateway: Simulate and Respond to a P1

## My Setup
- **VM:** vpn-server — OCI ap-hyderabad-1 — Public IP `140.245.218.252`
- **Monitoring:** Uptime Kuma (port 3001), Prometheus + Grafana (ports 9090/3000)
- **Gateway:** Nginx installed via apt — monitored via HTTP on port 80
- **Ticket Repo:** github.com/kssreelakshmi/cogs-support-lab-tickets
- **Incident ID:** INC-2026-0628-001
- **Total Downtime:** 29 minutes (23:21 — 23:50 IST)

---

## Pre-Lab Setup

Before starting the timer I made sure all services were running:

```bash
docker ps          # Uptime Kuma, Prometheus, Grafana, Node Exporter all up
sudo systemctl status nginx  # active (running)
```

One issue I ran into during setup — the Uptime Kuma monitor for Lab Nginx Server was pointed at the wrong IP (`140.245.253.205` — the vpn-client) instead of the vpn-server. Fixed it by editing the monitor URL to `http://140.245.218.252:80`. Also had to open port 80 in both iptables and the OCI Security List — the same double firewall issue from previous labs.

Once all 3 monitors showed green and all 4 tabs were open, I triggered the incident.

---

## The Incident — Step by Step

### T+0 — Detection (23:21 IST)

```bash
sudo systemctl stop nginx
```

Uptime Kuma detected the outage within 60 seconds and fired a red alert at **23:21:32 IST** with the message `connect ECONNREFUSED 140.245.218.252:80`. The toast notification appeared in the bottom right corner of the dashboard.

**Uptime Kuma red alert — 23:21 IST:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-1-1-image.png" alt="Uptime Kuma red alert showing Nginx down at 23:21 IST" width="800"/>
</p>

### T+2 — Ticket Created (23:23 IST)

Created GitHub Issue #4 immediately after the alert:

```
Title: [P1] Lab Nginx Gateway Unreachable — All Users Offline
Labels: P1-Critical, product:ztna, status:in-progress
Milestone: Sprint 1 — Lab Tickets
Project: Support Desk — Sprint 1
```

The ticket timestamp on GitHub proves creation within 2 minutes of detection.

📄 **P1 Ticket:** [Issue #4](https://github.com/kssreelakshmi/cogs-support-lab-tickets/issues/4)

### T+3 — Customer Acknowledgement (23:24 IST)

Posted the acknowledgement as a comment on the ticket within 3 minutes of detection:

```
Subject: Service Disruption — Nginx Gateway Unreachable

Dear Customer,
We are aware of an ongoing issue affecting the Nginx gateway. Our team 
has been alerted and investigation is underway.

Current Status: Service is down — all users affected
Detected At: 2026-06-28 23:21 IST
Impact: All users unable to connect through the gateway
Next Update: Within 15 minutes
```

### T+8 — Investigation

Checked two sources:

**systemctl status nginx:**
```
Active: inactive (dead) since Sun 2026-06-28 17:48:32 UTC
```
Confirmed the Nginx process was not running. Clean stop — no crash dump.

**Prometheus CPU query:**
```
100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100)
```
CPU was at 3-4% at the time of failure — no spike. This ruled out resource exhaustion as the cause. The failure was a clean process stop, not an OOM kill or CPU overload.

Added internal investigation note to ticket documenting both findings.

### T+15 — 15-Minute Update

Posted update to ticket:
```
15-minute update — investigation ongoing
Root cause identified: Nginx process is not running. 
Restarting service now. Expected resolution in 2 minutes.
```

### T+20 — Resolution (18:20 UTC)

```bash
sudo systemctl start nginx
```

Output confirmed:
```
Active: active (running) since Sun 2026-06-28 18:20:05 UTC
```

### T+22 — Recovery Confirmed (23:50 IST)

Uptime Kuma flipped green at **23:50:33 IST** with `200 - OK`. Total downtime: 29 minutes.

**Uptime Kuma green recovery — 23:50 IST:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-5-1-2-image.png" alt="Uptime Kuma showing all 3 monitors green after recovery" width="800"/>
</p>

### T+25 — Resolution Notice

Posted customer resolution notice to ticket:

```
Subject: Service Restored — Nginx Gateway Back Online

Incident ID: INC-2026-0628-001
Resolved At: 2026-06-28 18:20 UTC
Duration: 31 minutes
Root Cause: Nginx process stopped — no resource exhaustion detected
Action Taken: Nginx service restarted successfully
Prevention: Auto-restart will be configured
```

### T+30 — Ticket Closed

Changed label to `status:resolved`, project status to `Closed`, and closed the issue as Completed.

### T+35 — PIR Committed

📄 **PIR:** [PIR-2026-06-28-Nginx-Gateway-Failure.md](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/pirs/PIR-2026-06-28-Nginx-Gateway-Failure.md)

### T+45 — PACE Handover Committed

📄 **PACE Handover:** [handover-2026-06-28-1851.md](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/handovers/handover-2026-06-28-1851.md)

---

## What the Tools Showed

**Uptime Kuma** was the first to fire — it caught the outage within one check cycle (60 seconds) and gave an exact timestamp. This is what triggered the P1 response. Without it, the outage would only be detected when a customer called in.

**Prometheus** was used to rule out resource exhaustion. CPU was 3-4% at the time of failure — no spike before or after the stop command. This told me immediately it wasn't an OOM kill or runaway process — just a stopped service. That narrowed the fix to a simple restart without needing to investigate further.

**systemctl status** gave the exact stop time (17:48:32 UTC) which let me calculate the true incident start time — not when Uptime Kuma detected it, but when it actually went down.

---

## Issues I Ran Into

**Uptime Kuma monitor was pointed at the wrong IP.** The Lab Nginx Server monitor was checking `140.245.253.205` (vpn-client) instead of `140.245.218.252` (vpn-server). Fixed before starting the lab by editing the monitor URL.

**Port 80 was blocked after VM restart.** iptables rules don't persist across reboots by default. Had to re-add the rule with `sudo iptables -I INPUT -p tcp --dport 80 -j ACCEPT` and also open port 80 in the OCI Security List. Same double firewall issue from Labs 1-3.

**Git push rejected twice** due to remote changes. Fixed both times with `git pull --rebase origin main && git push`.

---

## What I Learned

**Monitoring tools tell you different things.** Uptime Kuma tells you something is down. Prometheus tells you why the system was behaving before it went down. systemctl tells you exactly what happened to the process. You need all three to do a proper P1 investigation — checking only one gives you an incomplete picture.

**The acknowledgement timer is real pressure.** Getting a customer acknowledgement out within 3 minutes while also creating a ticket and starting an investigation is genuinely difficult. Having the template ready before the incident starts is the only way to hit that window. In a real environment, some teams have auto-acknowledgement that fires the moment a P1 ticket is created.

**Auto-restart is a basic reliability requirement.** The entire 29-minute outage could have been reduced to under 60 seconds if Nginx had `Restart=always` configured in its systemd service. That's a one-line fix that prevents a class of outages entirely.

**Git timestamps are honest.** The commit history shows exactly when the PIR and handover were written. You can't backdate a git commit without rewriting history — which makes it a reliable audit trail for time-sensitive process requirements like "PIR within 24 hours."

---

## Full Incident Timeline

| Time | Event |
|------|-------|
| 17:48:32 UTC | Nginx process stopped (sudo systemctl stop nginx) |
| 23:21:32 IST | Uptime Kuma alert fired — ECONNREFUSED port 80 |
| 23:23 IST | P1 ticket #4 created in GitHub Issues |
| 23:24 IST | Customer acknowledgement posted |
| 23:29 IST | systemctl confirmed nginx inactive since 17:48 UTC |
| 23:30 IST | Prometheus CPU checked — no spike confirmed |
| 23:31 IST | Internal investigation note added to ticket |
| 23:33 IST | 15-minute update posted |
| 18:20:05 UTC | Nginx restarted — active (running) |
| 23:50:33 IST | Uptime Kuma confirmed green — 200 OK |
| 23:51 IST | Resolution notice sent |
| 23:52 IST | Ticket closed as resolved |

---

## Final Checklist

| Criteria | Status |
|---|---|
| Uptime Kuma red alert detected and screenshot taken | ✅ 23:21 IST |
| P1 ticket created within 2 minutes | ✅ 23:23 IST |
| Customer acknowledgement within 3 minutes | ✅ 23:24 IST |
| Prometheus investigation completed | ✅ No CPU spike |
| Internal investigation note added | ✅ |
| 15-minute update posted | ✅ |
| Nginx restarted | ✅ 18:20 UTC |
| Uptime Kuma green recovery screenshot taken | ✅ 23:50 IST |
| Resolution notice sent with correct structure | ✅ |
| Ticket closed as resolved | ✅ |
| PIR committed to GitHub | ✅ |
| PACE handover committed to GitHub | ✅ |
