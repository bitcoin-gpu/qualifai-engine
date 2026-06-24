# Lender New Lead — Workflow (Audit, Fixes & Appointment Flow)

The Facebook new-lead intake workflow. Instant speed-to-lead response, reply-based routing, and
appointment booking. Current sender identity in GHL: **Builder Funding / Hayden Levi** (investor
fix-&-flip / ground-up / SSR loans).

> GHL workflow name: **Lender New Lead**. Pairs with the separate **Reactivation Workflow**
> (`reactivation-workflow.md`) and the **Appointment Confirmation** workflow (below).

---

## What it does
FB lead form submitted → internal notify + create opportunity (New Lead) → instant SMS + email →
1-hour reply wait → route by reply (Hot / Planning-Nurture / No-reply) → Hot path qualifies → sends
application link → tracks the `application submitted` tag → notifies team. AI intent (`schedule-yes`)
classifies some replies.

**Consent:** leads opt in on the FB ad form. **Opt-out:** "STOP" is in the texts and GHL/carrier
honors it natively (sets DND-SMS).

---

## Critical fixes — DO IN THIS ORDER

### Part 1 — verify the two "handled" items
1. Confirm the FB form consent line covers **automated/recurring texts** (TCPA wording).
2. Test: text **STOP** from a test phone → confirm contact flips to **DND (SMS)** and stops.
3. Add proof-of-consent: first action on the trigger → **set field `SMS Consent = Yes`** (documents it +
   gives every SMS step something to gate on).

### Part 2 — critical safety (do before scaling)
4. **Stop on Appointment Booked** — add a "Customer Booked Appointment" trigger → **Remove from
   Workflow** + update stage to "Appointment Booked" + tag `appointment-booked`. (Booking = silent stop
   here; the Appointment Confirmation workflow sends the messaging.)
5. **Stop on Response = ON** (Settings) — a reply halts all pending sends.
6. **Kill duplicate enrollment** — Settings → **Allow Re-Entry = OFF**; on Create Opportunity → **Allow
   Duplicate = No** (update existing).

### Part 3 — close every dead-end (the #1 rule)
7. Every branch that ends with no action (esp. "Need Help → else/timeout") → add **tag + stage**
   (Nurture or Dead) + optional nurture enrollment. **No branch ends without a tag + a stage.**
8. Fix stuck "Application Sent" — every application-timeout path resolves to **Nurture** (don't park
   opportunities in "Application Sent" forever).

### Part 4 — biggest lead-saver: routing
9. Replace **exact-phrase If/Else** ("Working on a deal," "Yes I need help," "Circle back," "30,90")
   with **AI intent matching** (same `matches_intent` as `schedule-yes`). Intents:
   `working-on-deal`, `planning-ahead`, `not-interested`, `needs-help`. Real replies route correctly
   instead of dropping to Else.

### Part 5 — maintainability
10. **De-duplicate the Application block** — build ONE "Application Tracking" workflow triggered by tag
    `app-link-sent`; replace every repeated block with "Add tag `app-link-sent`."
11. Fix the contradiction: internal "**don't contact yet**" note fires while automation **does**
    contact. Reword to "auto-outreach started — review for personal follow-up," or remove.

---

## Improved blueprint (separate workflows)
- **Lender New Lead (intake):** FB form trigger → dedupe + create opp + internal notify → instant
  email + consent-gated instant SMS → 1hr reply wait → tag by **AI intent**. Stop-on-Response +
  Stop-on-Booking ON, Re-Entry OFF.
- **Qualification → Application** (triggered by `hot` tag): qualify Qs → application link → tag
  `app-link-sent`.
- **Application Tracking** (triggered by `app-link-sent`): the single Application-Sent → submitted →
  notify / need-help block.
- **Nurture** (triggered by `nurture` tag): long-term recurring.
- **Appointment Confirmation** (below).

---

## Appointment Confirmation & Reminders (separate workflow)

**Trigger:** Appointment Booked. **Settings:** Stop on Response = OFF (reminders are transactional).

**1 — Instant confirmation (SMS + Email)**
SMS:
```
You're all set, {{contact.first_name}}! Your call with Hayden at Builder Funding is booked for
{{appointment.start_time}}. We'll go over your deal and how fast we can fund it. Reply here if
anything changes.
```
Email subject: `Your Builder Funding call is confirmed — {{appointment.start_time}}`
```
Hi {{contact.first_name}},
You're confirmed for a call with Hayden Levi at Builder Funding on {{appointment.start_time}}.
We'll cover your deal, the numbers, and how quickly we can get you funded. To reschedule, just reply
or use the link in your calendar invite.
Talk soon, Hayden — Builder Funding
```

**2 — 24 hours before → SMS**
```
Hey {{contact.first_name}}, quick reminder — your call with Hayden at Builder Funding is tomorrow at
{{appointment.start_time}}. Have your deal details handy (purchase price, rehab budget, market).
Reply if you need to move it.
```

**3 — 1 hour before → SMS**
```
See you soon, {{contact.first_name}} — your call with Hayden is in about an hour
({{appointment.start_time}}). Talk then!
```

**4 — No-show (wait ~20 min after start → if status = No-Show)**
SMS to lead:
```
Hey {{contact.first_name}}, looks like we missed each other — no worries. Grab another time here:
{{calendar_link}}. Deals move fast and I'd hate for funding to hold yours up.
```
Internal: `⚠️ No-show: {{contact.name}} ({{contact.phone}}) missed their Builder Funding call.`
Then → update stage (Hot Lead / Nurture) so they're not stuck in "Appointment Booked."

**5 — Reschedule:** re-fire confirmation on the updated appointment.

---

## Compliance summary
- **SMS:** opt-in at FB form + `SMS Consent = Yes` field gate before each SMS; STOP honored natively.
- **Appt-booked exit:** Stop on Appointment Booked ON (intake/nurture).
- **Dead-ends:** none — every path ends in a tag + stage.
- **A2P 10DLC:** registered.

---

## Test checklist
| Test | Pass = |
|---|---|
| FB lead submits | Opp created (no dupe), internal notify, instant email + (consent) SMS |
| No SMS consent | Zero SMS; email only |
| Reply "working on a deal"/similar | Routes Hot via **intent** (not exact phrase) |
| Vague reply ("idk maybe") | Routes via intent or → broker; **never stuck** |
| Reply STOP | DND set; all messaging stops |
| Books a call | Intake stops silently; Appointment Confirmation fires |
| No response | Completes cadence → Nurture (not dead-end) |
| App link sent, tag never comes | Need-help → resolves to a stage (not stuck in "Application Sent") |
| Application submitted tag | Stage → Submitted, team notified |
| Duplicate FB submit | Enrolls once, no second opportunity |
| "Not interested" / "circle back" | → Dead stage, clean |
