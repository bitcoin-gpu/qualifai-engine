# Reactivation Workflow — Strict Audit & Fix

Audit of the existing GHL reactivation workflow (the one with triggers `engine-reactivation` +
`Customer replied`, Email 1 → SMS → Email 2 → SMS 2 → Email 3 → AI Pre-Qual routing).

**Verdict: structurally broken on the safety-critical paths** (replies, consent, stops, routing).
The email/SMS *content* and the *cadence concept* are fine and worth keeping. The *logic wiring* needs
a rebuild into clean separate workflows. Below is exactly what's wrong and the safe version.

---

## 1) WORKFLOW SUMMARY

**What it currently does:** Fires on the `engine-reactivation` tag AND on any customer reply. Sends
Email 1, waits, sends SMS 1, then runs a tangle of "Wait for Reply or Timeout" branches and
"Check SMS Consent" conditions, eventually reaching Email 2, SMS 2, Email 3, and finally an
"AI Pre-Qual Outcome Routing" block whose branches all dead-end at END.

**What it's intended to do:** Re-engage an old lead across email + (consented) SMS; the moment they
reply, stop the outreach and route them into pre-qualification → book if qualified, nurture if not;
honor opt-outs and never text without consent.

**The gap:** the intent is right; the wiring doesn't deliver it. Replies don't reach pre-qual, SMS
isn't reliably consent-gated, the reply-trigger can re-loop the whole sequence, and nothing stops on
booking or routes to DNC.

---

## 2) ISSUES FOUND

### 🔴 CRITICAL

**C1 — "Customer Replied" as a 2nd trigger on the same workflow → re-enrollment loop**
*Why:* a reply re-fires this workflow from the top and **re-sends Email 1** (and can stack duplicate
enrollments). Classic spam/loop.
*Fix:* **Remove the `Customer replied` trigger entirely.** Reply-stopping belongs in **Settings →
Stop on Response = ON**, not as a trigger. Reply-*routing* belongs in a separate Pre-Qual workflow.

**C2 — SMS not reliably gated by consent before sending**
*Why:* "Send SMS 1 (If consented)" runs in the cadence while the "Check SMS Consent" condition lives
elsewhere in a reply branch. If the gate isn't immediately before *every* SMS send, you risk texting
non-consented contacts = **TCPA exposure ($500–1,500/text).**
*Fix:* Put an **If/Else: `SMS Consent` = Yes immediately before EVERY SMS step.** Yes → send. No →
skip and continue. No exceptions.

**C3 — Replies don't route to Pre-Qual (they dead-end)**
*Why:* mid-sequence reply branches go to "Check SMS Consent → END," and the AI Pre-Qual block only
sits at the very bottom (after Email 3). **A borrower who replies to Email 1 never reaches
qualification** — the #1 thing this system exists to do.
*Fix:* Reactivation should just **stop on reply** (setting). A **separate Pre-Qual workflow** triggers
on the reply and runs qualification. Every reply at any stage routes correctly.

**C4 — No appointment-booked exit**
*Why:* if a contact books, the sales/nurture steps keep firing — they get "should I close your file?"
after they've already booked. Embarrassing + erodes trust.
*Fix:* Enable **Stop on Appointment Booked** (workflow setting) on Reactivation, New-Lead, and
Nurture. Booking halts all outreach.

**C5 — No STOP / unsubscribe → DNC path**
*Why:* a borrower saying "stop" still sits in the sequence = compliance violation + complaint risk.
*Fix:* In the Pre-Qual workflow, branch: reply contains STOP/unsubscribe/remove → **tag `engine-dnc`**
+ rely on GHL native opt-out. A `engine-dnc` contact is never messaged by any workflow.

**C6 — Stop-on-Response setting likely OFF**
*Why:* the design leans on manual reply branches instead of the setting, so a contact who replies can
still receive later scheduled steps.
*Fix:* **Settings → Stop on Response = ON** for all outbound sequences.

### 🟠 HIGH

**H1 — Consent branch is inverted** ("Has SMS Consent → END", "None → continue")
*Why:* backwards — consented contacts get dropped, non-consented continue (and can hit a later SMS).
*Fix:* Correct it: **Has Consent → send SMS; No Consent → skip the SMS, continue the email track.**

**H2 — AI Pre-Qual routing branches all → END with no actions**
*Why:* "Qualified → END" books nothing; "Not ready → END" adds no nurture tag. The routing is inert.
*Fix:* Qualified → tag `engine-qualified` + send booking message + set pipeline "Appointment Ready."
Not ready → tag `engine-nurture` + enroll Nurture workflow.

