
# Lab 4.1 — GitHub as a Support Desk: Full Ticket Lifecycle

## My Setup
- **Platform:** GitHub (github.com/kssreelakshmi/cogs-support-lab-tickets)
- **Simulation Method:** GitHub Issues as ticketing system, GitHub Projects (Board view) as queue management
- **Milestone:** Sprint 1 — Lab Tickets
- **Tickets Created:** 3 (P2, P3, P4)
- **Labels Created:** 10 custom labels covering priority, status, and product

---

## Part 1 — Setting Up the Support Desk Infrastructure

I created a new repository `cogs-support-lab-tickets` and built the full support desk structure inside it before touching any tickets — labels first, then milestone, then project board. This is intentional: in a real support environment, the taxonomy (how you categorize tickets) should be defined before tickets start coming in, not retrofitted afterward.

### Labels

I created 10 custom labels across three categories:

**Priority labels** (4): P1-Critical, P2-High, P3-Medium, P4-Low — colour-coded from red (urgent) to green (low). These map directly to SLA tiers. A P1 in a real environment would have a 15-minute response SLA; a P4 might be 2 business days.

**Status labels** (3): status:in-progress, status:waiting-on-customer, status:resolved — these track where a ticket is in its lifecycle independently of the project board column. Using both (board column + label) gives redundancy: you can filter the issues list by label even without opening the board.

**Product labels** (3): product:ztna, product:mfa, product:sso — these let support leads filter tickets by product area and route them to the right L2 team without reading the ticket body.

**All 10 custom labels with colours:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-4-1-1-image.png" alt="GitHub label list showing all 10 custom labels" width="800"/>
</p>

### Project Board

I created a GitHub Project in Board view named **Support Desk — Sprint 1** with five columns:

`Backlog → In Progress → Waiting on Customer → Resolved → Closed`

This mirrors the ticket lifecycle from Section 5. Each column represents a state in the support workflow. A ticket moves left-to-right as it progresses — the only exception is Waiting on Customer, which can send a ticket back to In Progress if the customer responds.

One thing I noticed: GitHub Projects automation moved newly created issues to Backlog automatically. I had to manually drag Ticket #1 to In Progress after creation since it came in with an active investigation already underway.

---

## Part 2 — Creating the 3 Mock Tickets

### Ticket 1 — P2: MFA SMS OTP failure (25 users, finance dept)

This is the highest-priority ticket in the sprint — a bulk MFA failure affecting 25 users in a financial institution. P2 because it's multi-user and production, but not P1 because email OTP still works (a workaround exists).

```
Title: [P2] MFA SMS OTP not delivered — finance department, 25 users affected

Customer Segment: Mid-size financial institution (~500 users)
Reported At: 09:15 IST
Impact: 25 users in finance dept cannot receive SMS OTP. Email OTP works.
Environment: Production
Steps Tried by Customer: Verified phone numbers are correct. Issue started post-weekend.

Labels: P2-High, product:mfa, status:in-progress
Milestone: Sprint 1 — Lab Tickets
```

Immediately after creating the ticket, I added an internal investigation note as a comment. This is a real support practice — internal notes are visible to the support team but not the customer. In Zoho Desk, this would be a private note. In GitHub, it's just a comment (no access control), but the `[Internal Note — NOT for customer]` header makes the intent clear.

### Ticket 2 — P3: Single user locked out post-password change

A single-user lockout after an AD password change. P3 because it's one user and there's a clear suspected cause (AD sync delay). The ZTNA client caches credentials and may not have picked up the new password yet.

```
Title: [P3] User locked out after password change — AD sync delay suspected

Customer Segment: SME (~150 users)
Reported At: 10:30 IST
Impact: 1 user cannot log in after changing AD password. ZTNA client shows authentication failure.
Environment: Production
Steps Tried by Customer: User confirmed new password works on Windows login. Issue only on ZTNA client.
Suspected Cause: AD sync delay — ZTNA may still be validating against cached old credentials.

Labels: P3-Medium, product:ztna
```

### Ticket 3 — P4: How-to question on adding a second gateway

A guidance request, not a break-fix. P4 because it's non-urgent and doesn't affect current users — the customer wants to add redundancy, which is a configuration question. In a real environment, this would be routed to a solutions engineer or pointed at documentation.

```
Title: [P4] How to add a second gateway — customer requesting guidance

Customer Segment: Small business (~50 users)
Reported At: 14:00 IST
Request: Customer wants to add a second gateway for redundancy. Asking for step-by-step guidance.
Environment: Production
Notes: Not a break-fix — documentation/guidance request only.

Labels: P4-Low, product:ztna
```

**All 3 tickets created in the Issues list:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-4-1-2-image.png" alt="Issues list showing all 3 tickets with labels and milestone" width="800"/>
</p>

---

## Part 3 — Full Ticket Lifecycle on Ticket #1

### Stage 1: Creation + Internal Investigation Note

Ticket created with P2-High, product:mfa, status:in-progress labels and added to the Sprint 1 milestone. Immediately added an internal note after checking the Kaleyra dashboard:

