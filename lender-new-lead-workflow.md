# Lender New Lead — Corrected Blueprint

The rebuilt, safe version of the Facebook new-lead workflow. Fixes all 17 audited issues: AI-intent
routing (no fragile exact phrases), **zero dead-ends**, dedup, single application sub-workflow,
appointment-booked exit, consistent identity (**Ty / Builder Funding**), filled links, real nurture.

> Sender identity: **Ty** + **Builder Funding** everywhere (no more Hayden / {{user.name}} / QualifAI
> mix). Builder Funding = placeholder brand for now. Full audit trail of what was broken is in this
> file's git history + chat.

---

## What was wrong (1-line each)
Exact-phrase routing dropped real replies · 7 dead-end branches · `[Book Call Link]` placeholder ·
`application submitted` tag dependency could silently fail · no appointment-booked exit · no global
stop-on-response · "Nurture" was a dead stage · 3 conflicting sender identities · duplicate enrollment ·
the application block duplicated 4×. **All fixed below.**

---

## 0) PREREQUISITES (build once)

**Pipeline stages:** New Lead → Contacted → Hot Lead → Application Sent → Application Submitted →
Appointment Booked → Nurture → Dead

**Tags:** `fb-lead` · `hot` · `nurture` · `dead` · `app-link-sent` · `application-submitted` ·
`appointment-booked` · `escalate` · `dnc`

**Custom field:** `SMS Consent` = Yes (set on entry — FB form opt-in)

**AI Intents (create in Conversation AI / If-Else intent):**
| Intent | Catches replies like |
|---|---|
| `working-on-deal` | "got one under contract," "yes working on a deal," "have a property" |
| `interested-fund` | "looking to fund," "need capital," "next 30-90 days," "ready" |
| `planning-ahead` | "planning," "not yet," "still looking," "future" |
| `exploring` | "just looking," "researching," "exploring options" |
| `not-interested` | "no thanks," "not interested," "remove me" |
| `needs-help` | "yes help," "how do I," "confused," "stuck" |
| `schedule-yes` | "yes," "sounds good," "let's do it," "book it" |

**Links (fill these — no placeholders):**
`[APPLICATION_LINK]` = https://api.leadconnectorhq.com/widget/form/QeiSbKnKofmeRTYKYrHO
`[CALL_LINK]` = https://api.leadconnectorhq.com/widget/bookings/mortgage-calendar-9i5r1

> ⚠️ **Verify before launch:** the application form at `[APPLICATION_LINK]` must **add the tag
> `application-submitted` on submit.** If it doesn't, every applicant falsely hits the "Need Help" path.
> Test it first.

---

## WORKFLOW 1 — "Lender New Lead: Intake"
**Trigger:** Facebook Lead Form Submitted
**Settings:** Stop on Response = ON · Stop on Appointment Booked = ON · Allow Re-Entry = OFF ·
Create Opportunity → Allow Duplicate = No

**Steps:**
1. Set field `SMS Consent = Yes`; Add tag `fb-lead`
2. **Internal notify (team SMS):**
   `New Builder Funding lead: {{contact.first_name}} {{contact.last_name}} {{contact.phone}}. Auto-outreach has started — review for personal follow-up.`
   *(fixed: no longer says "don't contact yet")*
3. Create Opportunity → stage **New Lead** (dedup on)
4. **Instant SMS:**
   `Hey {{contact.first_name}}, this is Ty with Builder Funding — we fund fix & flip, ground-up, and SSR deals. Saw your request. Are you working on a deal right now or planning ahead? (Reply STOP to opt out.)`
5. **Instant Email** — Subject: `Funding your next investment deal`
   `Hey {{contact.first_name}}, Builder Funding specializes in fast capital for fix & flips and ground-up construction. Reply here or text me back and I'll see if we're a fit for your next deal. — Ty, Builder Funding` + physical address + unsubscribe