**H3 — No tags added/removed → messy, unknowable pipeline state**
*Why:* without `engine-replied / qualified / nurture / dnc`, you can't tell what state a contact is in,
and workflows can't route cleanly.
*Fix:* Tag at every transition (see blueprint). Remove superseded tags (e.g., remove `engine-nurture`
when `engine-qualified` is added).

**H4 — Pre-Qual embedded inside the reactivation workflow**
*Why:* violates separation, creates the tangle, and means pre-qual only runs at the end.
*Fix:* Pre-Qual = its **own workflow** (or Conversation AI handoff).

**H5 — Duplicate-enrollment / re-entry risk**
*Why:* reply trigger + re-entry allowed = a contact can be in the sequence multiple times = double
messages.
*Fix:* **Allow Re-Entry = OFF** on Reactivation. Remove the reply trigger (C1).

### 🟡 MEDIUM

**M1 — No broker handoff / notification** for hot leads, "call me," or complaints.
*Fix:* In Pre-Qual, branch escalation → Create Task + Notify broker, stop automation.

**M2 — Redundant "Wait for Reply or Timeout" steps** create the tangle.
*Fix:* Delete them; Stop-on-Response handles reply-stopping natively.

**M3 — Non-responders may just hit END (stuck, no next action)**
*Why:* violates "no lead in a dead state."
*Fix:* End of Reactivation → tag `engine-nurture` → Nurture workflow. Everyone lands somewhere.

**M4 — Pipeline stages not updated** → confusion about where contacts are.
*Fix:* Set stages at transitions: New → Re-engaged → Qualified → Appt Booked → Won/Lost.

### 🟢 LOW

**L1 — Sending window:** confirm Tue–Thu business hours for deliverability (domain is recovering).
**L2 — No links in first cold email/SMS:** confirm — booking link only after reply/opt-in.

---

## 3) IMPROVED WORKFLOW BLUEPRINT (separate workflows)

### Workflow A — "Engine - Reactivation"
- **Trigger:** Contact Tag added = `engine-reactivation`
- **Settings:** Stop on Response = ON · Stop on Appointment Booked = ON · Allow Re-Entry = OFF
- **Steps:**
  1. Email 1 (Re-open)  *(no link)*
  2. Wait 2 days
  3. If/Else `SMS Consent = Yes` → SMS 1  | else skip
  4. Wait 2 days
  5. Email 2 (Rates/Market)
  6. Wait 3 days
  7. If/Else `SMS Consent = Yes` → SMS 2 | else skip
  8. Wait 3 days
  9. Email 3 (Breakup)
  10. Wait 2 days
  11. Remove tag `engine-reactivation`, Add tag `engine-nurture`
  12. END (→ Nurture workflow picks up via the tag)

### Workflow B — "Engine - Pre-Qual" (reply router)
- **Trigger:** Customer Replied — **Filter:** has tag `engine-reactivation` OR `engine-newlead`
- **Settings:** Stop on Response = OFF (this one *handles* the reply), Allow Re-Entry = ON
- **Steps:**
  1. Add tag `engine-replied`
  2. If reply contains STOP/unsubscribe/remove/"don't contact" → tag `engine-dnc` → END
  3. If reply contains "call me"/"speak to someone"/complaint/legal/foreclosure/bankruptcy/etc. →
     Create Task + Notify broker → tag `engine-escalated` → END
  4. Hand to **Conversation AI** (or workflow Q&A) → it captures fields + applies outcome tag
  5. If/Else `engine-qualified` → send Booking message (with calendar link) + set pipeline
     "Appointment Ready" + remove `engine-nurture` → END
  6. Else `engine-nurture` → add tag → END

### Workflow C — "Engine - New Lead Intake" (speed-to-lead)
- **Trigger:** Form Submitted OR Tag `engine-newlead` OR Inbound Message
- **Settings:** Stop on Response = ON · Stop on Appointment Booked = ON · Allow Re-Entry = OFF
- **Steps:** Instant Email + (consent-gated) Instant SMS → 5-min nudge → 30-min broker call task →
  Day1/Day2/Day4/Day7 multichannel cadence (each SMS consent-gated) → Day10 breakup →
  tag `engine-nurture` → END

### Workflow D — "Engine - Appointment Confirmation"
- **Trigger:** Appointment Booked
- **Steps:** Confirmation (email + consented SMS) → 24h reminder → 1h reminder →
  If No-Show → "reschedule?" + Notify broker → tag `engine-nurture`

### Workflow E — "Engine - Nurture"
- **Trigger:** Tag added `engine-nurture`
- **Settings:** Stop on Response = ON · Stop on Appointment Booked = ON
- **Steps:** monthly value email ×3 → loop quarterly. Any reply → Stop-on-Response halts → Pre-Qual
  (B) catches it.

---

## 4) CONSENT & COMPLIANCE REVIEW

