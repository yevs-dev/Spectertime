# Spectertime — Site & Automation System Documentation

_Last updated: 2026-08-12_

---

## Site

**URL:** spectertime.ai  
**Stack:** Single HTML file → GitHub → Netlify (auto-deploy on push)

**Files:**
- `spectertime_v3.html` — local working copy in the AI Automation project folder (edit here first)
- `Spectertime/index.html` — the file Netlify actually serves (git repo copy)

**Deploy workflow:**
1. Edit `spectertime_v3.html` locally
2. Copy to `Spectertime/index.html` (overwrite)
3. Open GitHub Desktop → commit changes → Push origin
4. Netlify detects the push and auto-deploys within ~30 seconds

**Git / Netlify:**
- GitHub repo: `Spectertime/` folder is the repo root, connected to Netlify
- Netlify build: no build step — deploys `index.html` as a static site
- Netlify auto-deploy is triggered on every push to the main branch
- No manual publish step needed

---

## Contact Form

Located in the `#contact` section of the site. Collects:
- Name
- Email
- Company
- Message (textarea, `name="message"`)

On submit, `handleSubmit()` fires immediately:
1. POSTs JSON `{name, email, company, message}` to the Make webhook
2. Shows success message in place of the form (no page reload)
3. Copy: _"Thanks — your request has been received. Check your inbox for a few short questions..."_

**Webhook URL:** `https://hook.eu1.make.com/bj7oio0ni3z9lnyqtj29fbccytzhshtc`

---

## Make Scenarios

### Scenario 1 — Contact Form → Qual Email → AC (ID: 6908602)
**Status:** Active  
**Trigger:** Instant webhook (hook ID: 3531345)

| # | Module | Config |
|---|--------|--------|
| 1 | Webhooks — Custom webhook | Hook: Spectertime — Contact Form |
| 2 | Gmail — Send an email | Connection: yev@spectertime.ai (9598981) · To: `{{1.email}}` · Subject: "One quick step before we talk" · Body type: Raw HTML · Content: qual email with Typeform link |
| 3 | ActiveCampaign — Create or Update a Contact | Email: `{{1.email}}` · First Name: `{{1.name}}` |
| 4 | ActiveCampaign — Add a Tag to a Contact | Contact ID: `{{3.id}}` · Tag: `Spectertime_contact_form_submitted` |

**Qual email Typeform link:**
```
https://form.typeform.com/to/z2L7aJCt#spectertime_form_email={{1.email}}&name={{1.name}}
```
Hidden fields pass email and name silently into the Typeform response.

---

### Scenario 2 — Typeform → AC Update + Calendar Shown Tag
**Status:** Active  
**Trigger:** Typeform — Watch Responses (instant)  
**Typeform webhook URL:** `https://hook.eu1.make.com/y1ijvc9uux5lnmhoex4rabsugd8l4dzb`  
**Typeform form ID:** `z2L7aJCt`

| # | Module | Config |
|---|--------|--------|
| 1 | Typeform — Watch Responses | Webhook registered in Typeform → Connect → Webhooks |
| 4 | ActiveCampaign — Create or Update a Contact | Email: `{{1.hidden.spectertime_form_email}}` · Custom fields: all 4 Typeform answers mapped to Spectertime quick qual field group |
| 5 | ActiveCampaign — Add a Tag to a Contact | Contact ID: `{{4.id}}` · Tag: `Spectertime_calendar_shown` |

**Custom fields mapped (Spectertime quick qual group):**
- What type of real estate business do you run?
- How many leads do you receive per month?
- What's your biggest bottleneck right now?
- What's your timeline for getting something in place?

**Note:** If the form is filled via direct Typeform URL (not the email link), `spectertime_form_email` will be empty and the AC module will error. This is expected — real submissions always come through the email link.

---

### Scenario 3 — Cal.com Booking → AC Tags
**Status:** Active  
**Trigger:** Cal.com — Watch Booking Created  
**Cal.com webhook:** Registered on Discovery call event type (not global) · Events: Booking created + Booking cancelled  
**Webhook URL:** `https://hook.eu1.make.com/x2wor3nor64w92fvdbcrmu3skdmbsqjd`

