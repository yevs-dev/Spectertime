# Spectertime — Site & Automation System Documentation

_Last updated: 2026-09-19_

---

## Site

**URL:** spectertime.ai  
**Stack:** Static HTML/CSS/JS — no framework, no build step → GitHub → Netlify (auto-deploy on push)  
**Font:** Inter (Google Fonts) — loaded in `<head>` on every page  
**Color scheme:** Dark background (`#0a0a0a`), off-white text (`#f5f5f0`), faint text (`#888`), border (`#222`), accent (`#c8f547` yellow-green)

---

## File Structure

```
Spectertime/                        ← git repo root (connected to Netlify)
  index.html                        ← homepage (serves spectertime.ai)
  insights/
    index.html                      ← insights listing page (spectertime.ai/insights/)
    what-can-run-itself/
      index.html                    ← article 1 (spectertime.ai/insights/what-can-run-itself/)
    the-guest-asked-reception/
      index.html                    ← article 2 (spectertime.ai/insights/the-guest-asked-reception/)
  Spectertime site automation doc.md  ← this file
```

> The old `spectertime_v3.html` working copy is no longer used. Edit the files in their correct locations (`Spectertime/index.html`, `Spectertime/insights/...`) directly — Claude works on them in the AI Automation project folder which is the git repo.

**Deploy workflow:**
1. Edit the relevant file(s) in the `Spectertime/` folder
2. Open GitHub Desktop → commit changes → Push origin
3. Netlify detects the push and auto-deploys within ~30 seconds

**Git / Netlify:**
- GitHub repo: `Spectertime/` folder is the repo root, connected to Netlify
- Netlify build: no build step — deploys static files as-is
- Netlify auto-deploy is triggered on every push to the main branch
- No manual publish step needed

---

## Pages

### Homepage (`index.html`)
Sections in order: Nav → Hero → Stats ticker → Services → How it works (steps) → About → Insights preview → Contact form → Footer

**Nav CTA:** "Book a 20-min Call" → `/#contact`  
**Hero:** Avatar image + label + title + subtitle + two CTA buttons  
**Contact form:** Collects name, email, company, message. Submits via `handleSubmit()` to Make webhook. See Contact Form section below.

### Insights Listing (`insights/index.html`)
Shows all published article cards. Each card links to its article subfolder (`/insights/what-can-run-itself/` etc.).  
Page header: eyebrow label + title + subtitle.  
**Back nav:** "← spectertime.ai" links to homepage root.

### Article: What Can Actually Run Itself (`insights/what-can-run-itself/index.html`)
Tag: Strategy  
Author: Yev Specter  
Content: Long-form article with multiple `<h2>` sections, a comparison table (`.comparison-table`), closing section (`.article-closing`), and CTA block (`.article-cta`).  
**Article CTA button:** "Book a 20-min Call" → `/#contact` (takes user to homepage contact form section)

### Article: The Guest Asked Reception (`insights/the-guest-asked-reception/index.html`)
Tag: Operations  
Author: Yev Specter  
Same structure as article 1. Longer article (~350 lines of HTML body).

---

## Animation System

All pages share the same scroll-reveal mechanism. It must be kept identical across homepage and all article/listing pages.

### How it works

**1. CSS pre-hide (in `<style>` block in `<head>`):**  
Header elements that animate on load are hidden before first paint to prevent flash of visible content:
```css
/* Homepage */
.hero-avatar, .hero-label, .hero-title, .hero-sub, .hero-actions { opacity: 0; transform: translateY(18px); }

/* Article pages */
.article-back, .article-tag, .article-title, .article-byline { opacity: 0; transform: translateY(18px); }

/* Insights listing */
.page-eyebrow, .page-title, .page-subtitle, .insight-card { opacity: 0; transform: translateY(18px); }
```

**2. IntersectionObserver (in `<script>` at end of `<body>`):**  
```javascript
const observer = new IntersectionObserver((entries) => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.style.opacity = '1';
      entry.target.style.transform = 'translateY(0)';
    }
  });
}, { threshold: 0.1 });
```
Articles use `_observer` (underscore prefix) to avoid naming conflict if homepage JS ever loads on the same page.

**3. `revealGroup()` helper:**  
Sets elements to `opacity: 0 + translateY` and registers them with the observer. The observer fires immediately for elements already in the viewport, and on scroll for elements below the fold.
```javascript
function revealGroup(selector, { y = 28, dur = 0.7, gap = 80 } = {}) {
  document.querySelectorAll(selector).forEach((el, i) => {
    el.style.opacity = '0';
    el.style.transform = `translateY(${y}px)`;
    el.style.transition = `opacity ${dur}s cubic-bezier(0.25,0.46,0.45,0.94) ${i * gap}ms, transform ${dur}s cubic-bezier(0.25,0.46,0.45,0.94) ${i * gap}ms`;
    observer.observe(el);
  });
}
```

**4. Hero/header stagger (load animation, not scroll):**  
Elements reveal sequentially on page load, 90ms apart, starting at 60ms:
```javascript
['.hero-avatar', '.hero-label', '.hero-title', '.hero-sub', '.hero-actions'].forEach((sel, i) => {
  const el = document.querySelector(sel);
  if (!el) return;
  el.style.opacity = '0';
  el.style.transform = 'translateY(18px)';
  el.style.transition = 'opacity 0.7s cubic-bezier(0.25,0.46,0.45,0.94), transform 0.7s cubic-bezier(0.25,0.46,0.45,0.94)';
  setTimeout(() => { el.style.opacity = '1'; el.style.transform = 'translateY(0)'; }, 60 + i * 90);
});
```

### What gets revealed on each page

**Homepage:**
```javascript
revealGroup('.service-card, .step');
revealGroup('.section-title, .section-sub, .contact-title, .contact-sub', { y: 18, gap: 0 });
revealGroup('.about-img', { y: 18, gap: 0 });
revealGroup('.about-title, .about-text', { y: 18, gap: 100 });
revealGroup('.contact-form-card', { y: 18, gap: 0 });
```

**Article pages (both articles, using `_revealGroup` / `_observer`):**
```javascript
// Header stagger (load animation)
['.article-back', '.article-tag', '.article-title', '.article-byline'].forEach(...)

// Scroll-reveal for body
_revealGroup('.article-body > p, .article-body > ol, .article-body > hr, .article-body h2, .article-body h3, .article-body .comparison-table', { y: 18, dur: 0.7, gap: 0 });
_revealGroup('.article-closing', { y: 18, gap: 0 });
_revealGroup('.article-cta', { y: 18, gap: 0 });
```

**Insights listing page (using `_revealGroup` / `_observer`):**
```javascript
// Header stagger (load animation)
['.page-eyebrow', '.page-title', '.page-subtitle'].forEach(...)

// Scroll-reveal for cards
_revealGroup('.insight-card', { y: 28, gap: 80 });
```

### Important rules
- **Never use `getBoundingClientRect()` to filter which elements get observed.** This breaks reveal for elements that happen to be near the viewport edge — just hide everything and let the observer handle it.
- **CSS pre-hide must match the JS stagger selectors exactly** — otherwise elements flash visible before JS hides them.
- **Do not add `opacity: 0` in CSS for scroll-reveal elements** (body paragraphs, cards, etc.) — only for load-stagger elements. JS sets `opacity: 0` on those just before the observer is registered, so no flash occurs.
- The `{ threshold: 0.1 }` setting on the observer means 10% of the element must be visible to trigger reveal. This is intentional — don't increase it or reveals will trigger too late.

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