6. Move stage → **Contacted**
7. **Wait** for reply or 1 hour
8. **IF replied → AI intent route** (else → go to step 9 cadence):
   - `working-on-deal` OR `interested-fund` → **Add tag `hot`** (→ WF2)
   - `planning-ahead` OR `exploring` → **Add tag `nurture`** (→ WF4)
   - `not-interested` → **Add tag `dead`** → Dead SMS → stage **Dead** → END
   - `needs-help` / unclear / no intent match → **Add tag `escalate`** + notify broker (a human reads it — never guess) → END
9. **No reply after 1 hr → re-engagement cadence:**
   a. **Checking-In SMS:** `Just checking in {{contact.first_name}} — looking to fund a deal in the next 30–90 days, or just exploring?`
   b. Wait reply or 1 day → **IF replied → AI intent route (same as step 8)**
   c. No reply → **Follow-Up SMS:** `We close fast once the numbers make sense. Want me to take a quick look at your deal, or should I circle back later?`
   d. Wait reply or 1 day → **IF replied → AI intent route**
   e. No reply → **Close-the-Loop SMS:** `Don't want to bug you — should I close this out for now, or still exploring funding options?`
   f. Wait reply or 1 day → **IF replied → AI intent route**
   g. **No reply → Add tag `nurture` → stage Nurture → END** *(no dead-end — everyone lands in nurture)*

**Dead SMS (reused):** `Got it — appreciate the reply. If a deal comes up later, just reply FUNDING and I'll jump back in.`

---

## WORKFLOW 2 — "Lender New Lead: Qualification"
**Trigger:** Tag added `hot`
**Settings:** Stop on Response = ON · Stop on Appointment Booked = ON
**Steps:**
1. Move stage → **Hot Lead**
2. **Qualification SMS:** `Awesome — a few quick Qs so I don't waste your time: fix & flip or ground-up? purchase price? rehab budget? market/state?`
3. Wait for reply or 1 day
   - **Reply →** Booking-Link SMS: `Based on what you shared, this looks promising. Here's the application — takes ~2 min: [APPLICATION_LINK]` → **Add tag `app-link-sent`** (→ WF3) → END
   - **Timeout →** Circling-Back SMS: `Hey {{contact.first_name}} — circling back. Once I've got those quick details I can show you exactly what Builder Funding can do for your deal.`
     - Wait reply or 1 day → **Reply →** Booking-Link SMS + tag `app-link-sent` → END
     - **No reply → Add tag `nurture` → stage Nurture → END** *(was a dead-end — now resolved)*

---

## WORKFLOW 3 — "Application Tracking" (single source — built ONCE)
**Trigger:** Tag added `app-link-sent`
**Settings:** Stop on Appointment Booked = ON
**Steps:**
1. Move stage → **Application Sent**
2. Wait for tag `application-submitted` or 1 day
   - **Tag added →** stage **Application Submitted** → Application-Received SMS:
     `Got your application, {{contact.first_name}} — thanks! The team will review it. Grab a time for a call here: [CALL_LINK]` → Internal notify team → END
   - **Timeout →** Need-Help SMS: `Quick heads up {{contact.first_name}} — I haven't seen your application come through yet. Want a hand finishing it?`
     - Wait reply or 1 day → **IF intent `needs-help`/`schedule-yes` →** Happy-to-Help SMS:
       `No problem — happy to help. Easiest is a quick call to walk through it together. Grab a time: [CALL_LINK]` *(fixed: real link, not [Book Call Link])* → END
     - **No reply / other → Add tag `nurture` → stage Nurture → END** *(was a dead-end — now resolved)*

> Every branch that used to repeat the "Booking → Application" block now just adds tag `app-link-sent`
> and lets THIS workflow handle it. One source of truth.

---

