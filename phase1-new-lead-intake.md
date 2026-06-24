# QualifAI Engine — Phase 1 New-Lead Intake ("Speed-to-Lead")

Instant response + qualification for **new** leads (web forms, inbound calls/texts, referrals, and
later ad leads). Reuses the **same pre-qual / booking / nurture back half** as the reactivation flow —
only the trigger and opener change.

> **The governing rule — NO LEAD WITHOUT A NEXT ACTION.** Every lead is always in exactly **one**
> state, and every state has either a scheduled next touch or is a defined terminal state. A lead can
> only ever exit into: **Booked**, **Nurture** (with a re-trigger date), **DNC/Closed**, or **Human
> Handoff**. Nothing silently stops. Ever.

---

## Why speed-to-lead is non-negotiable
A new lead contacted in **under 5 minutes** is many times more likely to convert than one contacted an
hour later. Most brokers fail at exactly this. So the new-lead flow's first job is **instant** response
(seconds), then **persistent** multi-channel follow-up until a terminal state.

---

## Entry points (every way a new lead arrives)
| Source | Trigger in GHL | Notes |
|---|---|---|
| Web form / landing page | Form Submitted | most common |
| Missed inbound call | Call Status = missed | → instant "text-back" |
| Inbound text | Inbound SMS received | reply instantly |
| Referral / manual add | Tag `engine-newlead` added | broker drops them in |
| Email inquiry | Inbound email / form | email-first |
| Ad lead (Phase 3) | FB/IG Lead Form | same flow, later |

**All entry points converge** to one workflow: tag `engine-newlead` → instant response → pre-qual.

---

## The instant response (first touch — within seconds)

### Text (if phone + consent present)
```
Hi {{first_name}}, thanks for reaching out to {{broker_company}}! This is
{{broker_name}}'s team — happy to help you get moving right now. Quick question
so I point you the right way: are you looking to buy or refinance?
(Reply STOP to opt out.)
```

### Email (always)
**Subject:** got your request — let's get you moving
```
Hi {{first_name}},

Thanks for reaching out to {{broker_company}}! I'd love to help. To get you the
right options fast, just reply with one thing: are you looking to buy or refinance?

Talk in a sec,
{{broker_name}}
{{broker_company}}
{{broker_address}} · Unsubscribe: {{unsubscribe}}
```

→ On reply, the **shared pre-qual conversation** (Q1–Q5 from the build kit) takes over.

---

## THE SCENARIO MATRIX — every lead behavior, every outcome

Each row = a thing a real lead does. Each has a defined system action + resulting state. **None end in "nothing."**