| # | Module | Config |
|---|--------|--------|
| 1 | Cal.com — Watch Booking Created | Webhook: registered on Discovery call event type |
| 2 | Router | Two routes based on `Trigger Event` value |
| **Route 1** | Filter: `Trigger Event` = `BOOKING_CREATED` | — |
| 7 | AC — Create or Update Contact | Email: `{{1.payload.attendees[].email}}` · First Name: `{{1.payload.attendees[].firstName}}` · Last Name: `{{1.payload.attendees[].lastName}}` |
| 9 | AC — Add Tag | Contact ID: `{{7.id}}` · Tag: `Spectertime_booking_made` |
| **Route 2** | Filter: `Trigger Event` = `BOOKING_CANCELLED` | — |
| 8 | AC — Create or Update Contact | Email: `{{1.payload.attendees[].email}}` · First Name: `{{1.payload.attendees[].firstName}}` · Last Name: `{{1.payload.attendees[].lastName}}` |
| 11 | AC — Add Tag | Contact ID: `{{8.id}}` · Tag: `Spectertime_booking_cancelled` |

**Note:** Each webhook fires only for the Discovery call event type. Other Cal.com event types have their own separate webhooks and won't trigger this scenario.

---

## Typeform — Quick Qualifier (z2L7aJCt)

**4 questions:**
1. What type of real estate business do you run? _(multiple choice: Solo agent / Small team 2–5 / Large team or brokerage / Property management / Other)_
2. How many leads do you receive per month? _(multiple choice)_
3. What's your biggest bottleneck right now? _(multiple choice)_
4. What's your timeline for getting something in place? _(multiple choice)_

**Ending screen:** "You're all set - pick a time below." with inline link → `cal.com/yev-s/30min`  
_(Auto-redirect not available on Basic plan — link is embedded in description text)_

**URL parameters (hidden fields):**
- `spectertime_form_email` — passed from qual email, carries respondent's email
- `name` — passed from qual email, carries respondent's name

**Webhook:** Registered under Connect → Webhooks. Typeform auto-disables on repeated 410 errors — re-enable if needed.

---

## ActiveCampaign

**Tags used:**
- `Spectertime_contact_form_submitted` — added when contact form submitted (Scenario 1)
- `Spectertime_calendar_shown` — added when Typeform completed (Scenario 2) · triggers AC automation
- `Spectertime_booking_made` — added when Cal.com booking confirmed (Scenario 3, Route 1)
- `Spectertime_booking_cancelled` — added when Cal.com booking cancelled (Scenario 3, Route 2)

**Custom field group:** Spectertime quick qual  
Fields mirror the 4 Typeform questions.

**Domain:** `spectertime.ai` — authenticated and verified in AC.

---

## AC Automation — Spectertime 24h delay
**Status:** Active

**Trigger:** Tag `Spectertime_calendar_shown` added  
**Flow:**
1. Wait 24 hours
2. If/Else: does contact have tag `Spectertime_booking_made`?
   - **YES** → Automation ends (they booked, no email needed)
   - **NO** → Send email "Still want to connect?" → Automation ends

**Follow-up email:**  
- Name: Spectertime 24 delay  
- Subject: "Still want to connect?"  
- From: Yev Specter · yev@spectertime.ai  
- Body:

> Hey {{first_name}},
>
> You filled out our form but never grabbed a time.
>
> Still interested in cutting out the busywork with AI? Pick a slot — 45 minutes. No pitch, just a conversation about what you're working with.
>
> → [Book a call](https://cal.com/yev-s/30min)
>
> Talk soon,  
> Yev

---

## Full Lead Flow (end to end)

```
spectertime.ai contact form
        ↓
Make Scenario 1
        ↓ (parallel)
Qual email sent to lead          AC contact created + contact_form_submitted tag
        ↓
Lead clicks Typeform link (email pre-loaded as hidden field)
        ↓
Lead completes Typeform
        ↓
Make Scenario 2
        ↓ (parallel)
AC contact updated               calendar_shown tag added
(qual answers stored)                    ↓
                              AC automation starts 24hr wait
        ↓
Lead clicks "Book your call" link on Typeform ending screen
        ↓
Cal.com booking made
        ↓
Make Scenario 3 → Route 1
        ↓
booking_made tag added to AC contact
        ↓
AC automation check: booking_made tag? → YES → automation ends, no email sent
```

If lead does NOT book within 24hrs:
```
AC automation 24hr wait expires
        ↓
If/Else check: booking_made tag? → NO
        ↓
Follow-up email sent: "Still want to connect?" with cal.com/yev-s/30min link
```

If lead cancels their booking:
```
Cal.com booking cancelled
        ↓
Make Scenario 3 → Route 2
        ↓
booking_cancelled tag added · booking_made tag removed
        ↓
(future: booking_cancelled tag can trigger separate re-engagement automation in AC)
```