## WORKFLOW 4 — "Lender New Lead: Nurture" (real, recurring)
**Trigger:** Tag added `nurture` (also: keyword `FUNDING` → remove `nurture`, add `hot` → re-enters WF2)
**Settings:** Stop on Response = ON · Stop on Appointment Booked = ON
**Steps:**
1. Move stage → **Nurture**
2. **Keep-in-Loop SMS:** `Perfect — I'll keep you in the loop. When a deal starts coming together, reply FUNDING and I'll jump in.`
3. Wait 30 days → Value SMS 1 (market/rate tip)
4. Wait 30 days → Value SMS 2 (deal-readiness tip)
5. Wait 30 days → Re-qualify SMS: `Still here when you're ready, {{contact.first_name}} — any deals on the horizon? Reply FUNDING and I'll jump in.`
6. Loop quarterly. Any reply → Stop-on-Response halts → handled by intent. `FUNDING` keyword → WF2.

---

## WORKFLOW 5 — "Appointment Confirmation & Reminders"
**Trigger:** Appointment Booked · **Settings:** Stop on Response = OFF (reminders transactional)
1. Confirmation (SMS + email) — `You're all set, {{contact.first_name}}! Your call with Ty at Builder Funding is booked for {{appointment.start_time}}.`
2. 24h before → reminder
3. 1h before → reminder
4. No-show → "missed you, reschedule?" + `[CALL_LINK]` + notify team → stage back to Hot Lead/Nurture
5. Stage → **Appointment Booked** on confirm

---

## WORKFLOW 6 — DNC / Opt-out (safety net)
- GHL honors **STOP** natively (sets DND-SMS). Add: keyword `STOP/unsubscribe/remove` → tag `dnc`.
- **Every outbound workflow filters out `dnc`** (do not enroll / immediately exit if tagged).

---

## END-STATE GUARANTEE (no lead slips)
Every contact lands in exactly one: **Application Submitted** · **Appointment Booked** · **Nurture
(recurring)** · **Dead** · **DNC** · **Escalate (human)**. There is no path that ends without a tag +
stage. The 7 old dead-ends now all resolve to **Nurture** or **Dead**.

---

## COMPLIANCE
- SMS: FB-form opt-in + `SMS Consent = Yes` gate; "Reply STOP" in first SMS; STOP→DND native + `dnc`.
- Email: physical address + unsubscribe (CAN-SPAM).
- Appt-booked exit ON; A2P 10DLC registered.

---

## MIGRATION (from your current build)
1. **Verify the application form adds `application-submitted`** (test — silent killer).
2. Turn **Stop on Response + Stop on Appointment Booked ON**, **Re-Entry OFF**, dedup opportunities.
3. Replace all **exact-phrase IF/ELSE** with the **AI intents** above; delete trailing spaces.
4. Replace `[Book Call Link]` with `[CALL_LINK]`.
5. Extract the repeated application block → **WF3** (delete the 3 duplicates).
6. Point every dead-end to **`nurture` tag → WF4** (or `dead`).
7. Make sender **Ty / Builder Funding** everywhere; fix the team "don't contact yet" note.
8. Build **WF4 Nurture** + the `FUNDING` keyword trigger.

---

## TEST CHECKLIST
| Test | Pass = |
|---|---|
| FB submit | Opp created (no dupe), team notified, instant SMS + email |
| Reply "got one under contract" | Routes **hot** via intent (not exact phrase) |
| Reply "just looking" | Routes **nurture** |
| Vague reply "idk" | → `escalate` + broker notified (never stuck) |
| Reply STOP | DND set + `dnc`; all messaging stops |
| Applies (form) | `application-submitted` tag → stage Submitted + team notified |
| Applies but tag missing | Confirms WF3 doesn't false-trigger Need-Help (verify tag!) |
| Books a call | All outreach stops; confirmation fires |
| No reply ever | Cadence → **Nurture** (not dead-end), recurring |
| Reply FUNDING (from nurture) | → `hot` → Qualification |
| Duplicate FB submit | One opp, no double messaging |
| "Not interested" | → Dead stage, clean |
