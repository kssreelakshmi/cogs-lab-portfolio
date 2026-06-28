# Lab 4.2 — PACE Handover + PIR

## My Setup
- **Repository:** github.com/kssreelakshmi/cogs-lab-portfolio
- **Folders created:** `handovers/` and `pirs/`
- **Files committed:** handover-2026-04-08-1800.md, PIR-2026-04-08-MFA-Bulk-Failure.md
- **Method:** Git commits as timestamped process records

---

## Part 1 — PACE Shift Handover

The PACE handover is a structured way to brief the incoming engineer at shift change. The idea is simple — when you're leaving, the person taking over shouldn't have to read through every ticket from scratch. PACE gives them exactly what they need in a fixed format: what's urgent, what's pending, what the context is, and what to expect next.

I wrote the handover for an imaginary shift ending at 18:00 IST on 2026-04-08, which was the same day as the MFA bulk OTP incident (Ticket #1001). This made it realistic — the P2 ticket was still open at shift change, so it needed to be handed over with SLA context.

### The 4 PACE Sections

**Priority Tickets** — lists only the tickets that need immediate attention with SLA deadlines. Ticket #1001 had an 18:30 deadline which meant the incoming engineer had 30 minutes from handover to get an update to the customer.

**Actions Pending** — what has already been set in motion that the incoming engineer needs to follow up on. L2 had been paged for #1001 and a follow-up email had been sent for #1002. These aren't things to start — they're things to watch and respond to.

**Context** — the background on each ticket so the incoming engineer understands what happened before they arrived. Without this section, they'd have to read the full ticket thread to understand where things stand.

**Expectations** — what is supposed to happen next and when. This is what the incoming engineer holds other people accountable to — L2 is expected to resolve by 18:30, customer is expected to confirm by 19:00.

The git commit timestamp on this file is the evidence that the handover happened at a specific time — same way a real support system would log when a handover note was written.

📄 **View PACE Handover file:** [handover-2026-04-08-1800.md](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/handovers/handover-2026-04-08-1800.md)

---

## Part 2 — Post-Incident Report (PIR)

The PIR is written after an incident is resolved. It's not about blame — it's about making sure the same incident doesn't happen again and that anyone who picks up a similar ticket in the future has a reference to work from.

I wrote the PIR for INC-2026-0408-001, the MFA SMS OTP bulk failure from the same day.

### The 7 Sections

**Incident Summary** — one paragraph covering what happened, when it was detected, and how it was resolved. Enough for a manager or team lead to understand the incident without reading the rest of the document.

**Timeline** — 9 entries from 09:15 to 11:45 IST. Each entry is a specific action taken or finding made. The timeline is the most useful part of a PIR for future reference — if this happens again, the timeline tells you exactly what to check and in what order.

**Root Cause** — the actual technical reason, not just "OTP failed." The shared Kaleyra sender ID was blacklisted by TRAI because another business on the same route violated DND rules. Our messages got blocked even though we didn't do anything wrong. No alert fired because sender ID monitoring wasn't configured.

**Impact Assessment** — 25 users, 2.5 hours, finance department. Email OTP worked as a workaround but had to be communicated manually which caused delays. No security compromise.

**Resolution Steps** — 6 steps in order, specific enough that someone else could follow them if this happened again. This is essentially a mini-runbook for this exact failure mode.

**Prevention Actions** — 3 actions with owners and due dates. The key ones: set up sender ID monitoring in Kaleyra so we get alerted before customers report it, and add the backup sender ID switch to the L1 runbook so this doesn't need L2 escalation next time.

**Open Items** — writing a quick guide on what to check in Kaleyra when SMS OTP fails in bulk and how to switch sender IDs.

📄 **View PIR file:** [PIR-2026-04-08-MFA-Bulk-Failure.md](https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/pirs/PIR-2026-04-08-MFA-Bulk-Failure.md)

---

## Why Git Works as a Process Record

The lab uses Git commits instead of a ticketing system for a specific reason — every commit has a timestamp that can't be faked. When I committed the handover file, GitHub recorded exactly when that happened. In a real support environment, this matters:

- Did the handover happen before or after the SLA deadline?
- Was the PIR written within 24 hours of the incident closing?
- Who made changes to the document and when?

Git's commit history answers all of these questions automatically. You don't need a separate audit trail — the repo is the audit trail.

This is also why commit messages matter. `Add PACE handover 2026-04-08 18:00` is immediately readable in the commit log. A message like `update file` tells you nothing about what changed or why.

---

## What I Learned

**PACE stops information from falling through the gaps at shift change.** Before this lab I thought a handover was just telling the next person what's open. The structure matters — without the Expectations section, the incoming engineer doesn't know what commitments have already been made to the customer. Without the Context section, they're starting from scratch on every ticket.

**A PIR is only useful if the root cause is specific.** Writing "OTP delivery failed" as the root cause helps nobody. Writing "shared Kaleyra sender ID blacklisted by TRAI due to co-tenant DND violation, no monitoring alert configured" gives the next engineer something to actually act on.

**Prevention actions need owners and dates or they don't happen.** A checklist of things to do with no owner assigned is just a wishlist. The three prevention actions in this PIR each have a team and a deadline — that's what turns a PIR from a document into actual change.

---

## Final Checklist

| Criteria | Status |
|---|---|
| handovers/ folder created in cogs-lab-portfolio | ✅ |
| PACE handover file committed with timestamp | ✅ |
| All 4 PACE sections filled (Priority, Actions, Context, Expectations) | ✅ |
| pirs/ folder created | ✅ |
| PIR file committed with timestamp | ✅ |
| All 7 PIR sections filled | ✅ |
| Timeline has 9 entries with real timestamps | ✅ |
| Root cause is specific and technical | ✅ |
| Prevention actions have owners and due dates | ✅ |
| Both commits have meaningful messages | ✅ |