```
[Internal Note — NOT for customer]
Checked Kaleyra dashboard: bulk REJECTED status codes for all 25 numbers.
Possible DND registration or sender ID blacklisting at carrier level.
Escalating to L2 for Kaleyra sender ID investigation.
```

This note documents what L1 found during initial triage — it becomes the starting point for L2 when they pick up the ticket. Without it, L2 would have to repeat the Kaleyra check.

### Stage 2: Escalation to L2

I added an escalation comment using the standard template from Section 5. The template ensures L2 gets everything they need in a structured format without having to read the full ticket thread:

```
🔺 ESCALATION — P2
Ticket: #1
Customer Segment: Mid-size financial institution
Issue: Bulk SMS OTP failure — 25 users, finance dept
Impact: 25 users cannot use MFA since this morning
Steps Tried: Kaleyra dashboard checked — REJECTED codes across all 25 numbers
Current Status: Carrier-level block suspected
Needs From L2: Verify Kaleyra sender ID status and check carrier whitelist
SLA Deadline: 45 minutes remaining
```

After posting this, I changed the project board status to **Waiting on Customer** and swapped the label from `status:in-progress` to `status:waiting-on-customer`.

### Stage 3: Resolution

After L2 investigation, the root cause was identified. I posted the resolution summary:

```
Resolution Summary
Root Cause: Kaleyra sender ID was flagged by TRAI for DND violation by another
sender sharing the route.
Action Taken: Switched to backup sender ID. Verified delivery for 3 test numbers.
Prevention: Added sender ID monitoring alert in Kaleyra dashboard.
Customer confirmed: All 25 users successfully receiving OTP as of 11:45 IST.
```

Changed label to `status:resolved`, moved board status to Resolved.

### Stage 4: Closure

Added a final customer sign-off comment: *"Customer confirmed sign-off. Closing ticket."*

Closed the issue as Completed. Board status moved to **Closed**. The purple Closed badge confirmed the issue was properly shut.

**Ticket #1 full comment thread (creation → internal note → escalation → resolution → closure):**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-4-1-3-image.png" alt="Ticket #1 full comment thread showing complete lifecycle" width="800"/>
</p>

**Project board — final state with tickets in different columns:**

<p align="center">
  <img src="https://github.com/kssreelakshmi/cogs-lab-portfolio/blob/main/screenshots/lab-4-1-4-image.png" alt="Project board showing Ticket 1 in Closed, Tickets 2 and 3 in Backlog" width="800"/>
</p>

---

## What I Learned About Ticket Hygiene

**Internal notes are underrated.** The escalation template is the visible, formal handoff — but the internal investigation note is what makes L2's job faster. If L1 documents what they checked, L2 doesn't repeat it. In a busy support queue where a P2 has a 45-minute SLA, that duplication is the difference between meeting SLA and missing it.

**Labels and board columns serve different purposes.** The board column shows workflow state (where is this ticket right now?). The label shows metadata that persists through transitions (what priority is it? what product?). You need both — a ticket in the Closed column still needs to carry `P2-High` and `product:mfa` so you can run reports later on what types of issues are getting escalated.

**GitHub Issues is surprisingly capable as a support desk.** It doesn't have SLA timers, customer-facing portals, or CSAT surveys — things Zoho Desk has out of the box. But the combination of Issues + Labels + Milestones + Projects covers the core workflow: create, triage, escalate, resolve, close. For an internal team or small support operation, this is enough.

**Ticket structure matters more than ticket volume.** Writing structured ticket bodies (Customer Segment, Impact, Steps Tried, Environment) feels like overhead on a mock lab. In production, a poorly structured ticket means the next person in the queue has to call the customer back to gather information they should have had from the start. Structure at creation time saves time at every subsequent stage.

---

## Issues I Ran Into

**GitHub Project automation moved Ticket #1 to Backlog on creation.** Even though I set the project during ticket creation and intended it to go straight to In Progress, the board automation defaulted new items to Backlog. I had to manually drag it to In Progress after creation. In a real environment, you'd configure automation rules to handle this based on labels.

**Ticket #3 initially landed in "No Status" column.** GitHub Projects showed a "No Status" column that I hadn't created — this happens when a ticket is added to the project without a status assigned. I dragged it to Backlog manually. The fix would be to set a default status in the project's automation settings.

**The "Close with comment" button closes immediately.** I accidentally started to click "Close with comment" before adding the resolution — caught it in time. In Zoho Desk there's a distinct workflow for this. In GitHub you have to be careful about the order: add resolution comment first, then close separately.

---

## Final Checklist

| Criteria | Status |
|---|---|
| Repository created: cogs-support-lab-tickets | ✅ |
| 10 custom labels created with correct colours | ✅ |
| Milestone created: Sprint 1 — Lab Tickets | ✅ |
| Project board created with 5 columns | ✅ |
| Ticket #1 (P2 MFA) created with structured body | ✅ |
| Ticket #2 (P3 Lockout) created with structured body | ✅ |
| Ticket #3 (P4 How-to) created with structured body | ✅ |
| Internal investigation note added to Ticket #1 | ✅ |
| Escalation comment added to Ticket #1 | ✅ |
| Resolution summary added to Ticket #1 | ✅ |
| Ticket #1 closed as Completed | ✅ |
| All tickets moved through board columns | ✅ |
