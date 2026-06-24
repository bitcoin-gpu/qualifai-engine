# QualifAI Engine — Phase 1 GHL Build Kit

Everything you paste into GHL tomorrow. The flow is **authored here**; tomorrow is **assembly**.
All copy is written in the **broker's voice** (the broker is the sender to their own old leads) and
follows CAN-SPAM (email) + TCPA/opt-out (SMS) rules from `phase1-reactivation-offer.md`.

> Reusable across clients — swap the merge fields ({{broker_name}}, {{broker_company}}, booking link).

---

## The flow at a glance

```
Trigger: contact added to "Reactivation" list (tag: engine-reactivation)
   │
   ├─ Email 1  (re-open)                          Day 0
   ├─ Wait 2 days  →  if no reply
   ├─ SMS 1  (consented contacts only)            Day 2
   ├─ Wait 2 days  →  if no reply
   ├─ Email 2  (rates/market changed)             Day 4
   ├─ Wait 3 days →  if no reply
   ├─ SMS 2  (consented only)                     Day 7
   ├─ Wait 3 days →  if no reply
   ├─ Email 3  (breakup / "close your file?")     Day 10
   │
   └─ REPLY at any point ─► stop sequence ─► AI Pre-Qual conversation
                                                   │
                          ┌── qualified ──► Book appointment ──► confirm + remind
                          └── not ready ──► Long-term nurture (monthly)
```

**Stop-on-reply and stop-on-booking are mandatory** (same as the LOS workflow).

---

## 0. GHL prerequisites (set these first, ~15 min)

- [ ] **Custom fields:** `loan_purpose` (buy/refi), `timeline`, `credit_band`, `consent_sms` (Y/N)
- [ ] **Tags:** `engine-reactivation`, `engine-replied`, `engine-qualified`, `engine-booked`, `engine-nurture`, `engine-dnc`
- [ ] **Pipeline:** "Engine — Reactivation" with stages: New → Re-engaged → Qualified → Appt Booked → Showed → Won/Lost
- [ ] **Calendar:** broker's booking calendar + link
- [ ] **A2P 10DLC** registered for the sending number (required for SMS)
- [ ] **From identity + reply routing** set to the broker
- [ ] **Suppression:** import broker's existing opt-outs / DNC first; tag `engine-dnc`

---

## 1. Reactivation EMAILS (CAN-SPAM compliant — plain text)

> Footer on **every** email: `{{broker_company}} · {{broker_address}} · Unsubscribe: {{unsubscribe}}`

### Email 1 — Re-open (Day 0)
**Subject:** still thinking about it?
```
Hi {{first_name}},

A while back you reached out about a home loan and we never got to finish the
conversation. No worries — life gets busy.

Quick question: are you still thinking about buying or refinancing, even just
down the road?

If so, reply and let me know roughly where you're at — I'll take a fresh look,
no pressure.

{{broker_name}}
{{broker_company}}
```

### Email 2 — Market/rates changed (Day 4)
**Subject:** worth a second look
```
Hi {{first_name}},

Things have moved since we last talked — rates, programs, and what people qualify
for have all shifted. A lot of folks who didn't fit a few months ago do now.

Takes me 5 minutes to see if anything's changed for you. Want me to check?

Just reply "yes" and I'll take it from there.

{{broker_name}}
{{broker_company}}
```

### Email 3 — Breakup (Day 10)
**Subject:** should I close your file?
```
Hi {{first_name}},

I haven't heard back, so I'll assume the timing isn't right — totally fine.

I'll go ahead and close out your file unless you tell me otherwise. If you'd
still like me to keep an eye out for you, just reply and I'll keep you posted
when something lines up.

Either way, wishing you the best.

{{broker_name}}
{{broker_company}}
```

---

## 2. Reactivation SMS (consented contacts only — `consent_sms = Y`)

> Branch the workflow: **only send SMS if `consent_sms = Y`.** Every SMS ends with opt-out.

### SMS 1 (Day 2)
```
Hi {{first_name}}, it's {{broker_name}} at {{broker_company}}. A while back you
looked into a home loan — still on your radar to buy or refi? Happy to take
another look. Reply STOP to opt out.
```

### SMS 2 (Day 7)
```
{{first_name}}, quick one — rates & programs have changed lately and a lot of
people qualify now who didn't before. Want me to recheck your options? Reply YES
or STOP to opt out.
```

---

## 3. AI Pre-Qualification conversation (fires on reply)

When a contact replies (`engine-replied`), the sequence stops and the pre-qual flow begins. Ask
**one question at a time** (conversational), capture answers into custom fields, then branch.

**Q1 — Purpose:** "Great to hear from you! Quick question — are you looking to **buy** or
**refinance**?" → save `loan_purpose`