| # | Lead does this… | System response | Lands in |
|---|---|---|---|
| 1 | Submits (any source) | Instant text + email (secs), start pre-qual | Awaiting reply (cadence armed) |
| 2 | Replies, engaged, answers pre-qual, **qualified** | Send booking link | **Booked** |
| 3 | Replies, engaged, **not ready / not qualified** | Warm message → long-term nurture | **Nurture** (monthly) |
| 4 | Replies with **a question only** ("what's the rate?") | AI answers briefly, then pivots back to next pre-qual question | Continue pre-qual |
| 5 | **No reply** to first touch | Enter escalating multi-channel cadence (below) | Cadence → Nurture |
| 6 | Replies **"STOP" / unsubscribe** | Opt-out, tag `engine-dnc`, suppress everywhere | **DNC** (terminal) |
| 7 | Replies **"not interested"** | One soft acknowledgment + offer to keep warm → light nurture | Nurture / Lost |
| 8 | **"Already working with a lender"** | Empathy + offer a no-pressure second opinion → nurture | Nurture (win-back) |
| 9 | **Wrong number / "who is this?"** | Clarify once; if wrong person → tag `bad-contact`, stop | Bad contact (terminal) |
| 10 | Starts pre-qual, then **goes silent mid-way** | Resume-prompt sequence (15min → 1hr → next day), picks up where they left off | Resume → cadence |
| 11 | **Books** an appointment | Confirmation + 24h + 1h reminders; pipeline → Appt Booked | **Booked** |
| 12 | **No-shows** the appointment | No-show cadence ("missed you — reschedule?") ×3 → nurture; **alert broker** | Reschedule → cadence |
| 13 | **Cancels** appointment | Immediate reschedule offer | Reschedule |
| 14 | **Reschedules** | New confirmation + reminders | Booked |
| 15 | **Asks to be called** (prefers phone) | Offer call-time booking + **alert broker** to call | Call task / Booked |
| 16 | Submits **after hours** | Instant auto-response engages now; human task queued for AM; AI keeps qualifying | Engaged / cadence |
| 17 | **Hot / high-intent** ("pre-approved, offer accepted, need it now") | **Immediate human-handoff alert** + fast-track booking | Handoff + Booked |
| 18 | Raises an **objection** (rate/credit/income worry) | AI addresses if simple; else route to broker; offer nurture | Nurture or Handoff |
| 19 | **Duplicate** (already in system) | Do NOT double-message; merge; resume existing state | Existing state |
| 20 | **Spam / bot / gibberish / fake** | Filter, tag `invalid`, no further outreach | Filtered (terminal) |
| 21 | **Re-engages from nurture** months later | Drop back into fresh pre-qual | Pre-qual |
| 22 | Qualified but **timeline far out** (6mo+) | Book a "future" touch with a **date-based re-trigger** at their timeline | Date-triggered nurture |
| 23 | Replies in **another language / unclear** | **Human-handoff alert** (don't guess) | Handoff |
| 24 | **Borderline** qualification (judgment call) | Route to **broker judgment** rather than auto-drop | Handoff |
| 25 | Books, **shows, but not ready to proceed** | Post-appointment nurture, scheduled re-touch | Nurture |
| 26 | **Replies angry / complaint** | Immediate human-handoff alert, pause automation | Handoff |
| 27 | Gives **partial info** then books anyway | Book it; broker collects rest on the call | Booked |

> **Coverage guarantee:** every possible lead behavior routes to one of four buckets — **Booked,
> Nurture (re-triggered), DNC/Closed/Invalid (terminal), or Human Handoff.** There is no fifth bucket
> called "nothing happens."

---

## The non-responder cadence (scenario #5 — the crack most leads fall through)

Persistence is where brokers lose leads. The AI does **6–8 touches across channels**, then nurture —
never a single touch and quit.

| When | Channel | Action |
|---|---|---|
| 0 min | Text + Email | Instant response |
| 5 min | Text | "Still there, {{first_name}}? Just need to know — buy or refinance?" |
| 30 min | **Broker task / AI call** | Attempt a call |
| 1 hour | Email | Re-send with a softer ask |
| Day 1 | Text + Email | "Want to make sure you got my note…" |
| Day 2 | Broker task / call | Second call attempt |
| Day 4 | Text | Value angle ("rates/programs changed…") |
| Day 7 | Email | "Should I keep your file open?" |
| Day 10 | Text + Email | Breakup → then **Nurture** (monthly), not stop |

→ Any reply at any step **stops the cadence** and starts pre-qual. After Day 10 with no reply, they go
to **long-term nurture** — still alive, never dropped.

---

## Human-handoff triggers (when the AI alerts a human immediately)
The AI hands off — and notifies the broker/you — when:
- 🔥 **Hot/high-intent** lead (ready now, pre-approved, offer in hand)
- 🙋 Lead **explicitly asks for a person** / a call
- ❓ Anything the AI **can't parse** (language, unusual situation)
- ⚖️ **Borderline qualification** (judgment call — never auto-reject a maybe)
- 😠 **Complaint / anger / escalation**
- 🚫 **No-show** (broker should know)

Handoff = create task + notify broker (SMS/email/in-app) + pause automation on that contact so a human
takes the wheel.

---

## The stall monitor (the final safety net)
A background check: **any lead with no activity AND no scheduled next touch for 3+ days → alert.**
This catches anything that somehow escaped the matrix (a workflow misfire, an unhandled reply). It's
the "nothing rots silently" guarantee — if a lead is ever stuck with no next action, a human is told.

---

## State machine (no dead ends)

```
                         ┌──────────► DNC / Invalid / Bad-contact  (terminal)
                         │
NEW LEAD ─► Instant ─► Awaiting ─┬─ reply ─► Pre-Qual ─┬─ qualified ─► BOOKED ─┬─ shows ─► (LOS / won)
            response    reply    │                     │                       └─ no-show ─► reschedule cadence ─┐
                         │       │                     └─ not ready ─► NURTURE ◄──────────────────────────────────┘
                         └─ no reply ─► CADENCE ─► (reply→Pre-Qual) or ─► NURTURE
                                                                            │
                                            NURTURE ─ re-engages / date-trigger ─► back to Pre-Qual
                                            (loops quarterly forever until Booked/DNC)

   Any state ──(hot / asks for human / can't parse / complaint / borderline)──► HUMAN HANDOFF
```

Every arrow lands somewhere. The only exits are **Booked → won**, **DNC/Invalid (terminal)**, or
**Nurture (which loops back)**. A lead literally cannot reach a state with no next action.

---

## GHL build spec (new-lead flow — connects to the shared back half)

1. **Workflow:** "Engine — New Lead Intake"
2. **Trigger:** Form Submitted **OR** Tag `engine-newlead` added **OR** Inbound SMS/missed call
3. **Settings:** stop-on-reply, stop-on-booking ON
4. **Step 1:** Instant Text (if phone+consent) + Instant Email (always) — **no wait before first touch**
5. **Branch:** reply? → go to shared **"Engine — Pre-Qual"** workflow (already built)
6. **No reply:** run the **cadence** table (waits + multi-channel steps above)
7. **Cadence exhausted:** add tag `engine-nurture` → shared nurture workflow
8. **Handoff conditions:** If/Else on keywords (hot terms, "call me", "person", complaint) → Create
   Task + Notify broker + remove from automation
9. **No-show:** Appointment status = no-show → no-show cadence + notify broker
10. **Stall monitor:** scheduled check — no activity + no pending step 3 days → notify

> Reuses the **Pre-Qual**, **Booking**, and **Nurture** workflows from `phase1-ghl-build-kit.md`. You
> build the *front* (instant response + cadence + handoff) and **wire it into the existing back half.**

---

## Tomorrow's add-on checklist (after the reactivation build)
- [ ] Build "Engine — New Lead Intake" workflow (instant response + cadence)
- [ ] Wire its reply branch into the existing "Engine — Pre-Qual" workflow
- [ ] Add handoff If/Else branches (hot / asks-for-human / complaint keywords)
- [ ] Add no-show cadence on the calendar/appointment
- [ ] Set up the stall-monitor scheduled check
- [ ] Test each scenario: submit a test lead, reply variants (engaged / question / STOP / "call me" /
      no-reply / no-show) → confirm each lands in the right state with a next action
```