| Control | Required state |
|---|---|
| **SMS consent** | Hard If/Else `SMS Consent = Yes` immediately before EVERY SMS. No = skip, never send. |
| **Opt-out** | "STOP" in any reply → `engine-dnc` + GHL native opt-out. `engine-dnc` = never messaged again, by any workflow. |
| **Stop-on-reply** | Setting ON for A, C, D, E. Reply halts outbound; B routes it. |
| **Appointment-booked exit** | Stop on Appointment Booked = ON for A, C, D, E. Booking halts outreach. |
| **DNC** | `engine-dnc` tag is the master suppression. Add a filter to every outbound workflow: do not enroll if `engine-dnc`. |
| **Email (CAN-SPAM)** | physical address + unsubscribe in every email; no links in first cold touch. |
| **A2P 10DLC** | registered before any SMS sends. |

---

## 5) PRE-QUAL ROUTING REVIEW

**How replies should route:** Reactivation/New-Lead **stop on reply** (setting). The **Pre-Qual
workflow** triggers on the reply (filtered to engine tags), so every reply — Email 1, SMS, Email 3,
anytime — routes to qualification. No reply ever dead-ends.

**Where Conversation AI takes over:** Pre-Qual step 4 hands the conversation to **GHL Conversation AI**
(`conversation-ai-prompt.md`). The AI runs the Q&A, writes fields (Loan Purpose, Timeline, Credit,
etc.), and applies `engine-qualified` or `engine-nurture`. The workflow then acts on the tag.
*(Free interim: replace step 4 with workflow Send-Question → If/Else steps until you turn on the
$97/mo Conversation AI.)*

**When the broker is notified:** immediately on escalation (step 3) — hot/ready-now, "call me,"
complaints, or distress signals (foreclosure, bankruptcy, lawsuit, divorce, death, fraud). Also on a
qualified booking (FYI notification) and on a no-show.

---

## 6) FINAL RECOMMENDED VERSION (preserving your work)

**Keep:** all your email + SMS copy, the Email 1→SMS→Email 2→SMS 2→Email 3 cadence, the wait timing.

**Change (minimum to make it safe):**
1. **Delete the `Customer replied` trigger** from Reactivation. (C1)
2. **Turn ON** Stop on Response + Stop on Appointment Booked; **turn OFF** Allow Re-Entry. (C4, C6, H5)
3. **Delete every "Wait for Reply or Timeout" branch** — the settings replace them. (M2)
4. **Put an If/Else `SMS Consent = Yes` directly before each SMS**, correctly (Yes→send). (C2, H1)
5. **Strip the AI Pre-Qual block out of this workflow** into the separate **Pre-Qual** workflow
   triggered by reply. (C3, H4)
6. **At the end, tag `engine-nurture`** so non-responders flow to Nurture, not END. (M3)
7. Build the **Pre-Qual** workflow with DNC + escalation + Conversation-AI handoff + qualified/nurture
   routing + tags. (C5, H2, H3, M1)

Result = your same sequence and copy, but linear and safe, with replies/consent/stops/routing handled
correctly across clean separate workflows.

---

## 7) TEST CHECKLIST (run each in GHL before going live)

| Test | Steps | Pass = |
|---|---|---|
| **New lead** | Submit a test form / add `engine-newlead` | Instant email (+SMS if consent) within seconds; cadence armed |
| **Old-lead reactivation** | Import a test contact, add `engine-reactivation` | Email 1 fires; SMS only if consent; cadence spaced correctly |
| **No SMS consent** | Test contact with `SMS Consent` = No | **Zero SMS sent**; emails still send |
| **Reply after Email 1** | Reply from the test contact | Reactivation **stops** (no Email 2); Pre-Qual starts |
| **STOP reply** | Reply "STOP" | Tagged `engine-dnc`; no further messages from any workflow |
| **Appointment booked** | Book via the calendar link | All outreach **stops**; confirmation + reminders fire |
| **No response** | Let it run untouched | Completes cadence → tagged `engine-nurture` → Nurture starts (not stuck at END) |
| **AI qualified** | Reply with: buying, 1–3 months, good credit | Tagged `engine-qualified`; booking message sent; pipeline → Appointment Ready |
| **Not ready / nurture** | Reply with: just researching / 12 months | Tagged `engine-nurture`; nurture message sent; no booking link |
| **Broker handoff** | Reply "call me" or "I'm facing foreclosure" | Automation stops; broker notified + task created; `engine-escalated` |
| **Duplicate enrollment** | Add the tag twice / reply mid-sequence | Contact enrolls **once**; no duplicate Email 1 |
| **DNC suppression** | Try to enroll a `engine-dnc` contact | **Not enrolled**; receives nothing |

> Run every row. A single failed row = do not go live until fixed. This touches real borrowers.