**Q2 — Timeline:** "Got it. What's your timeline — **now, 1–3 months, 3–6 months, or just exploring**?"
→ save `timeline`

**Q3 — Pre-approval:** "Have you been pre-approved before, or would this be a fresh start?"

**Q4 — Property/location:** "What area are you looking in (or where's the property for a refi)?"

**Q5 — Credit ballpark:** "Last one — roughly where's your credit: **excellent, good, fair, or not
sure**?" → save `credit_band`

### Qualifying logic (branch)
**QUALIFIED** (tag `engine-qualified` → booking) if:
- `timeline` = now / 1–3 mo / 3–6 mo **AND**
- `credit_band` = excellent / good / fair (not "no, terrible/just curious") **AND**
- genuine intent expressed

**NOT READY** (tag `engine-nurture` → long-term nurture) if:
- `timeline` = "just exploring" / 6+ months, **or** credit clearly too low right now, **or** vague/no
  real intent

> Keep the bar realistic — a "fair credit, 3–6 months" lead is still a qualified appointment for a
> broker. Define the exact bar per client in the agreement.

---

## 4. Booking + confirmation + reminders (qualified leads)

### Booking message
```
You're in a good spot to look at this, {{first_name}}. Let's grab 15 minutes so
{{broker_name}} can walk you through your options — pick a time that works:
{{booking_link}}
```

### Confirmation (on booking)
```
You're set for {{appointment_time}} with {{broker_name}}. You'll get a reminder
beforehand. Talk soon!
```

### Reminders
- **24h before:** `Reminder: your call with {{broker_name}} is tomorrow at {{appointment_time}}. Reply if you need to reschedule.`
- **1h before:** `See you in an hour, {{first_name}} — {{broker_name}} will call/meet you at {{appointment_time}}.`

---

## 5. Long-term nurture (not-ready leads — monthly)

Goal: stay top-of-mind so they re-surface when ready. One value touch/month, no pressure.

- **Month 1:** "Hey {{first_name}} — no rush at all, just keeping your file warm. If your timeline
  changes, I'm one reply away. Here's a quick read on what's happening with rates: [tip]."
- **Month 2:** First-time-buyer / refi tip relevant to their `loan_purpose`.
- **Month 3:** "Still here when you're ready — anything changed on your end?" (soft re-qualify)
- Loop quarterly. Any reply → back into the pre-qual flow.

---

## 6. GHL workflow blueprint (assembly order for tomorrow)

1. **Create Workflow:** "Engine — Reactivation"
2. **Trigger:** Contact Tag Added = `engine-reactivation`
3. **Workflow settings:** enable **stop on reply** + **stop on appointment booked**
4. **Step 1:** Send Email 1
5. **Wait** 2 days
6. **If/Else:** `consent_sms = Y` → Send SMS 1 (else skip)
7. **Wait** 2 days
8. **Send Email 2**
9. **Wait** 3 days
10. **If/Else:** `consent_sms = Y` → Send SMS 2 (else skip)
11. **Wait** 3 days
12. **Send Email 3**
13. **Separate Workflow — "Engine — Pre-Qual":** Trigger = Customer Replied → run Q1–Q5, set fields,
    branch on qualifying logic → tag `engine-qualified` or `engine-nurture`
14. **Qualified path:** send booking message → on booking, confirmation + reminders + move pipeline to
    "Appt Booked"
15. **Nurture path:** add to "Engine — Nurture" workflow (monthly touches)

> Build the **reactivation** workflow + **pre-qual** workflow as two linked workflows (cleaner than one
> giant flow). Tags connect them.

---

## 7. Lead import CSV template (what the broker sends you)

Header the broker's dead-lead export should map to:

```
First Name,Last Name,Email,Phone,Loan Purpose,Original Inquiry Date,Source,SMS Consent
```

- **Required:** First Name, Email (for email-first reactivation)
- **Phone + SMS Consent:** required to text — if `SMS Consent` is blank/No, that contact is
  **email-only**
- On import: apply tag `engine-reactivation`, suppress anyone matching existing opt-outs

---

## Tomorrow's build checklist (in order)

- [ ] Set prerequisites (§0): custom fields, tags, pipeline, calendar, 10DLC
- [ ] Build "Engine — Reactivation" workflow (§6 steps 1–12)
- [ ] Build "Engine — Pre-Qual" workflow (§6 step 13–14)
- [ ] Build "Engine — Nurture" workflow (§5)
- [ ] Paste all email + SMS copy (§1, §2, §4, §5)
- [ ] Test with your own contact: import → tag → confirm Email 1 fires, reply → pre-qual runs → book → confirm
- [ ] Confirm stop-on-reply + stop-on-booking work
- [ ] Confirm SMS only fires for `consent_sms = Y`
- [ ] Ready for first broker's list
```
